---
layout: post
title:  CI/CD Environment in GKE(Google Kubernetes Engine)
excerpt: 'Google Cloud Platform 을 활용한 CI/CD 환경 구축해본다.'
description: 'Google Cloud Platform 을 활용한 CI/CD 환경 구축해본다.'
comment: true
tag: [CI, CD, Kubernetes, k8s, Travis, Skaffold, GKE]
---

# Google Cloud Platform 을 활용한 CI/CD 환경 구축

## Prerequisite
---
- Git
- Docker
- Kubernetes
- Travis
- Skaffold

## Google Cloud Platform
---
### 클러스터 생성 및 Cloud Shell 활성화하기
- https://console.cloud.google.com/ 접속하여 'Console' 로 이동한다.
- 시작 메뉴에 'Kubernetes Engine' 선택한다.
- '클러스터'를 생성한다. (클러스터명 및 위치(Zone)만 기입하고 다른 것은 기본으로 둔다.)
- 클러스터가 정상적으로 생성되면 우측 상단에 'Cloud Shell 활성화'를 클릭한다.
  (Cloud Shell은 Prerequisite에 나열된 기술들에 대한 것을 모두 제공한다.)

### gcloud CLI를 통한 로그인
[YOUR_CLUSTER_NAME] 및 [YOUR_ZONE]은 위에서 생성한 클러스터의 클러스터명과 위치를 기입한다.

```
gcloud container clusters get-credentials [YOUR_CLUSTER_NAME] --zone [YOUR_ZONE]
```

### Git Fork & Clone
Github 에 있는 Spring boot 기반의 프로젝트를 본인의 Github에 Fork 한 후, clone 을 진행한다. (반드시 develop-gcr 를 받아야함)
프로젝트는 총 3개이다:
 - config-server : DB 정보를 담고 있는 ConfigMap
 - users-service : 유저 정보를 제공하는 서비스
 - users-detail-service : 유저에 대한 상세 정보를 제공하는 서비스

```
git clone https://github.com/[YOUR_GIT_ID]/spring-cloud-example-in-kubernetes.git
```

