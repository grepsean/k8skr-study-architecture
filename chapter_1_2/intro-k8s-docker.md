

* * *   
**발표자 : [강인호]**
* * *   
   
# **Chapter 1 쿠버네티스 소개**   

## **1.1 쿠버네티스 시스템이 필요한 이유**


![](/chapter_1_2/img/image1.png)   


## **1.2 컨테이너 기술 소개**
![](/chapter_1_2/img/image2.png)   
![](/chapter_1_2/img/image3.png)   
## **1.2.2 도커 컨테이너 플랫폼 소개**
![](/chapter_1_2/img/image4.png)   

## **1.3 쿠버네티스 소개**
![](/chapter_1_2/img/image5.png)   


### **1.3.3 쿠버네티스 클러스터 아키텍처**
![](/chapter_1_2/img/image6.png)   

### **1.3.4 쿠버네티스에서 애플리케이션 실행**
![](/chapter_1_2/img/image7.png)   


### **1.3.5 쿠버네티스의 장점**
	▪ 애플리케이션 배포 단순화
		○ Ex: 특정 애플리케이션이 SSD이 있는 노드에서 구동되어야 하는경우 (ex: kubectl label node node-name ssd=true)
	▪ 하드웨어 활용도 높이기
		○ 가용 리소스에 최대한 맞춰 mix and match
	▪ 상태 확인 및 자가 치유
		○ 노드 장애시 다른 노드로 재조정
	▪ 오토스케일링
		○ CCM (Cloud Controller manager)를 이용해 Worker Node 증가
	▪ 애플리케이션 개발 단순화
		○ 쿠버네티스가 제공하는 인프라를 이용하기 때문에 개발자가 "리더 선출"같은 메커니즘을 구현할 필요가 없다. 

# **Chapter 2 도커와 쿠버네티스의 첫걸음**
## **2.1.1 도커 설치 와 Hello World 컨테이너 실행**
![](/chapter_1_2/img/image8.png)
![](/chapter_1_2/img/image9.png)

## **2.1.2 간단한 Node.js 앱 만들기**
![](/chapter_1_2/img/image10.png)
![](/chapter_1_2/img/image11.png)
![](/chapter_1_2/img/image12.png)
![](/chapter_1_2/img/image13.png)

## **이미지 레이어의 이해**
![](/chapter_1_2/img/image14.png)

## **2.1.5 컨테이너 이미지 실행**
![](/chapter_1_2/img/image15.png)

~~~bash   
    § -d : detach : Run container in background and print container ID
~~~
○ 컨테이너 관련 추가 정보 
~~~bash
    § $ docker ps
    § $ docker inspect <container-name>
~~~

## **2.1.6 실행 중인 컨테이너 내부 탐색**
	$ docker exec -it <container-name> bash
    ○ 내부에서 보기 vs 외부에서 보기
        1. PID Namespace
            1. Linux는 기본적으로 PID=1 의 자식 프로세서로 구성된다.
            2. PID 네임스페이스를 새로 따게 되면 새로운 네임스페이스의 최상위가 마치 PID 1이 하는 것 처럼 끈 떨어진 넘을 child reaping 한다. 
            3. Process tree (pstree, pstree -p)

![](/chapter_1_2/img/image16.png)
그림출처 : https://bluese05.tistory.com/18

## **2.1.7 컨테이너 중지 및 제거**
    $ docker stop <container-name>
    $ docker rm <container-name>
## **2.1.8 이미지 레지스트리로 이미지 푸시**
    $ docker push 

## **2.2 쿠버네티스 클러스터 설정**
	▪  minikube vs cloud vendor 서비스 
	▪ 클러스터 노드 목록
    ~~~bash 
	$ kubectl get node
    ~~~
	▪ 객체의 세부 사항
	~~~bash 
    $ kubectl describe node <name>
    ~~~

## **2.3 쿠버네티스 첫 앱 실행**
~~~bash 
$kubectl run kubia --image=luksa/kubia --port=8080 --generator=run/v1
~~~
 ▪ Pod
![](/chapter_1_2/img/image17.png)

~~~bash 
$kubectl get pods
~~~

![](/chapter_1_2/img/image18.png)


## **2.3.2 웹 애플리케이션 액세스**
~~~bash      
$kubectl expose rc kubia --type=LoadBalancer --name kubia-http
$kubectl get services
$kubectl get replicationcontrollers
$kubectl scale rc kubia --replicas=43
~~~