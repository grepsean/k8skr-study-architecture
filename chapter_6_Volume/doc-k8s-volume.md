

* * *   
**발표자 : [한철희](https://chhanz.github.io)**
* * *   
   
# **Kubernetes Volume**   
* * *   
Kubernetes 에서 Volume 으로 사용 가능한 유형은 아래와 같습니다.   
> - emptyDir   
> - hostPath   
> - gitRepo   
> - Openstack Cinder   
> - cephfs   
> - iscsi   
> - rbd   
> - 그 외 Public Cloud Storage   
   
이처럼 Kubernetes 에서는 다양한 Volume 을 지원합니다.   
책에 소개된 emptyDir / hostPath / gitRepo 에 대해 예제와 함께 어떤식으로 사용이 되는지 확인 해보겠습니다.   
추가로 책에는 없는 nfs / cephfs / rbd 를 Kubernetes Volume 으로 사용 해보겠습니다.   
   
# **emptyDir**   
* * *   
emptyDir 은 Pod 과 함께 생성되고, 삭제되는 임시 Volume 입니다.   
컨테이너 단위로 관리되는 것이 아니고 Pod 단위로 관리가 되기 때문에 Pod 내 컨테이너가 Error 로 인해 삭제 혹은 재시작이 되더라도 emptyDir 은 삭제가 되지 않고 계속 사용이 가능합니다.   

## **emptyDir 예제**   
~~~bash   
# cat fortuneloop.sh
#!/bin/bash
trap "exit" SIGINT
mkdir /var/htdocs
while :
do
    echo $(date) Writing fortune to /var/htdocs/index.html
    /usr/games/fortune > /var/htdocs/index.html
    sleep 10
done
~~~   
위 스크립트를 이용해서 Docker 이미지를 Build 합니다.   
~~~Dockerfile
FROM ubuntu:latest
RUN apt-get update ; apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
ENTRYPOINT /bin/fortuneloop.sh
~~~   
Docker Image 를 Build 합니다.   
~~~bash   
# docker build -t han0495/fortune .
Sending build context to Docker daemon  3.072kB
Step 1/4 : FROM ubuntu:latest
 ---> 94e814e2efa8
Step 2/4 : RUN apt-get update ; apt-get -y install fortune
 ---> Running in 3c8f694a68af

< 중 략 >

Removing intermediate container 3c8f694a68af
 ---> 1e8b262e7bdf
Step 3/4 : ADD fortuneloop.sh /bin/fortuneloop.sh
 ---> 3eee41108b5b
Step 4/4 : ENTRYPOINT /bin/fortuneloop.sh
 ---> Running in 082ee707cdf1
Removing intermediate container 082ee707cdf1
 ---> 58d2d430a7b4
Successfully built 58d2d430a7b4
Successfully tagged han0495/fortune:latest
[root@m01 inside]#
~~~   
아래와 같은 yaml 파일을 작성합니다.   
~~~yml   
# cat fortune.yml
apiVersion: v1
kind: Pod
metadata:
  name: fortune
spec:
  containers:
  - image: han0495/fortune
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
~~~   
작성한 yaml 파일을 이용하여, Pod을 생성합니다.   
~~~sh   
[root@m01 fortune]# kubectl get po
NAME                              READY   STATUS    RESTARTS   AGE
fortune                           2/2     Running   0          5m10s
load-generator-557649ddcd-nl6js   1/1     Running   1          6d18h
php-apache-9bd5c887f-nm4h5        1/1     Running   0          6d18h
tomcat-f94554bb9-gkhpz            1/1     Running   0          7d
web-7d77974d4c-gd76n              1/1     Running   0          7d2h
 
[root@m01 fortune]# kubectl describe po fortune
Name:               fortune
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               w03/192.168.13.16
Start Time:         Mon, 08 Apr 2019 17:41:40 +0900
Labels:             <none>
Annotations:        <none>
Status:             Running
IP:                 10.233.89.8
Containers:
  html-generator:
    Container ID:   docker://e25bc8c3b94a2a02edc8b983eb77214a4644a99a3931c1f96f131819060cc676
    Image:          han0495/fortune
    Image ID:       docker-pullable://han0495/fortune@sha256:63d5786a84e67dcd5eec70d516d5788c8e3c3a90d23f23bec1825f7a4526bb00
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 08 Apr 2019 17:42:01 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/htdocs from html (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-vt6hm (ro)
  web-server:
    Container ID:   docker://ed7c2fd5adbb919fde6ed01d1a80fa74df689d1aa99bc8d883b1b68ed918dd09
    Image:          nginx:alpine
    Image ID:       docker-pullable://nginx@sha256:d5e177fed5e4f264e55b19b84bdc494078a06775612a4f60963f296756ea83aa
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 08 Apr 2019 17:42:09 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /usr/share/nginx/html from html (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-vt6hm (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  html:
    Type:    EmptyDir (a temporary directory that shares a pod``s lifetime)
    Medium:
  default-token-vt6hm:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-vt6hm
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  5m26s  default-scheduler  Successfully assigned default/fortune to w03
  Normal  Pulling    5m23s  kubelet, w03       pulling image "han0495/fortune"
  Normal  Pulled     5m5s   kubelet, w03       Successfully pulled image "han0495/fortune"
  Normal  Created    5m5s   kubelet, w03       Created container
  Normal  Started    5m4s   kubelet, w03       Started container
  Normal  Pulling    5m4s   kubelet, w03       pulling image "nginx:alpine"
  Normal  Pulled     4m56s  kubelet, w03       Successfully pulled image "nginx:alpine"
  Normal  Created    4m56s  kubelet, w03       Created container
  Normal  Started    4m56s  kubelet, w03       Started container
[root@m01 fortune]#
~~~   
실제로 파일이 Docker 이미지에 넣은 스크립트가 동작하는지 확인합니다.   
~~~sh   
[root@m01 fortune]# kubectl exec -ti fortune /bin/bash
Defaulting container name to html-generator.
Use 'kubectl describe pod/fortune -n default' to see all of the containers in this pod.
root@fortune:/#
root@fortune:/# cd /var
root@fortune:/var# cd htdocs/
root@fortune:/var/htdocs# ls
index.html
root@fortune:/var/htdocs#
root@fortune:/var/htdocs# cat index.html
You are standing on my toes.
root@fortune:/var/htdocs# while true
> do
> date
> cat index.html
> sleep 10
> done
Mon Apr  8 08:48:43 UTC 2019
Artistic ventures highlighted.  Rob a museum.
Mon Apr  8 08:48:53 UTC 2019
Never reveal your best argument.
Mon Apr  8 08:49:03 UTC 2019
You can create your own opportunities this week.  Blackmail a senior executive.
Mon Apr  8 08:49:13 UTC 2019
You have a strong appeal for members of the opposite sex.
Mon Apr  8 08:49:23 UTC 2019
You will be reincarnated as a toad; and you will be much happier.
^C
root@fortune:/var/htdocs#
~~~   
   
# **hostPath**   
* * *   
hostPath는 로컬 디스크의 경로를 Pod 에 Mount 해서 사용하는 Volume 방식입니다.   
Docker 에서 -v 옵션으로 Volume 을 연결하는 것과 동일하다고 생각하면 됩니다.   

## **hostPath 예제**   
~~~yaml   
# cat hostpath-pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: volumepath
      mountPath: /usr/share/nginx/html
  volumes:
  - name : volumepath
    hostPath:
      path: /imsi
      type: Directory
~~~   
nginx 의 index Directory 에 /imsi 라는 로컬 디스크 경로를 Mount 합니다.   
이후 Pod 을 해당 yaml 을 이용해서 구동합니다.   
~~~sh   
[root@m01 pod-example]# kubectl exec -ti hostpath -- /bin/bash
root@hostpath:/#
root@hostpath:/# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@hostpath:/#
root@hostpath:/# cd /usr
root@hostpath:/usr# cd share/
root@hostpath:/usr/share# cd nginx/
root@hostpath:/usr/share/nginx# cd html/
root@hostpath:/usr/share/nginx/html# ls
root@hostpath:/usr/share/nginx/html#
root@hostpath:/usr/share/nginx/html# touch test
root@hostpath:/usr/share/nginx/html# exit
exit
[root@m01 pod-example]#
~~~
Pod 이 구동된 Worker의 로컬 디스크 /imsi 경로를 확인해보면 아래와 같이 파일이 생성된 것을 확인 할 수 있습니다.   
~~~sh
[root@w01 ~]# ls -la /imsi
합계 0
drwxrwxrwx   2 root root  18  4월  9 13:15 .
dr-xr-xr-x. 18 root root 256  4월  9 09:45 ..
-rw-r--r--   1 root root   0  4월  9 13:15 test
[root@w01 ~]#
~~~   
   
# **gitRepo**   
* * *   
gitRepo 는 github에 있는 Repository 에서 Source 를 Clone 하고 해당 Clone 된 데이터를 Pod 의 Volume으로 할당합니다.   
   
# **gitRepo 예제**   
~~~yaml   
# cat git-http.yml
apiVersion: v1
kind: Pod
metadata:
  name: gitrepo-httpd
spec:
  containers:
  - image: httpd
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/local/apache2/htdocs
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    gitRepo:
      repository: https://github.com/chhanz/docker_training.git
      revision: master
      directory: .
~~~   
위와 같이 Github에서 docker_training.git Source를 Clone 합니다.   
~~~sh
[root@m01 tmp]# kubectl describe po gitrepo-httpd
Name:               gitrepo-httpd
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               w02/192.168.13.15
Start Time:         Mon, 08 Apr 2019 21:51:43 +0900
Labels:             <none>
Annotations:        <none>
Status:             Pending
IP:
Containers:
  web-server:
    Container ID:
    Image:          nginx:alpine
    Image ID:
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       ContainerCreating
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /usr/share/nginx/html from html (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-vt6hm (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Volumes:
  html:
    Type:        GitRepo (a volume that is pulled from git when the pod is created)
    Repository:  https://github.com/chhanz/docker_training.git
    Revision:    master
  default-token-vt6hm:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-vt6hm
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason       Age              From               Message
  ----     ------       ----             ----               -------
  Normal   Scheduled    6s               default-scheduler  Successfully assigned default/gitrepo-httpd to w02
  Warning  FailedMount  1s (x4 over 5s)  kubelet, w02       MountVolume.SetUp failed for volume "html" : failed to exec 'git clone -- https://github.com/chhanz/docker_training.git .': : executable file not found in $PATH
~~~   
위와 같이 에러가 발생하며, Volume 이 Mount 가 안되었습니다.   
이유는 Pod 이 실행될 Worker 노드에 git 명령어가 없어서 발생된 에러였습니다.   
~~~sh
[root@m01 gitrepo-pv]# kubectl describe po gitrepo-httpd
Name:               gitrepo-httpd
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               w01/192.168.13.14
Start Time:         Mon, 08 Apr 2019 21:58:23 +0900
Labels:             <none>
Annotations:        <none>
Status:             Running
IP:                 10.233.118.8
Containers:
  web-server:
    Container ID:   docker://38158075431bab4e7cfe22e34b615e66ac04e37ad7832a531d811cd67b27962a
    Image:          httpd
    Image ID:       docker-pullable://httpd@sha256:b4096b744d92d1825a36b3ace61ef4caa2ba57d0307b985cace4621139c285f7
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 08 Apr 2019 21:58:49 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /usr/local/apache2/htdocs from html (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-vt6hm (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  html:
    Type:        GitRepo (a volume that is pulled from git when the pod is created)
    Repository:  https://github.com/chhanz/docker_training.git
    Revision:    master
  default-token-vt6hm:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-vt6hm
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  28s   default-scheduler  Successfully assigned default/gitrepo-httpd to w01
  Normal  Pulling    21s   kubelet, w01       pulling image "httpd"
  Normal  Pulled     0s    kubelet, w01       Successfully pulled image "httpd"
  Normal  Created    0s    kubelet, w01       Created container
  Normal  Started    0s    kubelet, w01       Started container
[root@m01 gitrepo-pv]# kubectl get po
NAME                              READY   STATUS    RESTARTS   AGE
fortune                           2/2     Running   0          4h17m
gitrepo-httpd                     1/1     Running   0          33s
load-generator-557649ddcd-nl6js   1/1     Running   1          6d22h
php-apache-9bd5c887f-nm4h5        1/1     Running   0          6d22h
tomcat-f94554bb9-gkhpz            1/1     Running   0          7d4h
web-7d77974d4c-gd76n              1/1     Running   0          7d6h
[root@m01 gitrepo-pv]#
~~~
위와 같이 Worker 노드에 git 명령을 설치한 이후, 정상적으로 Volume 이 Mount 되는 것을 확인 할 수 있었습니다.   
~~~sh
[root@w01 /]# find * | grep index.html
 
var/lib/kubelet/pods/fcd69917-59fd-11e9-a751-001a4a160172/volumes/kubernetes.io~git-repo/html/index.html
 
[root@w01 /]# cd var/lib/kubelet/pods/fcd69917-59fd-11e9-a751-001a4a160172/volumes/kubernetes.io~git-repo/html/
[root@w01 html]# ls
copy.html  index.html  README.md  util
[root@w01 html]# ls -la
total 20
drwxrwxrwx 4 root root   82 Apr  8 21:58 .
drwxr-xr-x 3 root root   18 Apr  8 21:58 ..
-rw-r--r-- 1 root root 2070 Apr  8 21:58 copy.html
drwxr-xr-x 8 root root  180 Apr  8 21:58 .git
-rw-r--r-- 1 root root 9594 Apr  8 21:58 index.html
-rw-r--r-- 1 root root  108 Apr  8 21:58 README.md
drwxr-xr-x 2 root root   23 Apr  8 21:58 util
[root@w01 html]#
[root@w01 html]#
[root@w01 html]# cat README.md
# Docker Training ReadMe
Custumer training page
 
 - index.html
 - copy.html     // Copy&Paste
 - putty.exe
~~~   
위와 같이 Worker 노드에 해당 git Source 를 Clone 하고 Pod 에 Mount 된 것을 확인 할 수 있었습니다.
~~~sh
## Master Node
[root@m01 gitrepo-pv]# kubectl delete -f git-http.yml
pod "gitrepo-httpd" deleted
 
## Worker Node
[root@w01 kubernetes.io~git-repo]# cd ..
cd: error retrieving current directory: getcwd: cannot access parent directories: No such file or directory
[root@w01 ..]# ls
[root@w01 ..]#
[root@w01 ..]# ls -la
total 0
[root@w01 ..]# pwd
/var/lib/kubelet/pods/fcd69917-59fd-11e9-a751-001a4a160172/volumes/kubernetes.io~git-repo/..
[root@w01 ..]#
[root@w01 ..]#
[root@w01 ..]# cd ..
cd: error retrieving current directory: getcwd: cannot access parent directories: No such file or directory
[root@w01 ..]# ls
[root@w01 ..]#
[root@w01 ..]# ls -la /var/lib/kubelet/pods/fcd69917-59fd-11e9-a751-001a4a160172/
ls: cannot access /var/lib/kubelet/pods/fcd69917-59fd-11e9-a751-001a4a160172/: No such file or directory
[root@w01 ..]#
~~~
이처럼 해당 Pod이 삭제되면 Clone 된 Git Source는 삭제가 되는 것을 확인 하였습니다.   
위 테스트를 진행해보니 Git Source 를 Clone 만하고 Github Repository를 계속 Sync하는 것은 아닌 것으로 확인 하였습니다.   
   
지금까지 emptyDir / hostPath / gitRepo 에 대해 예제를 통해 확인 하였습니다.   
nfs / cephfs / rbd 은 다음 포스팅에서 이어서 설명하도록 하겠습니다.   
   
# **Kubernetes Volume #2**   
* * *   
이어서 Network Volume 으로 사용될 nfs / cephfs / ceph rbd 를 예제와 함께 알아보도록 하겠습니다.   
   
# **Persistent Volume 와 Persistent Volume Claim**   
* * *   
Persistent Volume 와 Persistent VolumeClaim 가 있는데, 
> **Persistent Volume**(이하 PV) 는 Kubernetes 에서 관리되는 저장소로 Pod 과는 다른 수명 주기로 관리됩니다.   
Pod 이 재실행 되더라도, PV의 데이터는 정책에 따라 **유지/삭제**가 됩니다.   
   
> **Persistent Volume Claim**(이하 PVC) 는 PV를 추상화하여 개발자가 손쉽게 PV를 사용 가능하게 만들어주는 기능입니다.   
개발자는 사용에 필요한 Volume의 크기, Volume의 정책을 선택하고 요청만 하면 됩니다.   
운영자는 개발자의 요청에 맞게 PV 를 생성하게 되고, PVC는 해당 PV를 가져가게 됩니다.   
   
![](/chapter_6_Volume/img/image1.png)   
   
이와 같은 방식을 **Static Provisioning** 이라 합니다.   
예제를 통해 Static Provisioning을 확인 해보겠습니다.   

# **Static Provisioning**
* * *
## **NFS**
NFS 서버를 PV로 사용하는 방식입니다.   
예제에 활용될 yaml 파일 내용은 아래와 같습니다.   
~~~yaml   
[root@m01 pod-example]# cat nfs-pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-nginx
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: nfsvol
      mountPath: /usr/share/nginx/html
  volumes:
  - name : nfsvol
    nfs:
      path: /data/nfs-ngnix
      server: 192.168.13.10
~~~   
위 yaml 파일을 이용해 Pod 을 생성하면   
~~~sh   
[root@m01 pod-example]# kubectl describe po nfs-nginx
Name:               nfs-nginx
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               w03/192.168.13.16
Start Time:         Sun, 14 Apr 2019 13:44:52 +0900
Labels:             <none>
Annotations:        <none>
Status:             Running
IP:                 10.233.89.5
Containers:
  nginx:
    Container ID:   docker://20fa842803535803e1c0c48c204cffe1d464f9f96e3fcf4d7eed11c0bb8aeed0
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:50174b19828157e94f8273e3991026dc7854ec7dd2bbb33e7d3bd91f0a4b333d
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sun, 14 Apr 2019 13:45:13 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /usr/share/nginx/html from nfsvol (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-9vmtn (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  nfsvol:
    Type:      NFS (an NFS mount that lasts the lifetime of a pod)
    Server:    192.168.13.10
    Path:      /data/nfs-ngnix
    ReadOnly:  false
  default-token-9vmtn:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-9vmtn
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  50s   default-scheduler  Successfully assigned default/nfs-nginx to w03
  Normal  Pulling    46s   kubelet, w03       pulling image "nginx"
  Normal  Pulled     28s   kubelet, w03       Successfully pulled image "nginx"
  Normal  Created    28s   kubelet, w03       Created container
  Normal  Started    28s   kubelet, w03       Started container
~~~   
nfs-nginx 라는 Pod 이 생성이 되고 위와 같이 **nfsvol** 이라는 Volume 이 Attach 된 것을 확인 할 수 있습니다.   
nginx 서비스가 연결된 Volume 을 통해 서비스가 되는지 확인해봅니다.  
~~~sh
[root@m01 pod-example]# curl 10.233.89.5
<html>
<body>
<p>NFS Index-v1</p>
</body>
</html>
[root@m01 pod-example]#

# Pod 내부에 접근해서 확인
[root@m01 pod-example]# kubectl exec -ti nfs-nginx /bin/bash
root@nfs-nginx:/#
root@nfs-nginx:/# cd /usr/share/nginx/html/
root@nfs-nginx:/usr/share/nginx/html# cat index.html
<html>
<body>
<p>NFS Index-v1</p>
</body>
</html>
root@nfs-nginx:/usr/share/nginx/html#
~~~   
**NFS Index-v1** 라는 index.html 을 가지고 있는 Volume 입니다.   
NFS 서버에 직접 접근해서 index.html 파일을 수정해보겠습니다.   
~~~sh   
[root@kube-depoly nfs-ngnix]# pwd
/data/nfs-ngnix
[root@kube-depoly nfs-ngnix]# cat index.html
<html>
<body>
<p>NFS Index-v1</p>
</body>
</html>
[root@kube-depoly nfs-ngnix]# vi index.html        // index.html 수정
[root@kube-depoly nfs-ngnix]# cat index.html
<html>
<body>
<p>NFS Index-v2</p>
</body>
</html>
[root@kube-depoly nfs-ngnix]#

# 적용 확인
[root@m01 pod-example]# curl 10.233.89.5
<html>
<body>
<p>NFS Index-v2</p>
</body>
</html>
[root@m01 pod-example]#
~~~   
이와 같이 Pod 에 NFS 서버가 연결 되어 있는 것을 확인 할 수 있었습니다.   
~~~sh
# NFS 로 Volume Attach 되어 있음
root@nfs-nginx:/usr/share/nginx/html# mount | grep nfs
192.168.13.10:/data/nfs-ngnix on /usr/share/nginx/html type nfs4 (rw,relatime,vers=4.1,rsize=262144,wsize=262144,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.13.15,local_lock=none,addr=192.168.13.10)
root@nfs-nginx:/usr/share/nginx/html#
~~~   
   
## **cephfs**
Software Defined Storage 인 ceph 를 이용하는 방식입니다.   
> Worker 노드에서 cephfs 를 사용하기 위해 ceph-common 패키지를 설치합니다.   
~~~sh   
# yum -y install epel-release
# rpm -Uvh https://download.ceph.com/rpm-luminous/el7/noarch/ceph-release-1-0.el7.noarch.rpm
# yum -y install ceph-common
~~~   

cephfs 를 사용하기 위해서는 Key 가 필요로 한데, 아래와 같은 방식으로 Key 값을 수집하고 Kubernete Secret 에 등록합니다.   

~~~sh
[root@s01 yum.repos.d]# ceph auth get client.admin
exported keyring for client.admin
[client.admin]
    key = AQC6s6Vc83jwKBAAtckE6yz3eTM9lWwK60QNYw==
    caps mds = "allow *"
    caps mgr = "allow *"
    caps mon = "allow *"
    caps osd = "allow *"
[root@s01 yum.repos.d]#
 
[root@s01 ceph]# ceph-authtool -p ceph.client.admin.keyring
AQC6s6Vc83jwKBAAtckE6yz3eTM9lWwK60QNYw==
[root@s01 ceph]#

# Secret 생성
[root@m01 cephfs]# cat ceph-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
data:
  key: QVFDNnM2VmM4M2p3S0JBQXRja0U2eXozZVRNOWxXd0s2MFFOWXc9PQ==

[root@m01 cephfs]# kubectl create -f ceph-secret.yaml
secret/ceph-secret created

[root@m01 cephfs]# kubectl get secret ceph-secret
NAME          TYPE     DATA   AGE
ceph-secret   Opaque   1      23s
[root@m01 cephfs]# kubectl get secret ceph-secret -o yaml
apiVersion: v1
data:
  key: QVFDNnM2VmM4M2p3S0JBQXRja0U2eXozZVRNOWxXd0s2MFFOWXc9PQ==
kind: Secret
metadata:
  creationTimestamp: "2019-04-14T06:13:44Z"
  name: ceph-secret
  namespace: default
  resourceVersion: "873772"
  selfLink: /api/v1/namespaces/default/secrets/ceph-secret
  uid: 7515ad43-5e7c-11e9-ba95-001a4a160172
type: Opaque
[root@m01 cephfs]#
~~~   
Pod 에 cephfs Volume 을 연결합니다.   
~~~yaml   
[root@m01 cephfs]# cat cephfs-with-secret.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cephfs-httpd
spec:
  containers:
  - name: cephfs-httpd
    image: httpd
    volumeMounts:
    - mountPath: /usr/local/apache2/htdocs
      name: cephfs
  volumes:
  - name: cephfs
    cephfs:
      monitors:
      - 192.168.13.6:6789
      - 192.168.13.7:6789
      - 192.168.13.8:6789
      user: admin
      path: /httpd-index
      secretRef:
        name: ceph-secret
      readOnly: false
~~~   
cephfs 의 /httpd-index 경로에는 index.html 이 존재합니다.   
Pod 을 생성합니다.   
~~~sh   
[root@m01 cephfs]# kubectl create -f cephfs-with-secret.yaml
pod/cephfs-httpd created
[root@m01 cephfs]# kubectl get po
NAME                              READY   STATUS    RESTARTS   AGE
cephfs-httpd                      1/1     Running   0          24s
load-generator-557649ddcd-jq987   1/1     Running   1          4d20h
php-apache-9bd5c887f-p6lrq        1/1     Running   0          4d20h
[root@m01 cephfs]#
 
[root@m01 cephfs]# kubectl describe po cephfs-httpd
Name:               cephfs-httpd
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               w03/192.168.13.16
Start Time:         Sun, 14 Apr 2019 15:16:48 +0900
Labels:             <none>
Annotations:        <none>
Status:             Running
IP:                 10.233.89.6
Containers:
  cephfs-httpd:
    Container ID:   docker://71e17fb3708a68448fdded4a20c81af63716a1146156dc5a5b4b8145a290f3dc
    Image:          httpd
    Image ID:       docker-pullable://httpd@sha256:b4096b744d92d1825a36b3ace61ef4caa2ba57d0307b985cace4621139c285f7
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sun, 14 Apr 2019 15:17:04 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /usr/local/apache2/htdocs from cephfs (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-9vmtn (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  cephfs:
    Type:        CephFS (a CephFS mount on the host that shares a pods lifetime)
    Monitors:    [192.168.13.6:6789 192.168.13.7:6789 192.168.13.8:6789]
    Path:        /httpd-index
    User:        admin
    SecretFile:
    SecretRef:   &LocalObjectReference{Name:ceph-secret,}
    ReadOnly:    false
  default-token-9vmtn:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-9vmtn
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  36s   default-scheduler  Successfully assigned default/cephfs-httpd to w03
  Normal  Pulling    33s   kubelet, w03       pulling image "httpd"
  Normal  Pulled     21s   kubelet, w03       Successfully pulled image "httpd"
  Normal  Created    20s   kubelet, w03       Created container
  Normal  Started    20s   kubelet, w03       Started container
[root@m01 cephfs]#

# Pod 테스트
[root@m01 cephfs]# curl 10.233.89.6
cephfs Index - v1
[root@m01 cephfs]#
~~~
cephfs는 NFS 와 매우 유사합니다. 그로 인해 NFS와 동일하게 PV에 직접 접근해서 파일 내용을 수정하고, Volume Attach 내용을 확인 할 수 있습니다.   
~~~sh   
# Mount 확인 - Pod 이 작동중인 Worker 노드
[root@w03 ~]# mount | grep ceph
192.168.13.6:6789,192.168.13.7:6789,192.168.13.8:6789:/httpd-index on /var/lib/kubelet/pods/e2b5c688-5e7c-11e9-ba95-001a4a160172/volumes/kubernetes.io~cephfs/cephfs type ceph (rw,relatime,name=admin,secret=<hidden>,acl,wsize=16777216)
[root@w03 ~]#
[root@w03 ~]# cd /var/lib/kubelet/pods/e2b5c688-5e7c-11e9-ba95-001a4a160172/volumes/kubernetes.io~cephfs/cephfs
[root@w03 cephfs]# ls -la
합계 1
drwxr-xr-x 1 root root  1  4월 14 15:05 .
drwxr-x--- 3 root root 20  4월 14 15:16 ..
-rw-r--r-- 1 root root 18  4월 14 15:05 index.html
[root@w03 cephfs]# cat index.html
cephfs Index - v1
[root@w03 cephfs]#

# Cephfs 에 접근해서 직접 파일 수정
[root@kube-depoly httpd-index]# pwd
/cephfs/httpd-index
[root@kube-depoly httpd-index]# vi index.html
[root@kube-depoly httpd-index]# cat index.html
cephfs Index - v2
[root@kube-depoly httpd-index]#
 
[root@m01 cephfs]# curl 10.233.89.6
cephfs Index - v2
[root@m01 cephfs]#
~~~   
   
   지금까지 Static Provisioning 관련 해서 확인 해보았습니다.   
      
개발자가 PVC를 통해 시스템 관리자에게 PV를 요구하는 과정을 통해 PV를 할당 받고 사용이 가능한데 이 과정을 자동화를 하게 되면 Dynamic Provisioning 이라고 합니다.   

# **Dynamic Provisioning**
* * *
## **ceph rbd**
Dynamic Provisioning는 PVC를 통해 요청하는 PV대해 동적으로 생성을 해주는 제공 방식을 말합니다.   
개발자는 StorageClass 를 통해 필요한 Storage Type을 지정하여 동적으로 할당을 받을 수 있습니다.   
~~~sh   
# Secret 생성 - ceph Login을 위한 Key
[root@m01 ceph-rbd]# kubectl create -f ceph-admin-secret.yml
secret/ceph-admin-secret created
[root@m01 ceph-rbd]# kubectl create -f ceph-secret.yml
secret/ceph-secret created
~~~   
~~~yaml   
# StorageClass 생성
[root@m01 ceph-rbd]# cat class.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: rbd
provisioner: ceph.com/rbd
parameters:
  monitors: 192.168.13.6:6789,192.168.13.7:6789,192.168.13.8:6789
  pool: kube
  adminId: admin
  adminSecretNamespace: kube-system
  adminSecretName: ceph-admin-secret
  userId: kube
  userSecretNamespace: kube-system
  userSecretName: ceph-secret
  imageFormat: "2"
  imageFeatures: layering
[root@m01 ceph-rbd]# kubectl get sc
NAME   PROVISIONER    AGE
rbd    ceph.com/rbd   17h
[root@m01 ceph-rbd]#
~~~   
StorageClass yaml 파일을 보면 **provisioner: ceph.com/rbd** 항목이 있습니다.   
> 위와 같이 ceph rbd 를 제공해줄 provisioner 가 필요합니다.   
> provisioner 상세 배포 방식은 [ceph-rbd-depoly](https://github.com/kubernetes-incubator/external-storage/blob/master/ceph/rbd/deploy/README.md)
문서를 참조합니다.   
   

~~~sh
# Pod 
[root@m01 ceph-rbd]# kubectl get po -n kube-system | grep rbd
rbd-provisioner-67b4857bcd-7ctlz           1/1     Running   0          17h

# Pod 상세 내역
[root@m01 ceph-rbd]# kubectl describe po rbd-provisioner-67b4857bcd-7ctlz -n kube-system
Name:               rbd-provisioner-67b4857bcd-7ctlz
Namespace:          kube-system
Priority:           0
PriorityClassName:  <none>
Node:               w02/192.168.13.15
Start Time:         Sun, 14 Apr 2019 21:13:49 +0900
Labels:             app=rbd-provisioner
                    pod-template-hash=67b4857bcd
Annotations:        <none>
Status:             Running
IP:                 10.233.96.9
Controlled By:      ReplicaSet/rbd-provisioner-67b4857bcd
Containers:
  rbd-provisioner:
    Container ID:   docker://8f25dca0c870685dc0140294787124e288793243ed6120921d278c701b6c7039
    Image:          quay.io/external_storage/rbd-provisioner:latest
    Image ID:       docker-pullable://quay.io/external_storage/rbd-provisioner@sha256:94fd36b8625141b62ff1addfa914d45f7b39619e55891bad0294263ecd2ce09a
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sun, 14 Apr 2019 21:13:54 +0900
    Ready:          True
    Restart Count:  0
    Environment:
      PROVISIONER_NAME:  ceph.com/rbd
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from rbd-provisioner-token-79f4c (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  rbd-provisioner-token-79f4c:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  rbd-provisioner-token-79f4c
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:          <none>
~~~   
PVC 를 통해 rbd PV를 요청합니다.  
~~~yaml   
[root@m01 ceph-rbd]# cat claim.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: rbd-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: rbd
  resources:
    requests:
      storage: 2Gi 
~~~   
~~~sh   
# PVC 생성
[root@m01 ceph-rbd]# kubectl create -f claim.yaml
persistentvolumeclaim/rbd-pvc created

# PVC 확인
[root@m01 ceph-rbd]# kubectl get pvc
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
claim1          Bound    pvc-fe9d8199-5eae-11e9-ba95-001a4a160172   1Gi        RWO            rbd            17h
nginx-vol-pvc   Bound    nginx-pv                                   3Gi        RWX                           17h
rbd-pvc         Bound    pvc-5b571f95-5f43-11e9-ba95-001a4a160172   2Gi        RWO            rbd            3s

# PV 연결 확인
[root@m01 ceph-rbd]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS   REASON   AGE
nginx-pv                                   3Gi        RWX            Retain           Bound    default/nginx-vol-pvc                           17h
pvc-5b571f95-5f43-11e9-ba95-001a4a160172   2Gi        RWO            Delete           Bound    default/rbd-pvc         rbd                     5s
pvc-fe9d8199-5eae-11e9-ba95-001a4a160172   1Gi        RWO            Delete           Bound    default/claim1          rbd                     17h
[root@m01 ceph-rbd]#
~~~   
위와 같이 PVC 를 요청하자 바로 PV가 ceph rbd 형식으로 생성이 되고 Attach 된 것을 확인 할 수 있었습니다.   
~~~sh   
[root@m01 ceph-rbd]# kubectl logs -f rbd-provisioner-67b4857bcd-7ctlz -n kube-system
I0414 12:13:54.944458       1 main.go:85] Creating RBD provisioner ceph.com/rbd with identity: ceph.com/rbd
I0414 12:13:54.949989       1 leaderelection.go:185] attempting to acquire leader lease  kube-system/ceph.com-rbd...
I0414 12:13:55.001529       1 leaderelection.go:194] successfully acquired lease kube-system/ceph.com-rbd
I0414 12:13:55.001754       1 event.go:221] Event(v1.ObjectReference{Kind:"Endpoints", Namespace:"kube-system", Name:"ceph.com-rbd", UID:"c5bb0c91-5eae-11e9-b387-001a4a160174", APIVersion:"v1", ResourceVersion:"919145", FieldPath:""}): type: 'Normal' reason: 'LeaderElection' rbd-provisioner-67b4857bcd-7ctlz_c5ecdcd7-5eae-11e9-a9ca-6e9439dbce0f became leader
I0414 12:13:55.001901       1 controller.go:631] Starting provisioner controller ceph.com/rbd_rbd-provisioner-67b4857bcd-7ctlz_c5ecdcd7-5eae-11e9-a9ca-6e9439dbce0f!
I0414 12:13:55.102448       1 controller.go:680] Started provisioner controller ceph.com/rbd_rbd-provisioner-67b4857bcd-7ctlz_c5ecdcd7-5eae-11e9-a9ca-6e9439dbce0f!
I0414 12:15:33.439001       1 controller.go:987] provision "default/claim1" class "rbd": started
I0414 12:15:33.455112       1 event.go:221] Event(v1.ObjectReference{Kind:"PersistentVolumeClaim", Namespace:"default", Name:"claim1", UID:"fe9d8199-5eae-11e9-ba95-001a4a160172", APIVersion:"v1", ResourceVersion:"919428", FieldPath:""}): type: 'Normal' reason: 'Provisioning' External provisioner is provisioning volume for claim "default/claim1"
I0414 12:15:35.895596       1 provision.go:132] successfully created rbd image "kubernetes-dynamic-pvc-00ab7b91-5eaf-11e9-a9ca-6e9439dbce0f"
I0414 12:15:35.895798       1 controller.go:1087] provision "default/claim1" class "rbd": volume "pvc-fe9d8199-5eae-11e9-ba95-001a4a160172" provisioned
I0414 12:15:35.895924       1 controller.go:1101] provision "default/claim1" class "rbd": trying to save persistentvvolume "pvc-fe9d8199-5eae-11e9-ba95-001a4a160172"
I0414 12:15:35.934205       1 controller.go:1108] provision "default/claim1" class "rbd": persistentvolume "pvc-fe9d8199-5eae-11e9-ba95-001a4a160172" saved
I0414 12:15:35.934420       1 controller.go:1149] provision "default/claim1" class "rbd": succeeded
I0414 12:15:35.935026       1 event.go:221] Event(v1.ObjectReference{Kind:"PersistentVolumeClaim", Namespace:"default", Name:"claim1", UID:"fe9d8199-5eae-11e9-ba95-001a4a160172", APIVersion:"v1", ResourceVersion:"919428", FieldPath:""}): type: 'Normal' reason: 'ProvisioningSucceeded' Successfully provisioned volume pvc-fe9d8199-5eae-11e9-ba95-001a4a160172
I0415 05:57:32.016818       1 controller.go:987] provision "default/rbd-pvc" class "rbd": started
I0415 05:57:32.037901       1 event.go:221] Event(v1.ObjectReference{Kind:"PersistentVolumeClaim", Namespace:"default", Name:"rbd-pvc", UID:"5b571f95-5f43-11e9-ba95-001a4a160172", APIVersion:"v1", ResourceVersion:"1081791", FieldPath:""}): type: 'Normal' reason: 'Provisioning' External provisioner is provisioning volume for claim "default/rbd-pvc"
I0415 05:57:33.772819       1 provision.go:132] successfully created rbd image "kubernetes-dynamic-pvc-5be57bb6-5f43-11e9-a9ca-6e9439dbce0f"
I0415 05:57:33.773007       1 controller.go:1087] provision "default/rbd-pvc" class "rbd": volume "pvc-5b571f95-5f43-11e9-ba95-001a4a160172" provisioned
I0415 05:57:33.773112       1 controller.go:1101] provision "default/rbd-pvc" class "rbd": trying to save persistentvvolume "pvc-5b571f95-5f43-11e9-ba95-001a4a160172"
I0415 05:57:33.793499       1 controller.go:1108] provision "default/rbd-pvc" class "rbd": persistentvolume "pvc-5b571f95-5f43-11e9-ba95-001a4a160172" saved
I0415 05:57:33.793633       1 controller.go:1149] provision "default/rbd-pvc" class "rbd": succeeded
I0415 05:57:33.793801       1 controller.go:987] provision "default/rbd-pvc" class "rbd": started
I0415 05:57:33.794971       1 event.go:221] Event(v1.ObjectReference{Kind:"PersistentVolumeClaim", Namespace:"default", Name:"rbd-pvc", UID:"5b571f95-5f43-11e9-ba95-001a4a160172", APIVersion:"v1", ResourceVersion:"1081791", FieldPath:""}): type: 'Normal' reason: 'ProvisioningSucceeded' Successfully provisioned volume pvc-5b571f95-5f43-11e9-ba95-001a4a160172
I0415 05:57:33.826515       1 controller.go:996] provision "default/rbd-pvc" class "rbd": persistentvolume "pvc-5b571f95-5f43-11e9-ba95-001a4a160172" already exists, skipping
~~~  
- 참조: rbd-provisioner docker log   
   
