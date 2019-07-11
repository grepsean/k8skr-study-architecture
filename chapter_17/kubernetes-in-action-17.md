# 17장 애플리케이션 개발 모범 사례

2019-07-10
윤평호

> 해당 파트의 그림은 책을 참조하십시요.
---
## 1. All together

![drawing](17.1 Resources in a typical application.png "Resources in typical application")
- podTemplate, podLifecycleHook, terminating, easyManagement, initContainer, localDevelopment

---
### PodTemplate

- 디플로이먼트 + 스테이트풀셋(상태)
  파드템플릿 형식으로

---
### Probe

Probe?
- Kubelet에서 파드의 정상 동작 진단 기능

|type|condition|
|-|-|
|ExecAction| exitcode 0 or die|
|TCPSocketAction| connect or die|
|HTTPGetAction| HTTP code 200 or die|

- livenessProbe, readnessProbe

---
### livenessProbe
- 컨테이너 동작 여부 확인
  ```yaml
  livenessProbe:
    httpGet:
      path: /healthz
      port: 8080
      httpHeaders:
      - name: Coach
        value: Yoon
    initialDelaySeconds: 5 # 준비시간
    periodSeconds: 5 # 간격
  ```
- 실패시? 해당 컨테이너는 재시작 정책 대상

---
### readnessProbe

- 요청 처리할 준비 여부 확인
  ```yaml
  readnessProbe:
    exec:
      command: ["cat /tmp/healthy"]
    ...
  ```
- 실패시? 엔드포인트 제거, 로딩 완료 대기

---
### Service

애플리케이션 서비스를 외부로 노출
 - LoadBalancer: cloud-provider
 - NodePort
 - HostPort
 - Ingress

---
### Private Image Registry

1. docker login
   ~/.docker/config.json 
2. regcred 시크릿 생성 및 사용 .red[*]
    ```bash
    kubectl create secret generic regcred \
        --from-file=.dockerconfigjson=~/.docker/config.json \
        --type=kubernetes.io/dockerconfigjson
    ```
3. ImagePullSecrets 지정
    ```yaml
    containers:
    - image: <...>
        imagePullSecrets:
    *   - name: regcred
    ```

.footnote[
.red[*] 시크릿은 ServiceAccount에 할당, 애플리케이션 메니페스트와 별개 구성
]

---
### ConfigMap

환경 변수, 설정 파일 등으로 사용 가능

```yaml
...
kind: ConfigMap
metadata:
  name: application-config
data:
  <LOG_LEVEL>: info
  <LOG_TYPE>: json
```

```yaml
...
containers:
  envFrom:
    configMapRef: applicaion-config
  volume:
  ...
```
---
### Volume

다양한 형식의 볼륨 사용
- emptyDir: 빈 볼륨, 포드 제거시 삭제
  - 파드 내에 데이터 공유 가능 .red[*]
  - 파드 제거 안되면 데이터 유지
  - 메모리로도 가능 .red[**]
- gitRepo: X
- PersistentVolume(PV)
  PersistentVolumeClaim(PVC)으로 요청하면 StorageClass에 따라 PV를 생성/검색 후 바인딩

.footnote[
.red[*] https://kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/

.red[**] emptyDir.medium: Memory, https://kubernetes.io/docs/concepts/storage/volumes/
]
---
### Job/CronJob/Daemonset

사용할 수 있다?
- **데몬셋 사용 권장**

데몬셋?
- 노드별 동작이 필요한 경우
---
### HorizontalPodAutoscalers
- 꼭 LimitRange, ResourceQuota 생성

---
### 자동생성 오브젝트

- 엔드포인트 컨트롤러가 생성한 서비스 엔드포인트 오브젝트
- 디플로이 컨트롤러가 생성한 레플리카셋
- 레플리카셋(또는 잡,크론잡,스테이트풀셋,데몬셋)에서 생성한 파드

---
### 레이블 사용하기

클러스터 관리를 위해 필수.
- 연락처, 관련 도구, 링크 등 다양하게 넣자

---
## 2. 파드 라이프사이클 이해하기
### 애플리케이션은 종료되고 재배치된다.
- 쿠버네티스를 사용하면, 애플리케이션을 이전과 다른 머신에서 구동할 수 있다.
- 머신 의존성이 없어야 한다.

