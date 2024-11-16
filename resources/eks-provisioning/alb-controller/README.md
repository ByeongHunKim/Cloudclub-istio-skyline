```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

```bash
export region=ap-northeast-1
echo "Region: ${region}"

export clusterName=eks-cluster
echo "Cluster Name: ${clusterName}"

export policyArn=$(aws iam list-policies \
  --query 'Policies[?PolicyName==`AWSLoadBalancerControllerIAMPolicy`].Arn' \
  --output text)
echo "Policy ARN: ${policyArn}"
```

```bash
eksctl create iamserviceaccount \
 --region ${region} \
 --name aws-load-balancer-controller \
 --namespace kube-system \
 --cluster ${clusterName} \
 --attach-policy-arn ${policyArn} \
 --override-existing-serviceaccounts \
 --approve

#2024-11-09 20:42:30 [ℹ]  1 iamserviceaccount (kube-system/aws-load-balancer-controller) was included (based on the include/exclude rules)
#2024-11-09 20:42:30 [!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
#2024-11-09 20:42:30 [ℹ]  1 task: {
#    2 sequential sub-tasks: {
#        create IAM role for serviceaccount "kube-system/aws-load-balancer-controller",
#        create serviceaccount "kube-system/aws-load-balancer-controller",
#    } }2024-11-09 20:42:30 [ℹ]  building iamserviceaccount stack "eksctl-eks-cluster-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
#2024-11-09 20:42:30 [ℹ]  deploying stack "eksctl-eks-cluster-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
#2024-11-09 20:42:31 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
#2024-11-09 20:43:01 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
#2024-11-09 20:43:01 [ℹ]  created serviceaccount "kube-system/aws-load-balancer-controller"

k get sa -n kube-system
#aws-load-balancer-controller           0         49s
```

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=${clusterName} \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
  
#NAME: aws-load-balancer-controller
#LAST DEPLOYED: Sat Nov  9 20:45:59 2024
#NAMESPACE: kube-system
#STATUS: deployed
#REVISION: 1
#TEST SUITE: None
#NOTES:
#AWS Load Balancer controller installed!

k get po -n kube-system
#NAME                                            READY   STATUS    RESTARTS   AGE
#aws-load-balancer-controller-84d4546585-dfbts   1/1     Running   0          53s
#aws-load-balancer-controller-84d4546585-r76fd   1/1     Running   0          53s
```