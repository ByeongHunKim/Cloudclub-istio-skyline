# EKS IAM Setup Commands Guide

## 1. IAM Policy 생성

```bash
# 정책 생성
aws iam create-policy \
    --policy-name eks-viewer-policy \
    --policy-document file://eks-viewer-policy.json
```

## 2. IAM 사용자 생성 및 정책 연결

```bash
# 정책 ARN 저장
POLICY_ARN=$(aws iam list-policies --query 'Policies[?PolicyName==`eks-viewer-policy`].Arn' --output text)

# 정책 ARN 확인
echo $POLICY_ARN

# IAM 사용자 생성 및 정책 연결
aws iam create-user --user-name esc-beep
aws iam attach-user-policy \
    --user-name esc-beep \
    --policy-arn $POLICY_ARN

aws iam create-user --user-name opp-13
aws iam attach-user-policy \
    --user-name opp-13 \
    --policy-arn $POLICY_ARN
```

## 3. Access Key 생성

```bash
# 각 사용자의 액세스 키 생성
aws iam create-access-key --user-name esc-beep
aws iam create-access-key --user-name opp-13
```

## 4. AWS CLI 프로필 설정

```bash
# AWS CLI 프로필 설정
aws configure --profile esc-beep
# AWS Access Key ID 입력
# AWS Secret Access Key 입력
# Default region name 입력 (예: ap-northeast-2)
# Default output format 입력 (예: json)

# 프로필 목록 확인
aws configure list-profiles
```

## 5. EKS Cluster 접근 설정

```bash
# AWS 프로필 변경
export AWS_PROFILE=esc-beep

# kubeconfig 업데이트
aws eks update-kubeconfig --name <cluster-name> --region <region>

# 권한 확인을 위한 테스트
aws eks list-clusters
aws sts get-caller-identity
```

## 권한 테스트

```bash
# EKS 클러스터 목록 조회 (성공)
aws eks list-clusters

# 자격 증명 확인 (성공)
aws sts get-caller-identity

# EC2 인스턴스 목록 조회 (실패 - 권한 없음)
aws ec2 describe-instances
```

## 주의사항

1. Access Key 생성 시 Secret Access Key는 한 번만 표시되므로 반드시 안전한 곳에 저장
2. 각 사용자는 개별 IAM 계정 사용 권장
3. AWS_PROFILE 환경변수를 설정하면 해당 프로필로 모든 AWS CLI 명령어가 실행됨
4. 이 설정으로는 EKS 클러스터 조회 관련 권한만 있으며, 다른 AWS 서비스 접근 불가
5. kubeconfig 업데이트 시 이전 설정은 덮어쓰기되므로 필요한 경우 백업 필요