---
### 변경될 IP 주소, 호스트 이름
- 새 파드는 새로운 IP 주소, 호스트명이 부여됨
- 상태저장이 필요없는 애플리케이션은 보통 문제 없다.
- 상태저장을 해야 하면, 스테이트풀셋 사용
  - 동일한 호스트명 유지
  - 단, IP 주소는 변경
- IP 주소에 의존적이면 안됨.
  - 호스트명 기반(DNS)의 스테이트풀셋 사용

---
### 디스크에서 데이터 사라짐

- PV가 아니라면, 오토스케일링을 통해 기존 데이터를 사용할 수 없다.
- 물론, 파드 라이프 사이클 동안에는 유지(버그가 있다면?)
![drawing](17.2 Files written to container's filesystem are lost when the container is restarted.png "files lost in restarting")

> kubelet은 같은 컨테이너를 다시 실행하지 않고, 항상 새 컨테이너를 생성.

---
### 데이터 보존을 위한 볼륨 사용

![drawing](17.3 Using a volume to persist data across container restarts.png "files safe in persistent volume")

- 항상 좋은 것은 아니다? 볼륨을 사용하는 것은 양날의 검
  - 저장된 데이터가 잘못된 경우는? CrashLoopBackOff? OTL...
  - 볼륨을 사용하지 않았다면, emptyDir 정도만 사용했다면? 크래시가 발생하지 않을 가능성이 크다.

---
### 2.2 죽은 파드 재스케쥴링

Kubelet은 크래시 발생한 파드는 정상이 될 때까지 무한정 재시작
- restartPolicy에 따라 back-off delay 수행 후, 지수값이 5분(320초)면 재설정
  10,20,40,... 320 = 10분
- 레플리카셋은 크래쉬라도 재스케쥴 하지 않음

![drawing](17.4 A ReplicaSet controller doesn’t reschedule dead pods.png "replicaset fail")

- 재스케쥴링한다고 문제를 해결하는 것은 아니라서

---
### 2.3 원하는 순서로 파드 시작

- YANL/JSON에 기록된 순서로 오브젝트 처리(etcd에도 기록)
- 단, 파드가 ready 하기까지의 순서 보장은 없다.
   - initContainer 포함시켜서 주 컨테이너가 시작되는 것을 조정할 수 있다.

---
#### initContainer

- 파드를 초기화 하는데 사용할 수 있는 컨테이너
  - 주 컨테이너와 볼륨을 공유할 수 있다.
- 여러 initContainer 가능
  - 정의한 순서대로 동작

---
#### Pod dependency

- initContainer를 쓰는 것보다 의존하는 서비스 준비가 필요없는 것이 낫다.

- readinessProbe 를 이용하면, 서비스 엔드포인트를 추가하지 않을 뿐만 아니라 롤링 업데이트에도 롤아웃을 방지할 수 있다.

---
### 2.4 파드 라이프사이클 훅

- 컨테이너 별로 실행되는 훅
  - post-start / pre-stop
  - 내부 명령어 혹은 HTTP GET 을 통해서

- 라이브 사이클 훅 실행 결과는 쿠버네티스에 로깅되지 않음
  - 실패시 FailedPostStartHook 표기되고
  - exit code 관련 로그 밖에 없음
  - emptyDir 볼륨을 이용해서 로그를 저장

---
#### post-start hook

- 컨테이너의 주프로세스가 시작된 다음에 실행
  - 추가 작업이 있어서, 기존 이미지 수정없이 동작
  - 병렬로 실행되나, 컨테이너의 주프로세스가 실행완료까지 기다릴 수 없음
  - 훅 완료시까지 ContainerCreating 상태, Pod 는 Pending
  - 실행되지 않거나, 0 아닌 exit code 면 주 컨테이너도 종료

---
#### pre-stop hook

- 컨테이너가 종료되기 직전에 실행
  - pre-stop 훅 이후에 컨테이너 주프로세스로 SIGTERM 보냄
  - 실패시, FailedPreStopHook 이 표기되나 컨테이너가 곧바로 종료되어 못 보고 지나칠 수 있다.
  - 종료 확인 완료가 중요한 경우, 전체 시스템이 올바르게 실행되고 있는지 확인 필요

---
#### SIGTERM 처리하지 못해 pre-stop hook 사용

- SIGTERM이 컨테이너 내부의 애플리케이션에 전달되지 않을 수 있다.
- CMD에서 쉘인 경우, SIGTERM 을 쉘이 받을 수도 있다.
- 쉘에서 받을 것이 아니라면, 컨테이너 구동 시 애플리케이션 바이너리를 직접 실행하도록
  ENTRYPOINT 나 CMD 에서 ["/mybinry"], ENTRYPOINT /mybinary

---
#### 컨테이너를 위한 라이프사이클 훅

- 라이프사이클 훅은 컨테이너를 위한 것이다. 
  - POD 종료하는 작업을 위해 pre-stop 훅을 사용해서 안된다.
  - 컨테이너 단위 명령이니까 여러번 실행 가능
  - 반복 실행으로 인해서 문제 발생

---
### 파드 셧다운

- API 서버의 파드 객체 삭제로 트리거된다.
- HTTP DELETE 요청을 받으면, 객체를 바로 삭제하지 않고 deletionTimestamp 필드 설정
  - kubelet이 종료 요청을 인식하면, 파드의 컨테이너를 종료하기 시작
    - 각 컨테이너에 정상적인 종료를 위해 제한된 시간(terminationGracePeriodsSeconds)이 주어짐
    - 단, 파드 단위로 설정
  
![drawing](17.5 The container termination sequence.png "The container termination sequence")

---
#### 종료 유예기간

파드 스펙에서 terminationGracePeriodsSeconds 필드로 설정
- 기본 30초
- 충분한 시간을 설정하도록
- 모든 컨테이너 종료시 kubelet 이 API 서버에 알리고, 파드 개체 및 리소스는 삭제됨

```shell
kubectl delete po mypod --grace-period=5

kubectl delete po mypod --grace-period=0 --force
```

- 스테이트풀셋의 파드는 강제 종료 사용에 주의해야 한다.
  - 파드가 실행 중이지 않거나
  - 접속이 끊어졌을 때(i.e. NodeLost)만 강제 종료하자.

---
#### 적절한 셧다운 핸들러

애플리케이션 종료 시간을 예측 불가?
- 스케일 다운시에는 파드가 삭제되어 종료됨

만약, 그 애플리케이션이 분산 스토리지라면?
- (파드 삭제시 데이터도 유실되니) 나머지 파드로 마이그레이션 필요하나
  다음 사유로 권장하지 않음
  - 컨테이너 종료가 전체 파드 종료는 아니다.
  - 시간 내에 종료절차가 끝날 것이라는 보장은 없다.
    - 종료 유예기간 만료, 종료 중 노드 실패
    - 노드 실패시, 노드 복구되어도 kubelet 은 종료절차를 다시 시작하지 않음

---
#### 셧다운 작업 전용 파드

종료시에 반드시 실행할 셧다운 작업을 완전하게 보장할 방법은?
- 종료 신호시, 삭제될 파드의 데이터를 나머지 파드로 마이그레이션하는 잡 생성?
- 끊임없이 실행되는 전용 파드를 이용해서 분리된 데이터를 확인
  - 전용 파드 대신 CronJob?


---
#### 셧다운 작업 전용 파드 - 스테이트풀셋
- 스케일 다운되면 PersistentVolumeClaim이 고아가 되고, PersistentVolume 데이터는 남음
  스케일 업되면 해당 PersistentVolume에 연결. 언제?

- 애플리케이션 업그레이드 중에 마이그레이션이 발생하지 않도록 구성.
  파드 복구를 위한 시간을 줘야 함.

![drawing](17.6 Using a dedicated pod to migrate data.png "Using a dedicated pod to migrate data")

---
## 3. 올바르게 처리되도록 보장
### 3.1 시작시 클라이언트 끊어짐 방지

- 파드 시작시 파드 레이블과 일치하는 셀렉터를 가진 모든 서비스에 엔드포인트 추가
  - 파드가 준비됨을 알리기 전에는 서비스 엔드포인트는 추가되지 않아, 어떠한 요청도 받지 않음

- 준비 신호(readinessProbe)를 지정하지 않으면, 항상 준비상태로 간주
  - kube-proxy가 서비스 규칙을 갱신하면 거의 즉시 요청을 받을 수 있다!
    `connection refused` 같은 오류를 경험할 수도 있으니, `readinessProbe` 지정하자

---
### 3.2 셧다운시 클라이언트 끊어짐 방지 (일부 흐름 생략)

<div class="mermaid">
sequenceDiagram
  participant 클라이언트
  participant API서버
  participant kubelet
  participant EndPoints 컨트롤러
  participant kubeProxies
  클라이언트->>API서버: 파드 삭제 요청
  API서버->>API서버: etcd에 상태저장
  API서버-->>클라이언트: 처리완료
  API서버-->kubelet: 파드 삭제 감지(비동기, A이벤트)
  kubelet->>kubelet: 컨테이너 종료: prestop hook -> SIGTERM -> 종료유예기간 초과시 강제 종료
  API서버-->EndPoints 컨트롤러: 파드 삭제 감지(비동기, B이벤트)
  EndPoints 컨트롤러->>API서버: 해당 파드 관련 엔드포인트 삭제 요청
  API서버->>API서버: etcd에 상태저장
  API서버-->kubeProxies: 엔드포인트 변경 감지(비동기, B이벤트 후속 이벤트)
  kubeProxies->>kubeProxies: 해당 파드 관련 iptables 삭제
</div>

- 문제는?

.footnote[
* iptables 수정되어도 기존 연결은 끊어지지 않음 :)
]

