# WHS 4기 컨테이너 보안 및 운영 과제
## AWS 기반 컨테이너 이미지 취약점 개선 실습

이번 실습에서는 AWS ECR과 Amazon Inspector를 이용하여 오래된 Python 이미지를 스캔하고, 최신 베이스 이미지로 변경했을 때 취약점이 얼마나 감소하는지 확인해 보았다.

## 실습 환경

- AWS CloudShell
- Amazon ECR (Private Repository)
- Amazon Inspector (Enhanced scanning)
- crane CLI (컨테이너 이미지 복사 도구, Docker 불필요)

## 사용 이미지

| 구분 | 이미지 |
|---|---|
| Before (취약) | python:3.6-slim |
| After (개선) | python:3.12-slim |

## 실습 과정

먼저 CloudShell에서 사용할 리전과 계정 정보를 변수로 설정했다.

이후 ECR Private Repository(`workshop/python-lab`)를 생성하고, Amazon Inspector의 Enhanced Scanning을 활성화했다. 처음에는 ECR 기본 스캔만 있는 줄 알았는데, 기본 스캔은 OS 패키지만 보고 Enhanced scanning(Inspector)은 npm/pip 같은 언어 패키지 의존성까지 잡아준다는 걸 알게 되어 Enhanced로 진행했다.

Docker 대신 `crane`이라는 CLI 도구를 이용해 이미지를 복사했다. Docker 데몬 설치 없이도 레지스트리 간 이미지 복사가 가능해서 CloudShell 환경에서 편하게 쓸 수 있었다.

```bash
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
```

참고로 처음에 `--platform` 옵션 없이 그냥 복사했더니 멀티 아키텍처 이미지(Image Index)가 통째로 올라가서 Inspector가 "UnsupportedImageError"를 띄우며 스캔을 못 했다. amd64 하나만 지정해서 다시 올리니 정상적으로 스캔됐다.

## 결과 비교

| 구분 | Critical | High |
|---|---|---|
| Before (3.6-slim) | 25 | 88 |
| After (3.12-slim) | 0 | 3 |

오래된 Python 3.6 이미지는 Critical과 High 취약점이 많이 발견되었다. 최신 Python 3.12 이미지로 변경한 뒤에는 Critical 취약점이 모두 사라지고 High도 크게 감소하는 것을 확인하였다. 애플리케이션 코드는 전혀 건드리지 않았는데도 이 정도 차이가 난다는 게 인상 깊었다.

### Before 스캔 결과
![before scan](./images/before_scan.png)

### After 스캔 결과
![after scan](./images/after_scan.png)

## 정리

실습 종료 후에는 불필요한 AWS 비용이 발생하지 않도록 생성했던 ECR Repository와 Inspector 설정을 삭제하고, crane도 제거하였다.

```bash
aws inspector2 disable --resource-types ECR --region $REGION
aws ecr delete-repository --repository-name workshop/python-lab --force --region $REGION
sudo rm -f /usr/local/bin/crane
```

## 실습하면서 느낀 점

이번 실습에서는 애플리케이션 코드를 변경하지 않고도 베이스 이미지만 최신 버전으로 바꾸는 것만으로 취약점을 크게 줄일 수 있다는 점을 확인했다.

평소에는 Docker 이미지를 받을 때 태그만 보고 선택했는데, 앞으로는 최신 LTS 이미지를 쓰고 있는지, 취약점 스캔 결과는 어떤지도 같이 확인해야겠다는 생각이 들었다.

또 Amazon Inspector는 이미지를 업로드하는 것만으로 자동으로 취약점을 분석해주기 때문에, 실제 운영 환경에서도 CI/CD 파이프라인에 넣어서 쓰면 활용도가 높을 것 같다는 생각이 들었다. 컨테이너 보안은 애플리케이션 코드뿐 아니라 베이스 이미지를 최신 상태로 유지하는 것도 중요하다는 걸 알게 됐다.