### Travis CI 설치
- GitHub marketplace(https://github.com/marketplace) 에 travis 검색해 설치 진행
  (Open Source 로 설치 진행한다. 그 외는 비용이 청구된다.)
- TravisCI 에서 Access 할 수 있는 Github Repository를 선택한다.
  ('Setting' >  'Applications' > 'Repository access')

### Travis CI 환경에서 빌드/배포를 위한 Google Cloud Build 및 Kubernetes Engine 권한 획득하기
- 아래 링크에 따라 'service-account' 를 생성한다. (json 형식으로 받는다.)
  (https://cloud.google.com/sdk/docs/authorizing?hl=ko#authorizing_with_a_service_account)
- 생성 뒤 json 파일을 다운로드 받는다.
- Cloud Shell 로 이동한다.
- 우측 상단에 '더보기' 선택 후, '파일 업로드'를 통해 .json 파일을 업로드한다.
- 업로드한 파일을 프로젝트 하위로 이동시킨다.

```
cd ~
mv [YOUR_JSON_FILE_NAME].json ~/spring-cloud-example-in-kubernetes/client_api_key.json
```

### Container Registry
이미지를 관리할 Registry 가 필요하다. 따라서 google 에서 지원하는 GCR(Google Container Registry) 를 사용한다.
Maven wrapper class를 통해 이미지를 빌드한다.
([YOUR_PROJECT_ID]는 Google Cloud Plaform Dashboard 상단에 My First Project를 클릭하면 ID를 확인 할 수 있다.)

```
./mvnw com.google.cloud.tools:jib-maven-plugin:build -Dimage=gcr.io/[YOUR_PROJECT_ID]/config-server:v1
./mvnw com.google.cloud.tools:jib-maven-plugin:build -Dimage=gcr.io/[YOUR_PROJECT_ID]/users-service:v1
./mvnw com.google.cloud.tools:jib-maven-plugin:build -Dimage=gcr.io/[YOUR_PROJECT_ID]/users-detail-service:v1
```

### sakffold.yml & deploy.yml 변경하기
[YOUR_PROJECT_ID]는 본인의 Project ID를 입력한다.

config-server
```
cd ~/spring-cloud-example-in-kubernetes/config-server
sed -i "s/nifty-inn-274523/[YOUR_PROJECT_ID]/g" skaffold.yml
sed -i "s/nifty-inn-274523/[YOUR_PROJECT_ID]/g" deploy-dev.yml
sed -i "s/nifty-inn-274523/[YOUR_PROJECT_ID]/g" deploy-stg.yml
```

users-service
```
cd ~/spring-cloud-example-in-kubernetes/users-service
sed -i "s/nifty-inn-274523/[YOUR_PROJECT_ID]/g" skaffold.yml
sed -i "s/nifty-inn-274523/[YOUR_PROJECT_ID]/g" deploy-dev.yml
sed -i "s/nifty-inn-274523/[YOUR_PROJECT_ID]/g" deploy-stg.yml
```

users-detail-service
```
cd ~/spring-cloud-example-in-kubernetes/users-service
sed -i "s/nifty-inn-274523/[YOUR_PROJECT_ID]/g" skaffold.yml
sed -i "s/nifty-inn-274523/[YOUR_PROJECT_ID]/g" deploy-dev.yml
sed -i "s/nifty-inn-274523/[YOUR_PROJECT_ID]/g" deploy-stg.yml
```

### .travis.yml 파일 변경하기
[YOUR_GIT_BRANCH_NAME] 는 본인이 빌드하고자 하는 branch 명을 기입한다.

```
cd ~/spring-cloud-example-in-kubernetes
vi .travis.yml
```

.travis.yml
```
branches:
  only:
    - [YOUR_GIT_BRANCH_NAME]
```

### TravisCI 에 환경변수 설정하기
- travisCI Home 이동
  (https://travis-ci.com/)
- 우측 중단에 'More options' > 'Settings' > 'Environment Variables' 에 아래 4가지 환경변수 추가:
  - GCLOUD_CLUSTER_NAME : Your GCLOUD Cluster Name
  - GCLOUD_ZONE: Your GCLOUD Zone Name
  - PROJECT_ID : Your GCLOUD Project ID
  - SERVICE_HOME : ${HOME}/build/[YOUR_GOOGLE_ID]/spring-cloud-example-in-kubernetes

### Git push
[YOUR_GIT_BRANCH_NAME]는 본인의 Git Branch 명을 기입한다.

```
git add --all
git commit -m 'travisCI test'
git push origin HEAD:[YOUR_GIT_BRANCH_NAME]
```

### 빌드/배포 확인하기
- Travis Home 이동
  (https://travis-ci.com/)
- 'Current' 탭에 현재 빌드되어 배포되는 로그를 확인 할 수 있다.
- Travis 가상환경에 필요한 설치 작업 진행
![Installation](https://raw.githubusercontent.com/atropos0116/atropos0116.github.io/master/assets/images/posts/gke-ci-cd-environment/travis_install.png)
- Travis 가상환경에서 gcloud 인증 과정
![GCLOUD Authorization](https://raw.githubusercontent.com/atropos0116/atropos0116.github.io/master/assets/images/posts/gke-ci-cd-environment/travis_gcloud_auth.png)
- Google Cloud Build를 이용한 이미지 빌드 과정
![GCLOUD Cloud Build Log](https://raw.githubusercontent.com/atropos0116/atropos0116.github.io/master/assets/images/posts/gke-ci-cd-environment/travis_skaffold_continuous_delivery_image_pull.jpg)
![GCLOUD Cloud Build Result](https://raw.githubusercontent.com/atropos0116/atropos0116.github.io/master/assets/images/posts/gke-ci-cd-environment/gke_deploy_service_result.jpg)
- Google Kubernetes Engine를 이용한 서비스 배포 과정
![Google Kubernetes Engine Log](https://raw.githubusercontent.com/atropos0116/atropos0116.github.io/master/assets/images/posts/gke-ci-cd-environment/travis_skaffold_continuous_delivery_service_deploy.jpg)
![Google Kubernetes Engine Result](https://raw.githubusercontent.com/atropos0116/atropos0116.github.io/master/assets/images/posts/gke-ci-cd-environment/gke_deploy_service_result.jpg)


### 서비스 확인하기
- 'Google Cloud Platform' > 'Kubernetes Engine' > '서비스 및 수신' > 'users-service' 의 '엔드포인트' 클릭
- [YOUR_URL]/users/atropos0116 입력하면 해당 유저의 상세정보 조회됨
