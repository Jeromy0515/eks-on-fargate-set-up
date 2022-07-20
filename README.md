# EKS on Fargate
![image](https://user-images.githubusercontent.com/77256585/179667389-336c516a-150b-49cb-b1a8-bd5ff71c5137.png)

## Install kuebctl for Linux
```
curl -o kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/linux/amd64/kubectl
curl -o kubectl.sha256 https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/linux/amd64/kubectl.sha256
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
```

## Install eksctl for Linux
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

## Install helm for Linux
```
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```

## Set up Kubeconfig
```
aws eks --region <region> update-kubeconfig --name <cluster-name>
```

## Create an IAM OIDC provider for cluster
```
eksctl utils associate-iam-oidc-provider --cluster <cluster-name> --approve
```

## Set Fargate logging

### Config log router
```
kubectl apply -f aws-observability-namespace.yaml
```

### Create ConfigMap for CloudWatch
```
kubectl apply -f aws-logging-cloudwatch-configmap.yaml
```

### Download policy
```
curl -o permissions.json https://raw.githubusercontent.com/aws-samples/amazon-eks-fluent-logging-examples/mainline/examples/fargate/cloudwatchlogs/permissions.json
```

### Create policy
```
aws iam create-policy --policy-name eks-fargate-logging-policy --policy-document file://permissions.json
```

### Attach policy to role
```
aws iam attach-role-policy \
  --policy-arn arn:aws:iam::<account-id>:policy/eks-fargate-logging-policy \
  --role-name <EKSFargatePodExecutionRole>
```
## Installing the AWS Load Balancer Controller add-on

### Create IAM Policy for AWS load balancer controller
```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://aws-load-balancer-controller-iam-policy.json
```

### Create IAM Role for AWS load balancer controller
```
eksctl create iamserviceaccount \
  --cluster=<cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name "AmazonEKSLoadBalancerControllerRole" \
  --attach-policy-arn=arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```
---
### Install cert-manager
```
kubectl apply \
    --validate=false \
    -f https://github.com/jetstack/cert-manager/releases/download/v1.5.4/cert-manager.yaml
```
### Edit aws-load-balancer-controller.yaml
```
sed -i.bak -e 's|your-cluster-name|<cluster-name>|' ./aws-load-balancer-controller.yaml
sed -i.bak -e 's|vpc-xxxxxxxx|<vpc-id>|' ./aws-load-balancer-controller.yaml
sed -i.bak -e 's|region-code|<region>|' ./aws-load-balancer-controller.yaml
```

### Apply the file
```
kubectl apply -f aws-load-balancer-controller.yaml
```
---
### Or using helm

```
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<vpc-id> \
```

#### Error: INSTALLATION FAILED: Kubernetes cluster unreachable: exec plugin: invalid apiVersion "client.authentication.k8s.io/v1alpha1"
```
curl -L https://git.io/get_helm.sh | bash -s -- --version v3.8.2
```

## Run CoreDNS on Fargate

### Remove annotation
```
kubectl patch deployment coredns -n kube-system --type=json -p='[{"op": "remove", "path": "/spec/template/metadata/annotations", "value": "eks.amazonaws.com/compute-type"}]'
```

### Restart CoreDNS
```
kubectl rollout restart -n kube-system deployment coredns
```

## CloudWatch Container Insights

### Create Namespace
```
kubectl create ns amazon-cloudwatch
```

### Set Environment variables
```
ClusterName=<cluster-name>
RegionName=<region>
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
[[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off'|| FluentBitReadFromTail='On'
[[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'
```

### Install YAML file
```
wget https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml
```

### Apply Environment variables
```
sed -i 's/{{cluster_name}}/'${ClusterName}'/;s/{{region_name}}/'${RegionName}'/;s/{{http_server_toggle}}/"'${FluentBitHttpServer}'"/;s/{{http_server_port}}/"'${FluentBitHttpPort}'"/;s/{{read_from_head}}/"'${FluentBitReadFromHead}'"/;s/{{read_from_tail}}/"'${FluentBitReadFromTail}'"/' cwagent-fluent-bit-quickstart.yaml 
```

### Open this yaml file and add this code to the 469th line
```
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: eks.amazonaws.com/compute-type
          operator: NotIn
          values:
          - fargate
```

### Deploy
```
kubectl apply -f cwagent-fluent-bit-quickstart.yaml 
```