Pod 에 PVC를 yaml 추가하여 Pod 생성 할때, PV를 요청하고 StorageClass 를 이용해서 동적으로 Volume을 할당 받았습니다.   
아래는 Pod 에 PVC를 추가한 yaml 구문 Template 입니다.   
~~~yaml   
[root@m01 ceph-rbd]# cat test-pod.yaml
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  containers:
  - name: test-pod
    image: gcr.io/google_containers/busybox:1.24
    command:
    - "/bin/sh"
    args:
    - "-c"
    - "touch /mnt/SUCCESS && exit 0 || exit 1"
    volumeMounts:
    - name: pvc
      mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
  - name: pvc
    persistentVolumeClaim:
      claimName: claim1
~~~   
   
지금까지 포스팅에서 소개된 Volume 은 Kubernetes 에서 제공되는 일부 입니다.   
다양한 Volume 을 제공하고 있으며, 상세 내용은 첨부된 문서 참고하시고 운영하시는 환경에 맞게 사용하면 됩니다.   
감사합니다.   
   
# 참고 문서
* * *   
**- ceph rbd 관련**   
  * [RBD Volume Provisioner for Kubernetes 1.5+](https://github.com/kubernetes-incubator/external-storage/tree/master/ceph/rbd)
  * [RBD Volume Provisioner On Kubernetes Depolyment](https://github.com/kubernetes-incubator/external-storage/blob/master/ceph/rbd/deploy/README.md)
  * [Example Ceph rbd](https://docs.openshift.com/container-platform/3.5/install_config/storage_examples/ceph_example.html)
     
**- Kubernetes 문서**   
  * [PV 관련](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
  * [Types of Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes)
  * [PV Access Mode 관련](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)
  * [PV Reclaim Policy](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#reclaim-policy)
  * [Example PVC](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)

**- Example yaml**    
  * [chhanz Github](https://github.com/chhanz/k8s-study-example)
   
