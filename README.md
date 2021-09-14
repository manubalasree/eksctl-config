

# Cluster setup for Akeyless API Gateway

#### Note

```
This seup uses eksctl to setup the cluster. I have created an image (AMI ID: ami-061eb82e898ec0da8. ) which contains all tyhe tools required to setup the cluster
```

## Deploy Cluster with eksctl

Deploy a VM from (AMI ID: ami-061eb82e898ec0da8) on the default VPC

ssh in to the vm and clone the following repo https://github.com/manubalasree/eksctl-config.git

Deploy cluster

```
$ cd eksctl-config/
$  eksctl create cluster -f eksctlCluster.yaml
```

## Install helm required tools

Add helm repos

```
helm repo stable     add     https://charts.helm.sh/stable             
helm repo bitnami    add     https://charts.bitnami.com/bitnami        
helm repo eks        add     https://aws.github.io/eks-charts          
helm repo akeyless   add     https://akeylesslabs.github.io/helm-charts
helm repo autoscaler add     https://kubernetes.github.io/autoscaler 
```

## Install metric server

This is required for cluster autoscaler and hpa to work

```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.5.0/components.yaml
```

## Cluster Autoscaler

#### Best Practice: Add IAM permissions to CA pod via either node's instance profile or IRSA

Create a policy **EKSFromEksctlClusterAutoscaler**

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeTags",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "ec2:DescribeLaunchTemplateVersions"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
```

Associate OIDC Provider

```
eksctl utils associate-iam-oidc-provider --region=us-east-2 --cluster=akl-api-gw-prod-cluster --approve
```

Create IAM service account with Autoscaling policy

```
eksctl create iamserviceaccount \
                --name cluster-autoscaler-aws-cluster-autoscaler \
                --namespace kube-system \
                --cluster  akl-api-gw-prod-cluster \
                --attach-policy-arn arn:aws:iam::639232547460:policy/EKSFromEksctlClusterAutoscaler \
                --approve \
                --region us-east-2
```

Check if Service Account is created

```
kubectl get sa cluster-autoscaler-aws-cluster-autoscaler -n kube-system -o yaml
```

Install Auto Scaler

```
helm install cluster-autoscaler     autoscaler/cluster-autoscaler     --namespace kube-system     --values overrides.yaml
```

Verify Installation

```
kubectl --namespace=kube-system get pods -l "app.kubernetes.io/name=aws-cluster-autoscaler"
```

## Setup Load Balancer

[see documentation](eksalb/README.md)

