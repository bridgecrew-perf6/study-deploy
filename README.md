
## Study-Deploy
- 목표 : `GitHub Action` + `CodeDeploy` + `Nginx`를 이용한 무중단 배포
- 참고 링크 : https://wbluke.tistory.com/39

---

## 무중단 배포 흐름도
![deploy.png](img/deploy.png)

### GitHub Action
1. GitHub Action에서 프로젝트 빌드
2. AWS S3에 jar파일 업로드
3. `AWS CodeDeploy`에 AWS S3에 업로드된 jar파일을 배포해달라고 요청

### AWS CodeDeploy
- EC2 인스턴스 내부에 있는 `CodeDeploy Agent`에 배포 명령 전달

### AWS CodeDeploy Agent
- 다음을 수행하는 배포 스크립트 준비 필요
  - 새로운 Spring Boot WAS 띄우기
  - Nginx 스위칭을 통한 무중단 배포
- S3의 `.jar`파일을 받아서, 주어진 스크립트에 따라 배포 진행

---

## GitHub Action 스크립트 작성
```yaml
name: logging-system

on:
  workflow_dispatch: # push를 통해 자동으로 배포하는 것이 아닌, Actions에서 수동으로 실행하여 배포하겠다는 뜻이다.

env:
  S3_BUCKET_NAME: logging-system-deploy220524 #S3의 버킷명을 여기다 적는다.
  PROJECT_NAME: study-deploy #프로젝트를 저장할 경로명을 적절히 입력

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v3
      
    - name: Set up JDK 11
      uses: actions/setup-java@v3
      with:
        java-version: '11'
        distribution: 'temurin'
        
    - name: Grant execute permission for gradlew
      run: chmod +x gradlew
      shell: bash
        
    - name: Build with Gradle
      uses: gradle/gradle-build-action@0d13054264b0bb894ded474f08ebb30921341cee
      with:
        arguments: build
      
    - name: Make zip file
      run: zip -r ./$GITHUB_SHA.zip .
      shell: bash

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}

    - name: Upload to S3
      run: aws s3 cp --region ap-northeast-2 ./$GITHUB_SHA.zip s3://$S3_BUCKET_NAME/$PROJECT_NAME/$GITHUB_SHA.zip
```
- S3에 버킷을 생성한다.
- S3에 대한 접근 권한을 가진 사용자를 생성
- repository의 secrets에 몇 가지 환경 변수를 추가
  - `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`
- 저장소의 Actions에서 적절히 배포 스크립트를 작성한다.
- Actions에서 배포를 수동 실행해보고, S3를 확인하여 빌드한 jar 파일의 압축본이 제대로 저장되어있는 지 확인한다.

---

## AWS CodeDeploy를 통한 배포

- 어플리케이션 최상단 경로에, `AppSpec.yml` 파일을 추가
  - `AppSpec.yml` : 배포에 관한 모든 절차를 적어둔 명세서
- `GitHub Action`이 `AWS CodeDeploy`에 프로젝트의 특정 버전을 배포해달라고 요청
- `CodeDeploy`는 배포를 진행할 EC2 인스턴스에 설치되어 있는 `CodeDeploy Agent`들에게 요청받은 버전을 배포 해달라고 요청.
- `Agent`들은 저장소에서 프로젝트 전체를 서버에 내려받고, AppSpec.yml이 명시한 절차대로 배포를 진행
- Agent는 배포를 진행하고, CodeDeploy에게 성공/실패 등의 결과를 전달

---

## EC2 인스턴스 셋팅
- SSH는 최소한의 고정 ip에서만(집 PC 정도) 접근할 수 있도록 허용
- HTTP,HTTPS 요청에 응할 수 있도록 인바운드 규칙에 80번, 443번 포트 개방
