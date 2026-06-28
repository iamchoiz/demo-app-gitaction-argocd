# demo-app-gitaction-argocd

> **GitHub Actions(CI) + ArgoCD(CD)** 기반 GitOps 파이프라인 데모 프로젝트  
> app-repo와 config-repo를 분리하여 GitOps 패턴을 실습합니다.

## Architecture

![Architecture](https://user-images.githubusercontent.com/77256060/166134200-6a4787c9-0719-49ad-a2cb-a11d7a6a3f3e.png)

---

## Tech Stack

| Category | Tool |
|---|---|
| CI | GitHub Actions |
| CD | ArgoCD |
| Container Registry | AWS ECR |
| App Packaging | Helm |
| Runtime | Tomcat (WAR), Nginx |

---

## 레포지토리 구성

| 레포 | 역할 |
|---|---|
| **app-repo** (this repo) | 애플리케이션 소스코드 + Dockerfile + GitHub Actions Workflow |
| **config-repo** | Helm values.yaml 관리 (이미지 태그 업데이트 대상) |

> app-repo와 config-repo를 분리하는 이유: 각 서비스 app-repo에 helm config가 함께 있으면 repo가 늘어날수록 관리가 복잡해집니다. config-repo를 표준화하여 한 곳에서 관리하는 것이 운영에 유리합니다.

---

## 사전 준비

**ECR 로그인 Secret 생성 (Kubernetes)**
```bash
kubectl create secret docker-registry ecr-cred \
  --docker-server=${aws_account_id}.dkr.ecr.ap-northeast-2.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region ap-northeast-2) \
  --namespace=default
```

**GitHub Secrets 등록**
| Secret | 설명 |
|---|---|
| `AWS_ACCESS_KEY_ID` | ECR 접근용 AWS 키 |
| `AWS_SECRET_ACCESS_KEY` | ECR 접근용 AWS 시크릿 |
| `GH_TOKEN` | config-repo push용 GitHub 토큰 |

---

## CI/CD 워크플로우

### Step 1 — 개발자 코드 푸시 (app-repo)

개발자가 소스를 수정하고 app-repo에 `git push`합니다.

### Step 2 — GitHub Actions CI

![GitHub Actions CI](https://user-images.githubusercontent.com/77256060/166134228-e69d6e9d-252b-4167-a176-f9a34ad7e74a.png)

1. AWS Credentials 설정 (GitHub Secret의 AWS Key 사용)
2. AWS ECR 로그인
3. 환경변수 설정: `ECR_REGISTRY`, `ECR_REPOSITORY`, `IMAGE_TAG` (`VERSION` 파일 기반)
4. Docker 이미지 빌드 후 ECR 푸시

### Step 3 — GitHub Actions CD

![GitHub Actions CD](https://user-images.githubusercontent.com/77256060/166134237-d94cb006-9c6b-4276-ba7b-b9a1e0da17ad.png)

1. config-repo를 클론
2. Helm `values.yaml`의 이미지 태그를 새 버전으로 수정
3. 수정된 `values.yaml`을 config-repo에 다시 푸시 (`GH_TOKEN` 필요)

### Step 4 — ArgoCD 자동 동기화

![ArgoCD Sync](https://user-images.githubusercontent.com/77256060/166134243-d8b0e291-123e-4045-adba-442372aa574a.png)

- ArgoCD가 config-repo를 **3분마다 polling**하여 변경 감지 시 자동 배포
- 또는 GitHub Webhook을 ArgoCD에 연동하여 즉시 동기화 가능
