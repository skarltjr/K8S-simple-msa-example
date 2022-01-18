# K8S-simple-msa-example

### 목적
- 쿠버네티스와 도커를 활용한 간단 msa를 흉내내보자

### 개요
![image](https://user-images.githubusercontent.com/62214428/149953735-ad629f3c-d7a6-4f1a-97c9-f1540383ec5f.png)
- 간단한 영화정보 사이트를 msa형태로 구축한다
- 쿠버네티스 클러스터는 사전에 미리 1개의 마스터와 3개의 워커 노드로 구성


### 과정
#### 1. 영화정보, UI컨테이너를 위한 코드 구현
  - ETC에 표기된 소스코드 참고
#### 2. 쿠버네티스 deployment에서 사용할 도커 이미지로 생성
  - ETC에 표기된 도커허브 이미지 참고
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
- 영화 정보 
  - 소스 코드 / springboot: 
  - 이미지 : 
- UI  
  - 소스 코드/ node js:
  - 이미지 : 