---
### 3.2 셧다운시 클라이언트 끊어짐 방지 - 해결

Why? (전체 노드에서) iptables 갱신되기 전에, 애플리케이션이 먼저 종료되어, 애플리케이션 요청은 받을 수 있음 :(

대안으로 readinessProbe?
- SIGTERM 수신 이후 readinessProbe 실패하면 되지 않나?
  - readinessProbe는 파드가 서비스 엔드포인트에서 제거되어야 함
  - 그러나? readinessProbe는 연속으로 실패해야 실패로 인정하고 제거함

- 문제를 완벽히 해결할 수 있는 방법은 없다. :(
  - kube-proxy 가 다 잘 해주기를 기다리자
  - 그런데 iptables 갱신도 느리다. 20k 서비스(160k 룰 추가), 5시간 걸린다.
     - 서비스 불가? X

- 5~10초 지연시간을 추가하는 것으로도 사용자 경험이 상당히 개선된다.

---
### 셧다운에 대한 정리

적절한 셧다운 절차
- 새 연결 수락하기 몇 초간 기다린다.
- 요청 중간에 있지 않은 모든 keep-alive 연결을 닫는다.
- 모든 활성 요청이 완료되기까지 기다린다.
- 이후에 셧다운 한다.

SIGTERM 수신 후 바로 종료하는 것은 쉽지 않다. 관리자가 결정해야 한다.

- (코드 수정없이 일반적으로) 가장 좋은 방법? pre-stop 훅 추가

  ```yaml
  lifecycle:
    preStop:
      exec:
        command: ["sh", "-c", "sleep 5"]
  ```

---
## 4. 쉬운 실행 관리
### 4.1 관리 가능한 컨테이너 이미지 만들기

이미지 패키징시, 바이너리, 추가 라이브러리, 전체 OS도 포함 가능
- debian 같은 경량 OS를 사용할 수도 있지만, ubuntu 같은 이미지도 가능

작은 크기의 이미지?
- 파드 배포나, 스케일링 주기가 빠르면 권장
- 디버깅 도구(curl, ping, dig 등)가 없어 불편
  - 구미에 맞게 스태틱 빌드 디버깅 도구 추가

---
### 4.2 적절히 태그, imagePullPolicy 현명하게

기본 도커 이미지 태그 사용하지 말것
- 기본은 latest. 새로운 이미지를 푸시해도 기존 파드는 기존 이미지 유지
- 이전 버전이 없으니, 롤백도 힘듬

동일 태그를 사용한다면?
- imagePullPolicy 를 Always 로 한다. 단,
  - 변경 체크로 인해 파드 시작이 느려질 수 있다.
  - 도커 레지스트리 연결 불가시 파드 실행도 불가

---
### 4.3 다차원 레이블 사용

모든 리소스에 레이블 지정할 수 있다. 잊지 말자.
- 리소스 개수가 많아질수록 레이블이 다양한 것이 좋다.

레이블에는 어떤 것이?
- 리소스가 속한 애플리케이션 이름(i.e app: nginx)
- 프론트엔드, 백엔드 등의 애플리케이션 계층
- 개발, 스테이지, 프로덕션 등의 환경
- 버전
- 릴리즈 유형 (블루그린, 카나리아, 롤링 등)
- 테넌트 (네임스페이스 대신 테넌트 별도로 파드를 실행할 경우)
- 샤드
- ...

---
### 4.4 어노테이션 사용

레이블 이외에도 리소스에 어노테이션으로 정보 추가할 수 있다.
- 적어도 리소스 사용자/작성자의 연락처는 있기를 권장
- 다른 서비스 이름을 포함하는 어노테이션을 추가하여 파드 종속성 표기
- 빌드 버전, 메타 데이터 등
- 애플리케이션 충돌에 원인 파악을 위해 많은 정보가 있는 것이 좋다.

---
### 4.5 종료 로그 포함

컨테이너의 종류 원인을 알 수 있어야 한다.
- 로그 파일에는 필요한 모든 디버깅 정보를 포함하는 것을 권장
- 로그에 종료 이유를 표기하면, `kubectl describe pod`의 출력으로 확인
- 종료 로그 파일 위치는 `terminationMessagePath` 필드로 지정
  - 기본값은 `/dev/termination-log`
- `terminationMessagePolicy`이 `FallbackToLogsOnError`이면 로그의 마지막 몇줄이 종료 메시지

---
### 4.6 로그 핸들링

로그를 표준 출력으로 로그를 출력하면, kubectl logs 로 확인 가능
- 이전 컨테이너의 로그는 --previous 스위치

로그 파일로 기록하면?
- kubectl exec <pod> cat <logfile>

---
#### 로그 파일 복사
#### 중앙 집중식 로깅
- 쿠버네티스는 중앙 집중식 로그를 지원하지 않음
    - 다양한 서드파티 사용

- ELK(Elasticsearch + logstash + fluentd) 사용

---
#### 여러줄로 된 로그 문구 다루기
- fluentd 에이전트는 로그를 각 행을 es 에 저장
  자바 스택 트레이스같이 여러 행에 걸치는 로그는? JSON 출력으로 변경
  - JSON 방식은 kubectl logs로 알아보기 어려움
    - 2가지 로그 사용
      - 사람이 읽을 수 있는 표준 출력 로그
      - JSON 으로 쓰는 fluentd 로그

---
## 5. 개발/테스트 모범 사례

(능률을 위해) 클러스터는 (가능한) 쓰지말고, IDE로만 개발/테스트 하자

---
### 로컬 이미지 빌드 후 minikube 로 이미지 복사
```bash
docker save <image> | (eval $(minikube docker-env) && docker load)
```
- 주의: imagePullPolicy 가 Always 이면 로컬 변경사항을 잃어버릴 수 있다.

---
### Minikube 와 쿠버네티스 클러스터 결합
- 애플리케이션을 Minikube로 개발할 때에 제약이 (거의) 없다.
- 쿠버네티스 클러스터에 Minikube 클러스터를 연결할 수 있다.
- 개발이 완료되면, 수정없이 로컬에서 원격으로 이동할 수 있다.

---
### 5.3 버전 관리 및 리소스 매니페스트 자동 배포
쿠버네티스는 선언적 모델 사용
- 원하는 상태를 말하면 조정을 취한다.
- 메니페스트를 버전관리시스템에 저장하여, 코드 리뷰 및 로그 추적으로 필요하면 롤백할 수 있도록 구성 가능
- 새로운 커밋시 `kubectl apply` 로 변경사항 반영
- [kube-applier](https://blog.box.com/introducing-kube-applier-declarative-configuration-for-kubernetes) 으로 자동화 배포 가능
---
### 5.4 Ksonnet

.left-column[
![drawing](https://habrastorage.org/webt/oh/iw/cz/ohiwczwzpduep0hfxrghx0xtook.jpeg )
]
.right-column[
YAML sucks?
- No. 

쿠버네티스 메니페스트 개체는 JSON 형식
- kubectl explain 으로 사용 가능한 옵션 확인 가능

훨씬 적은 코드로 구성할 수 있다면?
- Ksonnet 

![drawing](17.10 The kubia Deployment written with Ksonnet kubia.ksonnet.png  "ksonnet")
]

.footnote[
.red[*] http://leebriggs.co.uk/blog/2019/02/07/why-are-we-templating-yaml.html,  https://habr.com/en/post/437682/
]

---
### 5.5 CI/CD

Fabric8 프로젝트?
젠킨스에도 CI/CD 파이프라인을 제공하는 다양한 도구가 포함되어 있음


