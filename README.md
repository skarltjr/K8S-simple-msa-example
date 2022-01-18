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

apiVersion: v1
kind: Service
metadata:
  labels:
    app: ui
  name: ui-svc
spec:
  type: NodePort
  selector:
    app: ui
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 30080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui
  labels:
    app: ui
spec:
  replicas: 1
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
        image: skarltjr/msa-ui:2.1
        env:
        - name: INFO_IP
          value: "172.30.4.73"  -> info컨테이너가 올라간 노드 ip
        - name: INFO_PORT
          value: "30490"    -> info 컨테이너 서비스의 접근 port
        - name: UI_PORT
          value: "30080"  -> 바로 위 nodePort서비스에서 지정한 접근port
        - name: UI_IP
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP  -> 현재 ui파드의 hostIP(노드ip)
        ports:
        - containerPort: 3000
~                                        
```
### 이슈 및 해결
- 쿠버네티스에서 deployment를 생성할 때 pod가 a노드에 생성될수도, b노드에 생성될수도 있다.
- 그럼 `UI컨테이너`는 당연히 정보를 얻기 위해 접근해야할 영화 정보 컨테이너가 어디에 생성될지 모른다
- 이를해결하고자 `환경변수`를 사용했다
- `UI컨테이너`는 `.env`파일을 통해 접근할 영화 정보 컨테이너의 ip,port를 `INFO_IP , INFO_PORT' 환경변수로 다룬다
- 따라서 위에서 볼 수 있듯이 UI컨테이너를 배포하기위한 yaml에서 env부분에서 이를 명시해줌으로써 위 이슈를 해결
- 다만 아직 부족하다고 느낀것은 위 상황은 결국 INFO컨테이너가 먼저 동작해야만한다는 제한사항


### 이슈 및 해결2
상황
```
사용자가 UI컨테이너에 접근한 후 이미지를 클릭하면 해당 영화정보를 영화 정보 컨테이너로부터 전달받아오고
이를 UI컨테이너에서 렌더링하여 유저에게 전달
문제는 <a href="http://172.30.4.73:30490/info/3"><div class="row">
-> 이미지 클릭시 아래 UI컨테이너의 로직에 어떻게 접근하느냐?였다
즉 ui파드가 생성된 노드ip는 가변적이며 그 때마다 현재 ui파드의 hostIP정보를 아래 여기!!에 적용해야했다
<a href="http://<여기!!> :30490/info/3"><div class="row">

- 아래는 접근하고자하는 UI컨테이너의 api
app.get('/info/:movieNum', function(req, res){
    var targetUrl = infoBaseUrl+req.params.movieNum
    var data = {
        movieNumber : 0,
        title : "",
        content : ""
    }
    // 기본이 비동기라 외부 api호출 종료를 보장 후 다음으로 넘어가야한다
    console.log('current target uri = '+targetUrl)
    var response = request('GET',targetUrl)

    var result = JSON.parse(response.getBody())
    data.movieNumber = result.movieNumber
    data.title = result.title
    data.content = result.content

    res.render('info', {title:data.title,content:data.content});
});
```
- 해결
```
공식문서를 다 뒤져보고 생각해보며 해결했다
https://github.com/skarltjr/MSA-ui-container/blob/main/movies/.env에서 볼 수 있듯이
ui컨테이너의 ip를 환경변수로 받도록 설정
이 후 UI컨테이너 yaml에서 맨 마지막 status.hostIP로 현재 
ui컨테이너가 위치한 노드의 ip를 env 환경변수에 넘겨주도록 했고 해결할 수 있었다

        env:
        - name: INFO_IP
          value: "172.30.4.73"  -> info컨테이너가 올라간 노드 ip
        - name: INFO_PORT
          value: "30490"    -> info 컨테이너 서비스의 접근 port
        - name: UI_PORT
          value: "30080"  -> 바로 위 nodePort서비스에서 지정한 접근port
        - name: UI_IP
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP  -> 현재 ui파드의 hostIP(노드ip)
```







