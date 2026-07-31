# AWS 기반 컨테이너 이미지 취약점 개선 실습

## 실습 환경

- AWS CloudShell
- Amazon ECR (Private Repository)
- Amazon Inspector (Enhanced scanning)
- crane CLI (컨테이너 이미지 복사 도구, Docker 불필요)

## 사용 이미지

| 구분 | 이미지 |
|---|---|
| Before (취약) | `python:3.6-slim` |
| After (개선) | `python:3.12-slim` |

## 진행 절차

1. CloudShell에서 리전/계정 변수 설정
2. ECR 프라이빗 리포지토리 생성
3. Amazon Inspector Enhanced scanning 활성화
4. crane으로 `python:3.6-slim` → ECR `:before` 태그로 push
5. Inspector 자동 스캔 결과 확인 (Critical/High 수치)
6. crane으로 `python:3.12-slim` → ECR `:after` 태그로 push
7. 스캔 결과 재확인 및 Before/After 비교
8. 실습 종료 후 ECR 리포지토리, Inspector, 로컬 crane 삭제

## 주요 명령어

\`\`\`bash
export REGION=ap-northeast-2
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR=$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com

aws ecr create-repository --repository-name workshop/python-lab --region $REGION
aws inspector2 enable --resource-types ECR --region $REGION

aws ecr get-login-password --region $REGION | crane auth login --username AWS --password-stdin $ECR

crane copy --platform linux/amd64 \
  public.ecr.aws/docker/library/python:3.6-slim \
  $ECR/workshop/python-lab:before

crane copy --platform linux/amd64 \
  public.ecr.aws/docker/library/python:3.12-slim \
  $ECR/workshop/python-lab:after
\`\`\`

## 결과 비교

| 구분 | Critical | High |
|---|---|---|
| Before (3.6-slim) | 25 | 88 |
| After (3.12-slim) | 0 | 3 |

베이스 이미지를 3.6 → 3.12로 최신화한 것만으로 Critical 취약점이 전부 해소되고, High도 88건에서 3건으로 크게 줄어든 것을 확인했습니다.

### Before 스캔 결과
![before scan](./images/before_scan.png)

### After 스캔 결과
![after scan](./images/after_scan.png)

## 정리

\`\`\`bash
aws inspector2 disable --resource-types ECR --region $REGION
aws ecr delete-repository --repository-name workshop/python-lab --force --region $REGION
sudo rm -f /usr/local/bin/crane
\`\`\`

과금 방지를 위해 실습 종료 후 생성한 모든 AWS 리소스를 삭제했습니다.
