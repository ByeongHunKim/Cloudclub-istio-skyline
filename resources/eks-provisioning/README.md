# AWS configure 설정
- default 설정 주의
  - 다른 AWS 계정으로 요청 보낼 수 있음
  - `aws sts get-caller-identity` 으로 계정 확인 후 진행
```bash
aws configure --profile staging-eks-user
# aws access key쌍 + region 입력

export AWS_PROFILE=staging-eks-user

aws configure get region
# ap-northeast-1 확인

aws sts get-caller-identity
```

# AWS VPC 생성
- 퍼블릭 서브넷만 구성
- 생성 전에 어떤 서브넷 가용영역이 열려있는 지 확인
```bash
cd vpc
aws cloudformation create-stack --stack-name eks-vpc-stack --template-body file://vpc-stack.yaml
#{
#    "StackId": "arn:aws:cloudformation:ap-northeast-1:00xxxxxxxxxx:stack/eks-vpc-stack/f7cfa8xx-9dxx-11xx-84xx-0aa2162282xx"
#}
```
# AWS VPC 생성 확인
- 해당 서브넷 값을 eks config 파일 생성에 동적 사용
```bash
VPC_STACK="eks-vpc-stack"

# Stack outputs 가져오기
outputs=$(aws cloudformation describe-stacks --stack-name $VPC_STACK --query "Stacks[0].Outputs" --output json)

# 개별 값 추출
VPC_ID=$(echo $outputs | jq -r '.[] | select(.OutputKey=="VPCId").OutputValue')
PUBLIC_SUBNET_1_ID=$(echo $outputs | jq -r '.[] | select(.OutputKey=="PublicSubnet1Id").OutputValue')
PUBLIC_SUBNET_2_ID=$(echo $outputs | jq -r '.[] | select(.OutputKey=="PublicSubnet2Id").OutputValue')
PUBLIC_SUBNET_3_ID=$(echo $outputs | jq -r '.[] | select(.OutputKey=="PublicSubnet3Id").OutputValue')

# 결과 출력
echo "VPC_STACK: $VPC_STACK"
echo "VPC_ID: $VPC_ID"
echo "PUBLIC_SUBNET_1_ID (AZ-a): $PUBLIC_SUBNET_1_ID"
echo "PUBLIC_SUBNET_2_ID (AZ-c): $PUBLIC_SUBNET_2_ID"
echo "PUBLIC_SUBNET_3_ID (AZ-d): $PUBLIC_SUBNET_3_ID"

#VPC_STACK: eks-vpc-stack
#VPC_ID: vpc-xxxxea3ebccdaxxxx
#PUBLIC_SUBNET_1_ID (AZ-a): subnet-xxxxc6bce5036xxxx
#PUBLIC_SUBNET_2_ID (AZ-c): subnet-xxxxea97b9a51xxxx
#PUBLIC_SUBNET_3_ID (AZ-d): subnet-xxxx743f6f9e0xxxx
```

