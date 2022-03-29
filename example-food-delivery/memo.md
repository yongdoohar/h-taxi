## 참고
```
Cluster-Name: team03
Region-Code: ap-northeast-1
User-Account: team03
```

## IAM 보안 자격증명 설정
```
aws configure
```

## IAM 설정 확인
```
aws iam list-account-aliases
# Root계정의 별칭이 출력된다.
```

## AWS 클러스터 생성
```
eksctl create cluster --name team03 --version 1.21 --nodegroup-name standard-workers --node-type t3.medium --nodes 3 --nodes-min 1 --nodes-max 3
```

## AWS 클러스터 토큰 가져오기
```
aws eks --region ap-northeast-1 update-kubeconfig --name team03
kubectl get all
# 클러스터 설정확인
kubectl config current-context
```

## AWS 컨테이너 레지스트리(ECR) 생성
```
aws ecr create-repository --repository-name team03 --image-scanning-configuration scanOnPush=true --region ap-northeast-1
```

## ECR 생성 확인
```
aws ecr list-images --repository-name team03
```