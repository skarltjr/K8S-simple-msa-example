# K8S-simple-msa-example

### 목적
- 쿠버네티스와 도커를 활용한 간단 msa를 흉내내보자

### 개요
![image](https://user-images.githubusercontent.com/62214428/149953735-ad629f3c-d7a6-4f1a-97c9-f1540383ec5f.png)
- 간단한 영화정보 사이트를 msa형태로 구축한다
- 쿠버네티스 클러스터는 사전에 미리 1개의 마스터와 3개의 워커 노드로 구성
- 진행 순서
  - 사전에 영화 정보를 저장해둔다
    - 영화 정보 컨테이너도 nodePort로 외부에서 접속하도록 설정하였기에 가능
  - 클라이언트가 UI컨테이너에 접근
  - 원하는 영화 카드 선택
  - 해당 영화에 대한 초간단 정보 반환받는다
    - 이 때 UI컨테이너는 영화 정보 컨테이너에 접근
    - 해당 영화에 대한 정보를 영화 정보 컨테이너로부터 전달받아오고
    - 이를 클라이언트에게 "UI"컨테이너가 전달


### 과정
#### 1. 영화정보, UI컨테이너를 위한 코드 구현
  - 영화 정보 소스 코드 (springboot): https://github.com/skarltjr/MSA-info-container
  - UI 소스 코드 (node js) : https://github.com/skarltjr/MSA-ui-container
#### 2. 쿠버네티스 deployment에서 사용할 도커 이미지로 생성
  - 영화 정보 컨테이너 이미지(도커허브) : skarltjr/msa-info:1.0  / https://hub.docker.com/r/skarltjr/msa-info
  - UI컨테이너 이미지(도커허브) : skarltjr/msa-ui:2.0 / https://hub.docker.com/repository/docker/skarltjr/msa-ui
#### 3. 쿠버네티스에서 배포를 위한 deployment yaml 작성
```
info 컨테이너 

apiVersion: apps/v1
kind: Deployment
metadata:
  name: info
  labels:
    app: info
spec:
  replicas: 1
  selector:
    matchLabels:
      app: info
  template:
    metadata:
      labels:
        app: info
    spec:
      containers:
      - name: info
        image: skarltjr/msa-info:1.0
        ports:
        - containerPort: 8080
```

```
★ ui-컨테이너

apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui
  labels:
    app: ui
spec:
  replicas: 1 -> 레플리카는 임의적이로 1개로 설정
  selector:
    matchLabels:
      app: ui
  template:
    metadata:
      labels:
        app: ui
    spec:
      containers:
      - name: ui
        image: skarltjr/msa-ui:2.0 -> UI 컨테이너를 위해 만들어둔 이미지 활용
        env:
        - name: INFO_IP  ->  매번 변하는 영화정보 컨테이너 정보를 환경변수를 활용하여 대응 -> 이슈 및 해결에서 좀 더 상세하게 설명
          value: "172.30.4.73"
        - name: INFO_PORT
          value: "32240"
        ports:
        - containerPort: 3000
```
### 이슈 및 해결

### 문제점

### ETC

- UI  
  - 소스 코드/ node js:
  - 이미지 : 








