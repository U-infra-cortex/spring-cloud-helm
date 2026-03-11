# EKS Spring Cloud Helm 배포 및 문제 해결 가이드

이 문서는 AWS EKS 환경에서 Helm을 이용한 배포 과정 중 발생할 수 있는 주요 문제와 해결 방법을 정리한 가이드입니다.

---

## 🚀 1단계: 클러스터 인프라 확인 (EBS CSI 드라이버)

EKS 1.25 버전부터는 EBS 볼륨을 사용하기 위해 **EBS CSI 드라이버** 설치가 필수입니다.

### 1. 드라이버 설치 여부 확인
```bash
kubectl get pod -n kube-system | grep ebs-csi
```

### 2. 드라이버 설치 (없는 경우)
```bash
aws eks create-addon --cluster-name <클러스터명> --addon-name aws-ebs-csi-driver
```

### 3. IAM 권한(IRSA) 설정
드라이버가 정상 작동하려면 AWS EC2 API 호출을 위한 IAM 역할이 서비스 계정과 연동되어야 합니다.
- **IAM Role**: `AmazonEBSCSIDriverPolicy`가 포함된 역할 생성
- **Service Account**: `kube-system:ebs-csi-controller-sa`에 역할 ARN 주석 추가
  ```bash
  kubectl annotate sa ebs-csi-controller-sa -n kube-system eks.amazonaws.com/role-arn=arn:aws:iam::<AccountID>:role/<RoleName> --overwrite
  ```
- **재시작**: 드라이버 디플로이먼트 재시작
  ```bash
  kubectl rollout restart deployment ebs-csi-controller -n kube-system
  ```

---

## 📦 2단계: PVC (Persistent Volume Claim) 관리

이미 배포된 PVC의 설정(StorageClass 등)이 잘못되었을 때는 삭제 후 다시 생성해야 합니다.

### 1. 잘못된 PVC 삭제
```bash
kubectl delete pvc msa-postgres-pvc -n msa-ns
```

### 2. 올바른 설정으로 재배포
`values.yaml`에서 `storageClass: gp2`와 같이 클러스터가 지원하는 설정을 확인한 후 배포를 진행합니다.
```bash
helm template . -s templates/msa-postgres-pvc.yaml | kubectl apply -f - -n msa-ns
```

---

## 🛠️ 3단계: 장애 진단 및 복구 절차

배포 후 서비스가 정상화되지 않을 때의 핵심 체크리스트입니다.

### 1. 파드 상태 진단
```bash
kubectl get pods -n msa-ns
```
- **Pending**: 볼륨(PVC) 할당 대기 중. `kubectl describe pvc`로 상세 원인 파악.
- **CrashLoopBackOff**: 주로 데이터베이스(Postgres)가 아직 실행 전이라 애플리케이션 접속이 실패하는 경우 발생.

### 2. 로그 확인
```bash
kubectl logs <파드명> -n msa-ns
```

### 3. 서비스 강제 재시작 (Recovery)
데이터베이스가 정상 가동(`Running`)된 후에도, 이전에 실패했던 애플리케이션이 자동으로 정상화되지 않는 경우 명시적으로 재설정합니다.
```bash
kubectl rollout restart deployment spring-cloud-msa-msa-helm-user-deployment -n msa-ns
```

---

## 💡 요약 규칙
1. **PVC가 Pending이라면?** → StorageClass 이름과 EBS CSI 드라이버 구동 확인.
2. **DB 가동 후 앱이 계속 에러라면?** → 해당 앱 Deployment를 `restart` 하여 연결 갱신.