# AWS EKS 생성
```bash


cat <<EOF > eksctl-cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: eks-cluster
  region: ap-northeast-1
  version: "1.29"
  tags:
    Environment: "Staging"
addons:
  - name: vpc-cni
    version: latest
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
iam:
  withOIDC: true
vpc:
  id: $VPC_ID
  subnets:
    public:
      ap-northeast-1a:
        id: $PUBLIC_SUBNET_1_ID
        name: "eks-subnet-public1-ap-northeast-1a"
      ap-northeast-1c:
        id: $PUBLIC_SUBNET_2_ID
        name: "eks-subnet-public2-ap-northeast-1c"
      ap-northeast-1d:
        id: $PUBLIC_SUBNET_3_ID
        name: "eks-subnet-public3-ap-northeast-1d"
managedNodeGroups:
  - name: manage-group
    instanceType: m4.xlarge
    desiredCapacity: 3
    minSize: 3
    maxSize: 5
    privateNetworking: false
    volumeSize: 20
    ssh:
      allow: true
      publicKeyName: "common"
    iam:
      withAddonPolicies:
        autoScaler: true
        ebs: true
        albIngress: true
    tags:
      eksctl.io/nodegroup-name: "manage-group"
  - name: service-group
    instanceType: m4.xlarge
    desiredCapacity: 1
    minSize: 1
    maxSize: 5
    privateNetworking: false
    volumeSize: 20
    ssh:
      allow: true
      publicKeyName: "common"
    iam:
      withAddonPolicies:
        autoScaler: true
        ebs: true
        albIngress: true
    tags:
      eksctl.io/nodegroup-name: "service-group"
EOF

eksctl create cluster -f eksctl-cluster-config.yaml


#2024-11-08 22:47:00 [ℹ]  eksctl version 0.169.0
#2024-11-08 22:47:00 [ℹ]  using region ap-northeast-1
#2024-11-08 22:47:01 [✔]  using existing VPC (vpc-xxxxea3ebccdaxxxx) and subnets (private:map[] public:map[ap-northeast-1a:{subnet-xxxxc6bce50367xxxx ap-northeast-1a 10.1.0.0/20 0 } ap-northeast-1c:{subnet-xxxxea97b9a51xxxx ap-northeast-1c 10.1.16.0/20 0 } ap-northeast-1d:{subnet-xxxx743f6f9e0xxxx ap-northeast-1d 10.1.32.0/20 0 }])
#2024-11-08 22:47:01 [!]  custom VPC/subnets will be used; if resulting cluster doesn't function as expected, make sure to review the configuration of VPC/subnets
#2024-11-08 22:47:01 [ℹ]  nodegroup "manage-group" will use "" [AmazonLinux2/1.29]
#2024-11-08 22:47:01 [ℹ]  using EC2 key pair "common"
#2024-11-08 22:47:01 [ℹ]  nodegroup "service-group" will use "" [AmazonLinux2/1.29]
#2024-11-08 22:47:01 [ℹ]  using EC2 key pair "common"
#2024-11-08 22:47:01 [ℹ]  using Kubernetes version 1.29
#2024-11-08 22:47:01 [ℹ]  creating EKS cluster "eks-cluster" in "ap-northeast-1" region with managed nodes
#2024-11-08 22:47:01 [ℹ]  2 nodegroups (manage-group, service-group) were included (based on the include/exclude rules)
#2024-11-08 22:47:01 [ℹ]  will create a CloudFormation stack for cluster itself and 0 nodegroup stack(s)
#2024-11-08 22:47:01 [ℹ]  will create a CloudFormation stack for cluster itself and 2 managed nodegroup stack(s)
#2024-11-08 22:47:01 [ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=ap-northeast-1 --cluster=eks-cluster'
#2024-11-08 22:47:01 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "eks-cluster" in "ap-northeast-1"
#2024-11-08 22:47:01 [ℹ]  CloudWatch logging will not be enabled for cluster "eks-cluster" in "ap-northeast-1"
#2024-11-08 22:47:01 [ℹ]  you can enable it with 'eksctl utils update-cluster-logging --enable-types={SPECIFY-YOUR-LOG-TYPES-HERE (e.g. all)} --region=ap-northeast-1 --cluster=eks-cluster'
#2024-11-08 22:47:01 [ℹ]  
#2 sequential tasks: { create cluster control plane "eks-cluster", 
#    2 sequential sub-tasks: { 
#        5 sequential sub-tasks: { 
#            wait for control plane to become ready,
#            associate IAM OIDC provider,
#            no tasks,
#            restart daemonset "kube-system/aws-node",
#            1 task: { create addons },
#        },
#        2 parallel sub-tasks: { 
#            create managed nodegroup "manage-group",
#            create managed nodegroup "service-group",
#        },
#    } 
#}
#2024-11-08 22:47:01 [ℹ]  building cluster stack "eksctl-eks-cluster-cluster"
#2024-11-08 22:47:02 [ℹ]  deploying stack "eksctl-eks-cluster-cluster"
#2024-11-08 22:47:32 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-cluster"
#2024-11-08 22:48:02 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-cluster"
#2024-11-08 22:49:02 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-cluster"
#2024-11-08 22:50:02 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-cluster"
#2024-11-08 22:51:03 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-cluster"
#2024-11-08 22:52:03 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-cluster"
#2024-11-08 22:53:03 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-cluster"
#2024-11-08 22:54:04 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-cluster"
#2024-11-08 22:55:04 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-cluster"
#2024-11-08 22:57:07 [ℹ]  daemonset "kube-system/aws-node" restarted
#2024-11-08 22:57:08 [ℹ]  creating role using recommended policies
#2024-11-08 22:57:08 [ℹ]  deploying stack "eksctl-eks-cluster-addon-vpc-cni"
#2024-11-08 22:57:08 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-addon-vpc-cni"
#2024-11-08 22:57:39 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-addon-vpc-cni"
#2024-11-08 22:57:39 [ℹ]  creating addon
#2024-11-08 22:57:50 [ℹ]  addon "vpc-cni" active
#2024-11-08 22:57:50 [ℹ]  building managed nodegroup stack "eksctl-eks-cluster-nodegroup-manage-group"
#2024-11-08 22:57:50 [ℹ]  building managed nodegroup stack "eksctl-eks-cluster-nodegroup-service-group"
#2024-11-08 22:57:51 [ℹ]  deploying stack "eksctl-eks-cluster-nodegroup-manage-group"
#2024-11-08 22:57:51 [ℹ]  deploying stack "eksctl-eks-cluster-nodegroup-service-group"
#2024-11-08 22:57:51 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-nodegroup-manage-group"
#2024-11-08 22:57:51 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-nodegroup-service-group"
#2024-11-08 22:58:21 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-nodegroup-manage-group"
#2024-11-08 22:58:21 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-nodegroup-service-group"
#2024-11-08 22:59:07 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-nodegroup-manage-group"
#2024-11-08 22:59:10 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-nodegroup-service-group"
#2024-11-08 22:59:44 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-nodegroup-service-group"
#2024-11-08 23:00:30 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-nodegroup-manage-group"
#2024-11-08 23:00:52 [ℹ]  waiting for CloudFormation stack "eksctl-eks-cluster-nodegroup-service-group"
#2024-11-08 23:00:52 [ℹ]  waiting for the control plane to become ready
#2024-11-08 23:00:53 [✔]  saved kubeconfig as "/Users/gimbyeonghun/.kube/config"
#2024-11-08 23:00:53 [ℹ]  no tasks
#2024-11-08 23:00:53 [✔]  all EKS cluster resources for "eks-cluster" have been created
#2024-11-08 23:00:53 [ℹ]  nodegroup "manage-group" has 3 node(s)
#2024-11-08 23:00:53 [ℹ]  node "ip-10-1-1-173.ap-northeast-1.compute.internal" is ready
#2024-11-08 23:00:53 [ℹ]  node "ip-10-1-25-216.ap-northeast-1.compute.internal" is ready
#2024-11-08 23:00:53 [ℹ]  node "ip-10-1-46-53.ap-northeast-1.compute.internal" is ready
#2024-11-08 23:00:53 [ℹ]  waiting for at least 3 node(s) to become ready in "manage-group"
#2024-11-08 23:00:53 [ℹ]  nodegroup "manage-group" has 3 node(s)
#2024-11-08 23:00:53 [ℹ]  node "ip-10-1-1-173.ap-northeast-1.compute.internal" is ready
#2024-11-08 23:00:53 [ℹ]  node "ip-10-1-25-216.ap-northeast-1.compute.internal" is ready
#2024-11-08 23:00:53 [ℹ]  node "ip-10-1-46-53.ap-northeast-1.compute.internal" is ready
#2024-11-08 23:00:53 [ℹ]  nodegroup "service-group" has 1 node(s)
#2024-11-08 23:00:53 [ℹ]  node "ip-10-1-20-137.ap-northeast-1.compute.internal" is ready
#2024-11-08 23:00:53 [ℹ]  waiting for at least 1 node(s) to become ready in "service-group"
#2024-11-08 23:00:53 [ℹ]  nodegroup "service-group" has 1 node(s)
#2024-11-08 23:00:53 [ℹ]  node "ip-10-1-20-137.ap-northeast-1.compute.internal" is ready
#2024-11-08 23:00:55 [ℹ]  no recommended policies found, proceeding without any IAM
#2024-11-08 23:00:55 [ℹ]  creating addon
#2024-11-08 23:01:24 [ℹ]  addon "coredns" active
#2024-11-08 23:01:25 [ℹ]  no recommended policies found, proceeding without any IAM
#2024-11-08 23:01:25 [ℹ]  creating addon
#2024-11-08 23:02:04 [ℹ]  addon "kube-proxy" active
#2024-11-08 23:02:05 [ℹ]  kubectl command should work with "/Users/gimbyeonghun/.kube/config", try 'kubectl get nodes'
#2024-11-08 23:02:05 [✔]  EKS cluster "eks-cluster" in "ap-northeast-1" region is ready
```

# 도메인 구매
- 가비아
- 네임서버 aws로 이전
  - 호스팅 영역 생성
  - 가비아의 도메인 설정에서 네임서버 값을 aws route53에 생성된 걸로 변경

# AWS ACM 발급
- 도메인 입력
- CNAME 생성

# common.pem 생성
- EC2 -> 키페어
- common
   - worker node ssh 접속 시 사용

# kubeconfig 설정 변경
```bash
aws eks --region ap-northeast-1 update-kubeconfig --name eks-cluster --alias cloudclub-cluster

k get nodes
#NAME                                             STATUS   ROLES    AGE   VERSION
#ip-10-1-1-173.ap-northeast-1.compute.internal    Ready    <none>   21h   v1.29.8-eks-a737599
#ip-10-1-20-137.ap-northeast-1.compute.internal   Ready    <none>   21h   v1.29.8-eks-a737599
#ip-10-1-25-216.ap-northeast-1.compute.internal   Ready    <none>   21h   v1.29.8-eks-a737599
#ip-10-1-46-53.ap-northeast-1.compute.internal    Ready    <none>   21h   v1.29.8-eks-a737599
```