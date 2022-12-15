# Krakend on EKS

## Learning purpose only!

## 1. Prerequisite

Install: 
- aws cli
- kubectl
- eksctl
- helm

## 2. Create cluster

`eksctl create cluster -f cluster.yaml`

## 3. Install AWS LB Controller

### 3.1 Associate OIDC provider
eksctl utils associate-iam-oidc-provider --cluster krakend-app --region ap-southeast-3 --approve

### 3.2 Create IAM policy, role, and service account

Replace `111122223333` with your AWS account ID

```
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.4/docs/install/iam_policy.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

eksctl create iamserviceaccount \
  --cluster=krakend-app \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name "AmazonEKSLoadBalancerControllerRole" \
  --attach-policy-arn=arn:aws:iam::620583337209:policy/AWSLoadBalancerControllerIAMPolicy \
  --region ap-southeast-3 \
  --approve
```

### 3.3 Install controller

```
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=krakend-app \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set image.repository=296578399912.dkr.ecr.ap-southeast-3.amazonaws.com/amazon/aws-load-balancer-controller

kubectl get deployment -n kube-system aws-load-balancer-controller
```

Refer [here](https://docs.aws.amazon.com/eks/latest/userguide/add-ons-images.html) for installing image in other region.

## 4. Deploy sample app

`kubectl apply -f service.yaml`

Verify it works
```
kubectl exec --stdin --tty sample-app -n services -- /bin/sh
curl http://sample-app:8080
exit
```

## 5. Configure Krakend 

### 5.1 Create config file, save as krakend.json

Generate from [Kraken Designer](https://designer.krakend.io/), or alternatively follow this example:

```
{
    "version": 2,
    "extra_config": {
        "github_com/devopsfaith/krakend-gologging": {
            "level": "WARNING",
            "prefix": "[KRAKEND]",
            "syslog": false,
            "stdout": true
        }
    },
    "timeout": "3000ms",
    "cache_ttl": "300s",
    "port": 8080,
    "output_encoding": "json",
    "name": "apig",
    "endpoints": [
        {
            "endpoint": "/",
            "method": "GET",
            "backend": [
                {
                    "url_pattern": "/",
                    "host": [
                        "sample-service.services:8080"
                    ]
                }
            ]
        }
    ]
}
```

### 5.2 Create KrakenD config map

We are creating config map to inject KrakenD config dynamically. Alternatively, whenever we update KrakenD config, we'll have to push new KrakenD image to build a new image version, and re-deploy the KrakenD containers.

```
kubectl create configmap krakend-cfg --from-file=./krakend.json -n infra
kubectl describe configmap krakend-cfg -n infra
```

Updating krakend config:

```
kubectl create configmap krakend-cfg --from-file=./krakend.json -n infra --dry-run=client -o yaml | kubectl apply -f -
kubectl rollout restart deployment krakend-deploy -n infra
```

### 5.3 Spin up KrakenD containers and expose as service

```
kubectl apply -f apigateway.yaml
kubectl get svc apigateway -n infra
```

## 6. Other best practices

- Install EBS CSI Driver
- Configure node auto scaling (cluster autoscaler or Karpenter)