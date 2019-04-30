* * *   
**발표자 : [강인호]**
* * *   
   
## **1. History of Container Technology**  
ref :  http://docker-saigon.github.io/post/Docker-Internals/
![](/chapter_1_2/img/image2_1.png) 



## **2. What is namespace and cgroup**
    ▪ 모두가 아는 바와 같이 chroot on steroid 에서 다른 것 처럼 linux 입장에서는 container라는 것은 존재하지 않는다. 단지 Namespace(What we can see, isolation) 와 cgroup(resource limitation)으로 프로세스가 돌아갈 수 있는 격리된 환경을 제공하는 것 뿐이다.
![](/chapter_1_2/img/image2_2.png) 
![](/chapter_1_2/img/image2_3.png) 

### **▪ Namespace 별 실험**
        
    □  https://bluese05.tistory.com/11 원본 : https://blog.yadutaf.fr/2013/12/22/introduction-to-linux-namespaces-part-1-uts/
    ▪ UTS 예시
 ![](/chapter_1_2/img/image2_4.png)        
## **3. Container를 간단히 만드는 다양한 예시들이 있다**
  ▪ [Containers from Scratch  : Liz Rice](https://www.youtube.com/watch?v=8fi7uSYlOdc)

![](/chapter_1_2/img/image2_5.png) 

    ▪ Docker implemented in around 100 lines of bash : https://github.com/p8952/bocker
![](/chapter_1_2/img/image2_6.png) 

## **4. Cgroup**
    ▪ Limits how much you can use

![](/chapter_1_2/img/image2_7.png) 
    ▪ 출처 : https://medium.com/@nagarwal/understanding-the-docker-internals-7ccb052ce9fe
    

![](/chapter_1_2/img/image2_8.png) 
    ▪ 출처 :   https://blog.selectel.com/containerization-mechanisms-cgroups/
    ▪ 좀더 자세한 사항은 : Jerome Petazzoni : https://www.slideshare.net/jpetazzo/anatomy-of-a-container-namespaces-cgroups-some-filesystem-magic-linuxcon


## **5. Docker** 
    ▪ 그중 이런 컨테이너 기술 + 이미지관리 등의 플랫폼으로 도커
![](/chapter_1_2/img/image2_9.png)
    
## **6. 도커 구조**
    ▪ Docker Architecture - Engine, Containerd, runc : http://www.studytrails.com/devops/docker-architecture-engine-containerd-runc/
        ○ Docker Achitecture
![](/chapter_1_2/img/image2_10.png)

        ○ Docker Engine
![](/chapter_1_2/img/image2_11.png)
            
            ○ Docker Server
            ○ Snapshotter
                □ 이전 이미지에서 변경분을 layer를 통해서 저장하도록 한다. 이전에는 graphdriver로 snapshot을 떴는데, containerd에서는 snapshotter를 이용한다. 
                □  
                □ The storage drivers supported are overlay2, aufs(used by older versions), devicemapper (older CentOS and RHEL kernel versions), btrfs and zfs if the hosts use them and vfs for testing.
                □ 
 ![](/chapter_1_2/img/image2_12.png)               
            
        ○ Docker API 
![](/chapter_1_2/img/image2_13.png)
    
# **Docker 제대로 알기**
## **1. Docker Image 구조**
https://rampart81.github.io/post/docker_image/
    ▪ Boot layer 다음에는 rootfs 라고 하는 root 파일 시스템 layer이다. Root 파일 시스템은 실제 OS 가 설치된다(예를 들어 Debian 이나 Ubuntu). 
    ▪ 원래 리눅스에서는 root filesystem은 처음 mount될때는 read-only로 mount가 된후 integrity check후 read-write으로 바뀐다. 하지만 Docker에서는 계속해서 read-only 모드이다. Read-write 모드로 변환하지 않는 이유는 Docker는 union mount를 사용해서 read-only 파일 시스템들을 root 파일 시스템 위에 덮는 구조로 이루어져 있기 때문이다. 참고로 union mount는 여러게의 파일 시스템을 mount하되 하나의 파일 시스템만이 mount된것 처럼 사용하는 방법이다.
    
    ▪ 이러한 pattern을 copy on write이라고 한다. 이 copy on write 구조는 Dockerfile 을 이용해 docker image 를 빌드할때 효과적이다. Docker image를 빌드할때 Dockerfile의 각각의 instruction이 바로 filesystem / image가 되는것이다. 그러므로 빌드를 할때 처음부터 다 빌드하는 것이 아니라 바뀐 파일시스템만 mount하면 됨으로 효과적이고 빠르게 빌드 할 수 있으며 빌드가 도중에 실패하더라도 마지막으로 성공한 instruction은 빌드가 되어있는 상태이므로 그 이미지에 shell 접속을 해서 디버깅을 한다거나 하는 것이 가능해진다.
    
![](/chapter_1_2/img/image2_14.png)
![](/chapter_1_2/img/image2_15.png)
![](/chapter_1_2/img/image2_16.png)


    ▪ 참조 : Docker Doc : https://docs.docker.com/storage/storagedriver/
## **2. 2015년 도커가 원칙을 가지고 플랫폼을 분리했다**
▪ runC : https://blog.docker.com/2015/06/runc/
    ○ 2015년 6월 Docker platform에서 분리했다. 분리하면서 멋진 몇가지 원칙을 내놓았는데 이름하여 "infrastructure plumbing manifesto"
        ○ 3가지 원칙
            1. Whenever possible, re-use existing plumbing and contribute improvement back
            2. Follow the unix principles : several simple components are better than a single, complicated one
            3. Define standard interfaces which can be used to combine many simple components into a more sophisticated system
        ○ 그 첫번째가 runC이다.
            □ 
![](/chapter_1_2/img/image2_17.png)
        
        
▪ 참고 자료 : 
    ○ Julia Even : what even is a container : namepsaces and cgroup : https://jvns.ca/blog/2016/10/10/what-even-is-a-container/
    ○ 한글로 여러 영문 글을 짬뽕해 놓아 한 눈에 보기 좋은 블로그 : https://tech.ssut.me/what-even-is-a-container/
    ○ 안승규 컨테이너 자료 :https://ahnseungkyu.com/m/245?category=564278
    ○ Container Internal : 자료 모아놓은 싸이트 : https://medium.com/@nagarwal/understanding-the-docker-internals-7ccb052ce9fe
▪ 약간의 잡지식
    ○ ENTRYPOINT vs CMD 차이
        ○ ENTRYPOINT와 CMD를 함께 쓰면 parameter를 overriding 한다. 


# **Container Runtime 제대로 알기** 
	○ Docker vs Containerd vs runC
## **Container Runtime**
	○ Container Runtime Part 1 : An Introduction to Container Runtimes : https://www.ianlewis.org/en/container-runtimes-part-1-introduction-container-r
![](/chapter_1_2/img/image2_18.png)

	○ Container Runtime Part 2 : Anatomy of a Low-Level Container Runtime : https://www.ianlewis.org/en/container-runtimes-part-2-anatomy-low-level-contai
		○ $docker export $(docker create busybox) | tar -xf - -C roots  : 요렇게 하면 busybox의 내용을 rootfs에 extract하고 
![](/chapter_1_2/img/image2_19.png)
		○ Sample하게 container를 구현하는 방법 - 시연용으로 좋겠다.
![](/chapter_1_2/img/image2_20.png)

	○ What's the difference between runc, containerd, docker ? : https://medium.com/@alenkacz/whats-the-difference-between-runc-containerd-docker-3fc8f79d4d6e
		§ Docker 가 2016년 쯤에 하나로 되어 있던 모듈을 나누기 시작했다.
			□ Containerd: Daemon으로 API Facade역할을 한다. Containerd 로 인해서 더 이상 syscall을 할 필요가 없어졌다. Snapshot 같은 걸 containerd가 수행(?)
			□ Shim 은 runc를 이용해서 container를 실행해주는 역할을 한다? 
		§ Container shim의 역할 : https://groups.google.com/forum/#!topic/docker-dev/zaZFlvIx1_k
![](/chapter_1_2/img/image2_21.png)

		
## **Containerd**
![](/chapter_1_2/img/image2_22.png)
