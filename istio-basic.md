## 준비사항
- 클러스터에 istio 설치
- 영문 문자열을 리턴하는 app 작성 (sitpv-en)
- 한글 문자열을 리턴하는 app 작성 (sitpv-ko)
### APP 생성
- main.py
```python
from fastapi import FastAPI
app = FastAPI()
@app.get("/")
async def root():
    return {"message": "Hello SIT PV MSA"}
```
```python
from fastapi import FastAPI
app = FastAPI()
@app.get("/")
async def root():
    return {"메세지": "안녕하세요 SIT PV MSA"}
- requirements.txt
```text
anyio==3.4.0
asgiref==3.4.1
click==8.0.3
colorama==0.4.4
fastapi==0.70.1
h11==0.12.0
httptools==0.3.0
idna==3.3
pydantic==1.8.2
python-dotenv==0.19.2
PyYAML==6.0
sniffio==1.2.0
starlette==0.16.0
typing_extensions==4.0.1
uvicorn==0.16.0
watchgod==0.7
websockets==10.1
```
- Docker file
```Dockerfile
FROM python:latest
COPY ./api /api
WORKDIR /api
RUN pip install -r requirements.txt
EXPOSE 8080
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```
- 도커라이징
```bash
docker login
docker build -t zasmin/sitpv-en:1.0 .
docker push zasmin/sitpv-en:1.0
```
- K8S 배포용 YAML 파일 (sitpv.yaml)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sitpv-en
  labels:
    app: sit
    version: en
spec:
  containers:
  - image: zasmin/sitpv-en:1.0
    imagePullPolicy: IfNotPresent
    name: sitpv-en
---
apiVersion: v1
kind: Pod
metadata:
  name: sitpv-ko
  labels:
    app: uengine
    version: ko
spec:
  containers:
  - image: zasmin/sitpv-ko:1.0
    imagePullPolicy: IfNotPresent
    name: sitpv-ko
```
- pod 배포 및 확인
```bash
kubectl create namespace sit
kubectl label namespace sit istio-injection=enabled
kubectl create -f sitpv.yaml
kubectl get pod --show-labels
```
## CASES
아래 5개의 샘플을 통해 K8S 서비스 object와 istio object를 이해합니다.
- Case 1 : Kuberntes service 가 바라보는 endpoints 가 다수인 경우 트래픽은 endpoints 들로 자동 round robin 됩니다.
- Case 2 : Kuberntes service 에 spec.selector 로 라벨을 지정하여 라우팅 룰을 수동으로 지정할 수 있습니다.
- Case 3 : Istio VirtualService 를 활용하여 HTTP Request URI에 따라 각 pod 들로 라우팅 되도록 정의할 수 있습니다.
- Case 4 : Istio VirtualService 를 활용하여 각 pod 들로 라우팅 되는 비율을 정의할 수 있습니다.
- Case 5 : Istio DestinationRule 에 subset 을 구성하여 요청에 대한 destionation 을 정의합니다. Case 4와 동일한 결과를 리턴 받습니다.

### Case1
생성한 2개의 pod가 서로 같은 APP 이고 버전만 다르다고 가정하고 트래픽을 확인합니다.
```bash
kubectl expose pod sitpv-ko --port=8080 --type=LoadBalancer
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: sitpv-ko
  labels:
    app: hello
spec:
  selector:
    app: hello
  ports:
  - name: http
    protocol: TCP
    port: 8080
```
- service object 확인
```bash
kubectl get endpoints -l app=sit
```
- service object 에 트래픽 전달
```bash
for i in {1..10}; do curl http://172.18.70.3:8080; sleep 0.5 && echo -e "\n"; done
```

### Case2
istio 의 Route API를 사용하지 않고 servie object에 labeling을 통해 pod 라우팅룰을 지정할 수도 있습니다.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: sitpv-ko
  labels:
    app: hello
spec:
  selector:
    app: en
    version: v1
  ports:
  - name: http
    protocol: TCP
    port: 8080
```
- service object 확인
```bash
kubectl get endpoints -l app=en
```
### Case3
각 pod 를 바라보는 service object 를 생성하고 각 service 에 VirtualService CRDs 를 사용하여 라우트 룰셋을 정의하여 줍니다. 룰셋은 기본적으로 sitpv-ko 로 라우트 되지만 URI prefix가 /en 이면 sitpv-en 으로 라우트 되도록합니다.

- VirtualService 생성
기본적으로 svc sitpv-ko 로 라우트되고 URI prefix가 /en 이면 svc sitpv-en으로 라우트되는 룰셋
spec.hosts 는 대상 service
spec.http.route.destination 는 기본 라우트 service
spec.http.match.* 에 라우트 조건 지정
spec.*.destination.host 는 destination service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: sitpv-svc
  labels:
    app: hello
spec:
  selector:
    app: hello
  ports:
  - name: http
    protocol: TCP
    port: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: sitpv-ko
  labels:
    app: hello
spec:
  selector:
    app: hello
    version: ko
  ports:
  - name: http
    protocol: TCP
    port: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: sitpv-en
  labels:
    app: hello
spec:
  selector:
    app: hello
    version: en
  ports:
  - name: http
    protocol: TCP
    port: 8080
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: sitpv-vs
spec:
  hosts:
  - "sitpv-svc.sit.svc.cluster.local"
  http:
  - match:
    - uri:
        prefix: /en
    route:
    - destination:
        host: "sitpv-en.sit.svc.cluster.local"
  - route:
    - destination:
        host: "sitpv-ko.sit.svc.cluster.local"
```
- 트래픽을 생성합니다.
```bash
for i in {1..10}; do curl http://172.18.70.3:8080; sleep 0.5 && echo -e "\n"; done
for i in {1..10}; do curl http://172.18.70.3:8080/en; sleep 0.5 && echo -e "\n"; done
```
### Case4
VirtualService 는 Destination weight 스펙을 통해 라우트되는 비율을 정의할 수 있습니다.
- VirtualService 를 수정 적용
ko:en = 90:10 비율로 라우트 되도록 합니다.
spec..route..destination.weight 에 라우트 비율 정의 (%)
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: sitpv-vs
spec:
  hosts:
  - "sitpv-svc.sit.svc.cluster.local"
  http:
  - route:
    - destination:
        host: "sitpv-ko.sit.svc.cluster.local"
      weight: 90
    - destination:
        host: "sitpv-en.sit.svc.cluster.local"
      weight: 10
```
- 트래픽을 생성합니다.
```bash
for i in {1..20}; do curl http://172.18.70.3:8080; sleep 0.5 && echo -e "\n"; done
```
### Case5
pod 별 service 를 구성하는 대신 DestinationRule 을 이용하여 동일한 결과 구현합니다.
- DestinationRule 생성하여 subset 을 구성하고 VirtualService 에서 이를 지정합니다.
service 는 label app=hello 로 타켓 endpoints 정의
DestinationRule 에서 label version=v1, version=v2 으로 subset v1, v2 을 구성

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-hello
spec:
  host: sitpv-svc.sit.svc.cluster.local
  subsets:
  - name: ko
    labels:
      version: ko
  - name: en
    labels:
      version: en
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: sitpv-vs
spec:
  hosts:
  - "sitpv-svc.sit.svc.cluster.local"
  http:
  - route:
    - destination:
        host: "sitpv-svc.sit.svc.cluster.local"
        subset: ko
      weight: 90
    - destination:
        host: "sitpv-svc.sit.svc.cluster.local"
        subset: en
      weight: 10
```
```bash
for i in {1..20}; do curl http://172.18.70.3:8080; sleep 0.5 && echo -e "\n"; done
```
