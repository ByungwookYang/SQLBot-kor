# GitLab CI/CD 자동 배포 시스템 구축 가이드

## 개요
Backend 코드를 GitLab에 업로드하고, 코드 변경 시 자동으로 Docker 컨테이너를 업데이트하는 CI/CD 시스템을 구축했습니다.

## 1. GitLab 저장소 설정

### 1.1 프로젝트 생성
- GitLab.com에서 새 프로젝트 `sqlbot` 생성
- Private 저장소로 설정

### 1.2 Personal Access Token 생성
- GitLab → Profile → Access Tokens
- Token name: `my-token`
- Scopes: `write_repository` 체크
- 생성된 토큰을 안전하게 저장

## 2. Backend 코드 업로드

### 2.1 Git 초기화 및 업로드
```bash
# backend 폴더에서 실행
cd /Users/yang/Downloads/sqlbot/sqlbot/backend
git init
git remote add origin https://gitlab.com/bw.yang/sqlbot.git
git add .
git commit -m "backend files upload"
git branch -M main
git push -f origin main
```

### 2.2 인증 설정
- Username: `bw.yang`
- Password: 생성한 Personal Access Token 입력

## 3. GitLab Runner 설정

### 3.1 GitLab Runner 설치 (macOS)
```bash
brew install gitlab-runner
```

### 3.2 Runner 등록
```bash
gitlab-runner register \
  --url https://gitlab.com \
  --token [프로젝트-토큰]
```

등록 시 입력 정보:
- GitLab URL: `https://gitlab.com`
- Description: `SQLBot Shell Runner`
- Tags: `paichb11x`
- Executor: `shell`

### 3.3 편의를 위한 alias 설정
```bash
echo "alias runner='gitlab-runner run'" >> ~/.zshrc
source ~/.zshrc
```

## 4. CI/CD 파이프라인 설정

### 4.1 .gitlab-ci.yml 파일 생성
GitLab 웹에서 `.gitlab-ci.yml` 파일 생성:

```yaml
stages:          
  - deploy
  - stop
  - start
  - restart
  - update-template
  - update-source
  - pip-list
  - pip-install
  - sqlbot-account-create

deploy-job-paichb11x:     
  stage: deploy  
  script:
    - echo "Deploying ..."
    - docker cp apps/chat/task/llm.py sqlbot:/opt/sqlbot/app/apps/chat/task/.
    - docker cp apps/system/api/user.py sqlbot:/opt/sqlbot/app/apps/system/api/.
    - docker cp common/utils/locale.py sqlbot:/opt/sqlbot/app/common/utils/.
    - docker cp apps/datasource/crud/datasource.py sqlbot:/opt/sqlbot/app/apps/datasource/crud/.
    - docker cp apps/db/db_sql.py sqlbot:/opt/sqlbot/app/apps/db/.
    - docker cp main.py sqlbot:/opt/sqlbot/app/.
    - docker cp template.yaml sqlbot:/opt/sqlbot/app/.
    - sleep 1
    - docker restart sqlbot
    - echo "Successfully deployed."
  tags:
    - paichb11x

restart:     
  stage: restart  
  script:
    - echo "restart SQLBot application..."
    - docker stop sqlbot
    - sleep 3
    - docker start sqlbot
    - echo "SQLBot successfully restarted."
  only:
    variables:
        - $RELEASE == "restart123"
  tags:
    - paichb11x

# 기타 job들 (stop, start, pip-install 등)도 동일한 패턴으로 설정
```

### 4.2 환경 변수 설정
GitLab → Settings → CI/CD → Variables:
- Key: `RELEASE`
- Value: `restart123`
- Type: Variable

## 5. Docker 컨테이너 설정

### 5.1 SQLBot 컨테이너 실행
```bash
sudo docker run -d \
  --platform linux/amd64 \
  --name sqlbot \
  --restart always \
  -p 8025:8000 \
  -p 8026:8001 \
  -p 8432:5432 \
  -e LANG=en_US.UTF-8 \
  -e LC_ALL=en_US.UTF-8 \
  -e LANGUAGE=en \
  -v $(pwd)/excel:/opt/sqlbot/data/excel \
  -v $(pwd)/images:/opt/sqlbot/images \
  -v $(pwd)/logs:/opt/sqlbot/logs \
  -v $(pwd)/pg:/var/lib/postgresql/data \
  sqlbot
```

## 6. 일상 사용법

### 6.1 컴퓨터 시작 시
```bash
# 1. Docker 상태 확인
docker ps

# 2. GitLab Runner 시작
runner
```

### 6.2 코드 배포
1. GitLab 웹에서 코드 수정
2. 커밋하면 자동으로 파이프라인 실행
3. Docker 컨테이너 자동 업데이트
4. 변경사항 즉시 반영

### 6.3 수동 파이프라인 실행
- GitLab → CI/CD → Pipelines → "New pipeline"
- Variables에 `RELEASE` = `restart123` 입력
- "Run pipeline" 클릭

## 7. 트러블슈팅

### 7.1 Runner 연결 문제
```bash
# Runner 상태 확인
gitlab-runner status

# Runner 재시작
runner
```

### 7.2 Docker 컨테이너 문제
```bash
# 컨테이너 상태 확인
docker ps

# 컨테이너 로그 확인
docker logs sqlbot

# 컨테이너 재시작
docker restart sqlbot
```

### 7.3 파이프라인 실패
- GitLab → CI/CD → Pipelines에서 실패한 job 클릭
- 에러 로그 확인
- 파일 경로나 권한 문제 해결

## 8. 주요 특징

- **완전 자동화**: 코드 변경 → 자동 배포
- **실시간 반영**: Docker 컨테이너 즉시 업데이트  
- **간단한 운영**: `runner` 명령어로 시작
- **안정적 운영**: Shell executor 사용으로 권한 문제 해결

## 9. 보안 고려사항

- Personal Access Token은 안전하게 보관
- GitLab Variables를 통한 민감 정보 관리
- 컨테이너는 `--restart always` 옵션으로 자동 재시작

이 시스템을 통해 GitLab에서 코드만 수정하면 자동으로 프로덕션 환경에 반영되는 완전한 CI/CD 파이프라인이 구축되었습니다.
