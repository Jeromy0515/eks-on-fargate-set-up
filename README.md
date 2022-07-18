# EKS on Fargate
![image](https://user-images.githubusercontent.com/77256585/179463954-82e8f7c1-c395-4221-b67e-913ade5909bc.png)

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

## 