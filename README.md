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
- 쿠버네티스에서 deployment를 생성할 때 pod가 a노드에 생성될수도, b노드에 생성될수도 있다.
- 그럼 `UI컨테이너`는 당연히 정보를 얻기 위해 접근해야할 영화 정보 컨테이너가 어디에 생성될지 모른다
- 이를해결하고자 `환경변수`를 사용했다
- `UI컨테이너`는 `.env`파일을 통해 접근할 영화 정보 컨테이너의 ip,port를 `INFO_IP , INFO_PORT' 환경변수로 다룬다
- 따라서 위에서 볼 수 있듯이 UI컨테이너를 배포하기위한 yaml에서 env부분에서 이를 명시해줌으로써 위 이슈를 해결
- 다만 아직 부족하다고 느낀것은 위 상황은 결국 INFO컨테이너가 먼저 동작해야만한다는 제한사항
### 문제점
```
<a href="http://172.30.4.73:30490/info/1"><div class="row">
  <div class="column nature">
    <div class="content">
      <img src="assets/1.jpg" alt="Mountains" style="width:100%">
      <h4>Evil Dead 2013</h4>
      
    </div>
    </a>
  </div>
```
- 해당 이미지를 클릭하면 링크대로 아래를 수행
```
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

    res.send(result)
});
```
- 문제는 html에서 환경변수 사용법을 몰라서 직접 영화 정보 컨테이너의 ip, port를 명시해준것
- 여전히 해결 못했음.









