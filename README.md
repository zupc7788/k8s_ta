# 개요
금번 활동을 통해 Private Cloud 환경을 구축하는 방법에 대해 이해한다.

---


# 목표 아키텍처

해당 메뉴얼을 통해 Kubernetes 기반의 Private Cloud 시스템 구축을 수행한다. (TA,PA, AA 영역)

또한 보안상 이슈가 될 수 있는 특정 기업의 Service 관련 컨텐츠는 포함하지 않으며,

구성원이 TA구성부터 PA, AA를 전반적으로 구성할 수 있도록 제품 중심으로 메뉴얼화 한다.



![1](https://user-images.githubusercontent.com/65584952/82286539-e8721e00-99d8-11ea-97b1-058548be68f6.PNG)
---

# 수행 절차

K8S기반 Private Cloud System 실행/운영 환경을 구성하며, 다음 절차에 따라 진행한다.

   1. Infra 구성(VM구축)
   2. Kubernetes(Master/Node) 구성
   3. Internal Network 구성
   4. Private Registry 구성
   5. Application Load Balancer 구성
   5. Shared Storage(NFS) 구성
   6. CI/CD 구성
   7. Monitoring(Prometheus/Grafana) 구성
   8. Logging(ELK) 구성
   9. Application Mig. (Refactoring)
   10. Cloud Native. (Rebuild)

---

# Infra 구성(VM구축)

## 1. 기본 Technical Architecture 설계

### [본 메뉴얼 구현 아키텍처 구성]
금번 테스트 환경 구축은 단일 Master Node에 3개 Worker Node로 이루어진 클러스터를 구축한다.

다만 실제 엔터프라이즈 환경의 운영 클러스터는 서비스 가용성을 고려하여 복수개의 Master 서버를 구성하는게 일반적인 구성이며, 해당 내용은 아래 참고 자료 확인하기 바란다. (로드밸런서가 필요해서 그렇지 멀티 마스터 서버 구성도 그렇지 어렵지 않습니다.)

![3](https://user-images.githubusercontent.com/65584952/82286703-4272e380-99d9-11ea-9ac3-17a3fcf8ea4e.jpg)


### [참고자료: 운영 환경 권고 아키텍처]

금번 과정은 단일 Master Node에 다중 Worker Node로 구성을 진행하나, 실제 엔터프라이즈 환경에서는 falut-tolerant를 고려 Master서버 다중화가 필요 하다. (일반적으로 3, 5, 7대로 구성)
또한 다중 Master Node <- Worker Node 구간 통신을 위해 로드밸런서(L4, L7, Haproxy, nginx등)를 통해 부하 분산처리 구성이 필요하다.

관련 구성 방안은 아래 Master 노드 이중화 관련 링크를 참조바란다.

https://kubernetes.io/ko/docs/tasks/administer-cluster/highly-available-master/

![a11](https://user-images.githubusercontent.com/65584952/82410978-d614e480-9aab-11ea-91aa-11b79ac4df7c.PNG)


---
## 2. VM 스펙 선정

VMWare에서 다음 스펙으로 VM주문한다. 

물론 테스트 환경이므로 최저 구성한 스펙이며, 실제 운영 환경은 훨씬 고스펙이 필요하다.

|*용도*|*Hostname*|*CPU*|*MEM*|*Disk*|
|-|-|-|-|-|
|Master|test-master|2Core|2GB|50GB|
|Node1|test-node1|2Core|4GB|50GB|
|Node2|test-node2|2Core|4GB|50GB|
|Node3|test-node3|2Core|4GB|50GB|


---

## 3. OS 이미지 다운로드

아래 다운로드 링크에서 이미지를 다운로드 한다.

http://ftp.kaist.ac.kr/CentOS/7.8.2003/isos/x86_64/CentOS-7-x86_64-Minimal-2003.iso


일반적으로 RedHat계열(RedHat, CentOS등)과 Ubuntu계열을 많이 쓰나, 금번 테스트는 RedHat계열인 CentOS를 사용한다.

CentOS이미지에도 DVD ISO / Everything ISO / Minimal ISO 등 여러가지가 존재하나,
일반적으로 On-premise에서는 DVD ISO를 주로 사용하며, Public Cloud에서는 Minimal을 주로 사용한다. (사유 X-Windows 유무)

참고로 아래 링크를 통해 최신 버전의 CentOS를 다운 받을 수 있다.

```
https://www.centos.org/download/
```

---


## 4. VM 생성

금번 환경 구축은 VMWare Workstation 또는 VMWare Player로 진행한다.

다만, 실제 엔터프라이즈 환경에서는 VMWare ESXi Hypervisor 또는 Xen 기반의 VM으로 시스템을 구축한다.

(Virtual Box로 해도 무방하나, Master-Node간의 라우팅 설정을 추가로 해줘야 하므로, 가급적 VMWare계열로 하는게 편함)

### New Virtual Machine 생성
![V1](https://user-images.githubusercontent.com/65584952/82286541-e90ab480-99d8-11ea-97ba-eb60a7a80a7e.png)

### Linux 생성
![V2](https://user-images.githubusercontent.com/65584952/82286543-e9a34b00-99d8-11ea-8a72-c327e07d1af1.png)

### VM명 지정 (Master, node1, node2, node3등 구분이 쉽게 지정)
![V3](https://user-images.githubusercontent.com/65584952/82286546-e9a34b00-99d8-11ea-86f2-ff1e013427db.png)

### First Disk 용량 지정 (향후 증설 가능하나 LVM작업이 추가 필요하므로 Node는 가급적 60GB이상 설정)
![V4](https://user-images.githubusercontent.com/65584952/82286547-ea3be180-99d8-11ea-931b-81011a8d559a.png)

### VM 기본 설정 완료
![V5](https://user-images.githubusercontent.com/65584952/82286534-e6a85a80-99d8-11ea-8ec5-7deb24e401f5.png)

### VM 세부 설정 수행
![V6](https://user-images.githubusercontent.com/65584952/82286535-e7d98780-99d8-11ea-9ce6-940afbad7d72.png)

### Processors와 Memory를 사전 정의한 스펙으로 변경
![V7](https://user-images.githubusercontent.com/65584952/82286537-e7d98780-99d8-11ea-9884-59e315e865ab.png)

---
## 5. Linux OS 설치

### 언어 설정: 한국어
![L1](https://user-images.githubusercontent.com/65584952/82287023-3176a200-99da-11ea-8c58-a8f38a0b7237.PNG)

### 파티션 설정:설치 대상 선택
![L2_2](https://user-images.githubusercontent.com/65584952/82292025-02196280-99e5-11ea-80f7-1ea56e87f2a3.png)

### 파티션 설정: 자동으로 선택
실제 운영기에 구성할때는 용도별로 파일시스템과 Swap 파티션을 구분하여 설정하나,
금번 테스트 환경은 자동으로 생성함 (/에 전체 파일시스템 할당 및 Swap 자동 생성)

![L3](https://user-images.githubusercontent.com/65584952/82287025-320f3880-99da-11ea-98b8-29746a3d906c.PNG)

### 네트워크 설정: 네트워크 및 호스트명 선택
![L4](https://user-images.githubusercontent.com/65584952/82287027-320f3880-99da-11ea-9d32-38b71aea24cc.PNG)

### 네트워크 설정: 호스트명 및 이더넷 카드 활성화
1. 이더넷 카드 On
2. 호스트명 입력 -> 적용
3. 완료
![l5](https://user-images.githubusercontent.com/65584952/82287018-2facde80-99da-11ea-8199-a29949452966.PNG)

### 설치 시작
OS설치를 시작하며, 설치가 완료되면 재부팅 한다.
![L6](https://user-images.githubusercontent.com/65584952/82287019-30457500-99da-11ea-803e-3d0d3240530d.PNG)

### ROOT암호 설정
![L7](https://user-images.githubusercontent.com/65584952/82287022-30de0b80-99da-11ea-8f7b-3c7c80842b73.PNG)

---
## 6. Linux OS 환경 설정

Kubernetes Cluster를 구성하기 위한 OS환경 설정을 수행한다. 실제 엔터프라이즈 환경에서는 운영 효율성, 성능, 가용성을 고려하여 추가로 아래 작업이 필요하나,

  1. OS커널 파라메터 튜닝
  2. 볼륨 파티션 분리
  3. 계정 분리 및 퍼미션 정책 적용
  4. rsyslog 구성
  5. NTP, DNS연동
  6. 서비스-백업망 분리
  7. NIC티밍 (VM일 경우 ESXi에서 진행)
  8. OS보안 진단
  9. 운영 솔루션 설치(백업, 백신, 모니터링 등)
  10. OS 방화벽 적용(iptables 또는 firewalld 구성 및 정책 적용)

금번 과정에서는 K8S설치에 필요한 최소한의 환경 설정 작업만 수행한다.


### 가. hosts파일 수정(/etc/hosts)
/etc/hosts파일에 각 서버의 호스트명과 IP를 입력한다. (IP는 ip a 명령으로 확인 가능)
```

vi /etc/hosts
192.168.111.169	test-master
192.168.111.172	test-node1
192.168.111.173	test-node2
192.168.111.170	test-node3
```

### 나. Linux 최신 업데이트 및 추가 유틸리티 설치 (root계정)

리눅스에 설치된 패키지를 최신 버전으로 업데이트 한다.

추가로 디스크 관리 패키지와 네트워크 관리 패키지를 설치한다.

#### [패키지 업데이트]
```
yum -y update
yum install -y yum-utils  device-mapper-persistent-data   lvm2 net-tools nfs-utils
```

### 다. SELinux(Security Enhanced Linux) 작동 중지

SELinux는 리눅스 커널 레벨의 보안 정책 관리 툴로 프로세스나 Native Object에 대한 관리를 수행한다.

따라서 기본적으로 OS가 제공하는 툴이 아닐 경우 정상 작동하지 않는 경우가 다수 존재하므로 반드시 Permissive로 변경한다.

1. Enforcing(작동), 2. Permissive(작동하지 않으나 기록), 3. Disabled(완전 중지)

#### [설정 변경]
```
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

#### [확인]
```
[root@test-node1 ~]# getenforce
Permissive
```

### 라. OS 스왑 중지
메모리 부족을 해결하기 위해 Linux는 Swap Disk를 제공한다.

하지만 고성능이 요구되는 시스템의 경우는 일반적으로 Swap을 사용하지 않고, 고용량 메모리를 할당하는것을 권고한다.

(ex. Kubernetes, Oracle Database등)

#### [설정 전: swap이 3GB 설정되어 있음]
```
[root@test-master ~]# free -m
              total        used        free      shared  buff/cache   available
Mem:           2784         217        2033          11         533        2338
Swap:          3071           0        3071
```

#### [설정: swap off 및 fstab에 영구 swap off 설정 적용]
```
swapoff -a && sed -i '/swap/s/^/#/' /etc/fstab
```

#### [확인: Swap이 0GB로 변경 됨]
```
[root@test-master ~]# free -m
              total        used        free      shared  buff/cache   available
Mem:           2784         216        2035          11         532        2339
Swap:             0           0           0

```

### 마. OS방화벽 중지
보안정책이 잘 짜여진 기업의 경우는, 네트워크 레벨의 방화벽 뿐만 아니라 OS에서 Outbound/Inbound통신을 제어한다.

Linux의 경우 전통적으로 RedHat 7이상에선 보통 Firewalld를 사용하며, 전통적으로는 iptables로 제어하기도 한다.

하지만 Kubernetes의 경우는, 자체적으로 Kube-proxy정책에 의해 라우팅 룰이 정의 되므로 반드시 OS방화벽을 Off해야 한다.

(사실 켜도 룰이 작동 안 된다.)

#### [OS방화벽 중지]
```
systemctl stop firewalld && systemctl disable firewalld
```

#### [OS방화벽 상태 확인]
다음과 같이 Active상태가 "inactive (dead)" 일 경우 정상적으로 중지된 것이다.

```
[root@test-master ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```

### 바. 브릿지 네트워크 관련 OS커널 파라미터 변경

```
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

### 사. 고정IP 설정

기본적으로 VM이 구성되면, DHCP에 의해 유동IP가 할당된다.
해당 IP는 언제든 바뀔 수 있으므로, 향후에 VM재부팅 시에 정상작동되지 않을 수 있다.

따라서 아래와 같이 /etc/sysconfig/network-scripts/ifcfg-ens33 파일에 IPADDR추가를 통해 ens33 NIC디바이스에 고정 IP를 할당한다.

IP는 /etc/hosts에 선언한 IP와 동일하게 구성하면 된다. (DHCP로 부터 할당 받은 IP를 향후에 고정으로 쓸 예정)


```
vi /etc/sysconfig/network-scripts/ifcfg-ens33

TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
#BOOTPROTO="dhcp"   
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens33"
UUID="852aa063-eb1f-4406-bc68-9c4a9f1ae6ee"
DEVICE="ens33"
ONBOOT="yes"
IPADDR="192.168.111.169" # 고정 IP를 선언한다. DHCP에서 할당 받은, /etc/hosts파일에 추가한 IP로 지정하면 향후 해당 IP가 고정된다.

```


---
## 7. Docker, Kubernetes 엔진 설치

### 가. Docker 설치

#### [설치 명령]
Docker 설치 위한 Yum Repository를 추가 한 후, Yum으로 패키지 설치 진행한다.

Yum Repository를 추가하지 않을 경우, OS Default Repository에서 다운로드 받으므로 주의 한다.

또한 만약 내부망에서 구성할 경우에는 외부 Repository를 연동 할 수 없으므로 Forward Proxy방식의 Yum Repository를 추가 구축해야 한다.

```
yum-config-manager    --add-repo     https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce docker-ce-cli containerd.io -y
```

#### [진행 과정]


```
[root@test-master ~]# yum-config-manager    --add-repo     https://download.docker.com/linux/centos/docker-ce.repo
Loaded plugins: fastestmirror
adding repo from: https://download.docker.com/linux/centos/docker-ce.repo
grabbing file https://download.docker.com/linux/centos/docker-ce.repo to /etc/yum.repos.d/docker-ce.repo
repo saved to /etc/yum.repos.d/docker-ce.repo
[root@test-master ~]# yum install docker-ce docker-ce-cli containerd.io -y
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirror.navercorp.com
 * extras: mirror.navercorp.com
 * updates: mirror.kakao.com
docker-ce-stable                                                                                                                      | 3.5 kB  00:00:00
(1/2): docker-ce-stable/x86_64/primary_db                                                                                             |  42 kB  00:00:00
(2/2): docker-ce-stable/x86_64/updateinfo                                                                                             |   55 B  00:00:00
Resolving Dependencies
--> Running transaction check
---> Package containerd.io.x86_64 0:1.2.13-3.2.el7 will be installed
--> Processing Dependency: container-selinux >= 2:2.74 for package: containerd.io-1.2.13-3.2.el7.x86_64
--> Processing Dependency: libseccomp for package: containerd.io-1.2.13-3.2.el7.x86_64
---> Package docker-ce.x86_64 3:19.03.9-3.el7 will be installed
--> Processing Dependency: libcgroup for package: 3:docker-ce-19.03.9-3.el7.x86_64
---> Package docker-ce-cli.x86_64 1:19.03.9-3.el7 will be installed
--> Running transaction check
---> Package container-selinux.noarch 2:2.119.1-1.c57a6f9.el7 will be installed
--> Processing Dependency: policycoreutils-python for package: 2:container-selinux-2.119.1-1.c57a6f9.el7.noarch
---> Package libcgroup.x86_64 0:0.41-21.el7 will be installed
---> Package libseccomp.x86_64 0:2.3.1-4.el7 will be installed
--> Running transaction check
---> Package policycoreutils-python.x86_64 0:2.5-34.el7 will be installed
--> Processing Dependency: setools-libs >= 3.3.8-4 for package: policycoreutils-python-2.5-34.el7.x86_64
--> Processing Dependency: libsemanage-python >= 2.5-14 for package: policycoreutils-python-2.5-34.el7.x86_64
--> Processing Dependency: audit-libs-python >= 2.1.3-4 for package: policycoreutils-python-2.5-34.el7.x86_64
--> Processing Dependency: python-IPy for package: policycoreutils-python-2.5-34.el7.x86_64
--> Processing Dependency: libqpol.so.1(VERS_1.4)(64bit) for package: policycoreutils-python-2.5-34.el7.x86_64
--> Processing Dependency: libqpol.so.1(VERS_1.2)(64bit) for package: policycoreutils-python-2.5-34.el7.x86_64
--> Processing Dependency: libapol.so.4(VERS_4.0)(64bit) for package: policycoreutils-python-2.5-34.el7.x86_64
--> Processing Dependency: checkpolicy for package: policycoreutils-python-2.5-34.el7.x86_64
--> Processing Dependency: libqpol.so.1()(64bit) for package: policycoreutils-python-2.5-34.el7.x86_64
--> Processing Dependency: libapol.so.4()(64bit) for package: policycoreutils-python-2.5-34.el7.x86_64
--> Running transaction check
---> Package audit-libs-python.x86_64 0:2.8.5-4.el7 will be installed
---> Package checkpolicy.x86_64 0:2.5-8.el7 will be installed
---> Package libsemanage-python.x86_64 0:2.5-14.el7 will be installed
---> Package python-IPy.noarch 0:0.75-6.el7 will be installed
---> Package setools-libs.x86_64 0:3.3.8-4.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

=============================================================================================================================================================
 Package                                   Arch                      Version                                       Repository                           Size
=============================================================================================================================================================
Installing:
 containerd.io                             x86_64                    1.2.13-3.2.el7                                docker-ce-stable                     25 M
 docker-ce                                 x86_64                    3:19.03.9-3.el7                               docker-ce-stable                     24 M
 docker-ce-cli                             x86_64                    1:19.03.9-3.el7                               docker-ce-stable                     38 M
Installing for dependencies:
 audit-libs-python                         x86_64                    2.8.5-4.el7                                   base                                 76 k
 checkpolicy                               x86_64                    2.5-8.el7                                     base                                295 k
 container-selinux                         noarch                    2:2.119.1-1.c57a6f9.el7                       extras                               40 k
 libcgroup                                 x86_64                    0.41-21.el7                                   base                                 66 k
 libseccomp                                x86_64                    2.3.1-4.el7                                   base                                 56 k
 libsemanage-python                        x86_64                    2.5-14.el7                                    base                                113 k
 policycoreutils-python                    x86_64                    2.5-34.el7                                    base                                457 k
 python-IPy                                noarch                    0.75-6.el7                                    base                                 32 k
 setools-libs                              x86_64                    3.3.8-4.el7                                   base                                620 k

Transaction Summary
=============================================================================================================================================================
Install  3 Packages (+9 Dependent packages)

```
### 나. Docker 기동
#### [Docker 기동]
```
systemctl start docker && systemctl enable docker
```

### 다. Kubernetes Repository 추가
아래 명령어를 통해 /etc/yum.repos.d 경로에 Kubernetes Repository를 추가한다.

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

### 라. Kubernetes Repository 추가 확인

yum -y update를 통해 추가된 Repository를 확인한다.

#### [확인 절차]
```
[root@test-master ~]# yum -y update
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirror.navercorp.com
 * extras: mirror.navercorp.com
 * updates: mirror.kakao.com
kubernetes/signature                                                                                  |  454 B  00:00:00
Retrieving key from https://packages.cloud.google.com/yum/doc/yum-key.gpg
Importing GPG key 0xA7317B0F:
 Userid     : "Google Cloud Packages Automatic Signing Key <gc-team@google.com>"
 Fingerprint: d0bc 747f d8ca f711 7500 d6fa 3746 c208 a731 7b0f
 From       : https://packages.cloud.google.com/yum/doc/yum-key.gpg
Retrieving key from https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
kubernetes/signature                                                                                  | 1.4 kB  00:00:00 !!!
kubernetes/primary                                                                                    |  68 kB  00:00:01
kubernetes                                                                                                           496/496
No packages marked for update

```

### 마. Kubernetes 패키지 설치

yum을 통해 Master 및 Node에 Kubernetes 패키지를 설치한다.

#### [패키지 설치 절차]
```
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

```

#### [패키지 설치 진행 내역]
```
[root@test-node1 ~]# yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: ftp.kaist.ac.kr
 * extras: ftp.kaist.ac.kr
 * updates: ftp.kaist.ac.kr
kubernetes/signature                                                                                  |  454 B  00:00:00
Retrieving key from https://packages.cloud.google.com/yum/doc/yum-key.gpg
Importing GPG key 0xA7317B0F:
 Userid     : "Google Cloud Packages Automatic Signing Key <gc-team@google.com>"
 Fingerprint: d0bc 747f d8ca f711 7500 d6fa 3746 c208 a731 7b0f
 From       : https://packages.cloud.google.com/yum/doc/yum-key.gpg
Retrieving key from https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
kubernetes/signature                                                                                  | 1.4 kB  00:00:00 !!!
kubernetes/primary                                                                                    |  68 kB  00:00:01
kubernetes                                                                                                           496/496
Resolving Dependencies
--> Running transaction check
---> Package kubeadm.x86_64 0:1.18.2-0 will be installed
--> Processing Dependency: kubernetes-cni >= 0.7.5 for package: kubeadm-1.18.2-0.x86_64
--> Processing Dependency: cri-tools >= 1.13.0 for package: kubeadm-1.18.2-0.x86_64
---> Package kubectl.x86_64 0:1.18.2-0 will be installed
---> Package kubelet.x86_64 0:1.18.2-0 will be installed
--> Processing Dependency: socat for package: kubelet-1.18.2-0.x86_64
--> Processing Dependency: conntrack for package: kubelet-1.18.2-0.x86_64
--> Running transaction check
---> Package conntrack-tools.x86_64 0:1.4.4-7.el7 will be installed
--> Processing Dependency: libnetfilter_cttimeout.so.1(LIBNETFILTER_CTTIMEOUT_1.1)(64bit) for package: conntrack-tools-1.4.4-7.el7.x86_64
--> Processing Dependency: libnetfilter_cttimeout.so.1(LIBNETFILTER_CTTIMEOUT_1.0)(64bit) for package: conntrack-tools-1.4.4-7.el7.x86_64
--> Processing Dependency: libnetfilter_cthelper.so.0(LIBNETFILTER_CTHELPER_1.0)(64bit) for package: conntrack-tools-1.4.4-7.el7.x86_64
--> Processing Dependency: libnetfilter_queue.so.1()(64bit) for package: conntrack-tools-1.4.4-7.el7.x86_64
--> Processing Dependency: libnetfilter_cttimeout.so.1()(64bit) for package: conntrack-tools-1.4.4-7.el7.x86_64
--> Processing Dependency: libnetfilter_cthelper.so.0()(64bit) for package: conntrack-tools-1.4.4-7.el7.x86_64
---> Package cri-tools.x86_64 0:1.13.0-0 will be installed
---> Package kubernetes-cni.x86_64 0:0.7.5-0 will be installed
---> Package socat.x86_64 0:1.7.3.2-2.el7 will be installed
--> Running transaction check
---> Package libnetfilter_cthelper.x86_64 0:1.0.0-11.el7 will be installed
---> Package libnetfilter_cttimeout.x86_64 0:1.0.0-7.el7 will be installed
---> Package libnetfilter_queue.x86_64 0:1.0.2-2.el7_2 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

=============================================================================================================================
 Package                               Arch                  Version                         Repository                 Size
=============================================================================================================================
Installing:
 kubeadm                               x86_64                1.18.2-0                        kubernetes                8.8 M
 kubectl                               x86_64                1.18.2-0                        kubernetes                9.5 M
 kubelet                               x86_64                1.18.2-0                        kubernetes                 21 M
Installing for dependencies:
 conntrack-tools                       x86_64                1.4.4-7.el7                     base                      187 k
 cri-tools                             x86_64                1.13.0-0                        kubernetes                5.1 M
 kubernetes-cni                        x86_64                0.7.5-0                         kubernetes                 10 M
 libnetfilter_cthelper                 x86_64                1.0.0-11.el7                    base                       18 k
 libnetfilter_cttimeout                x86_64                1.0.0-7.el7                     base                       18 k
 libnetfilter_queue                    x86_64                1.0.2-2.el7_2                   base                       23 k
 socat                                 x86_64                1.7.3.2-2.el7                   base                      290 k

Transaction Summary
=============================================================================================================================
Install  3 Packages (+7 Dependent packages)

Total download size: 55 M
Installed size: 246 M
Downloading packages:
중략
```


### 바. Kubernetes 서비스 기동 및 영구 서비스 등록
systemctl start 명령으로 서비스 기동.

systemctl enable 명령으로 서비스 등록. (추후 리눅스 재부팅시에도 자동 기동됨)

#### [서비스 기동 및 등록]
```
systemctl enable kubelet && systemctl start kubelet
```

---
## 8. Master Node 구성

### [Kubeadm 초기화 수행] (Master Node에서만 수행)

kubeadm init명령으로 수행하여 Master Node를 초기 구성한다.
pod-network-cidr은 K8S POD의 네트워크 설정이며, POD간의 통신을 위한 애드온인 pod Network Add-on 이 사용하는 CIDR값이다.

해당값은 실제 호스트 네트워크와 절대로 겹치지 않게 주의하여 구성해야 한다. (해당 설정을 하지 않으면 디폴트로 구성 됨)

```
kubeadm init --pod-network-cidr 10.244.0.0/16
```

```
[root@test-master /]# kubeadm init --pod-network-cidr 10.244.0.0/16
W0520 09:12:34.249547    9675 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.18.2
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
        [WARNING Service-Kubelet]: kubelet service is not enabled, please run 'systemctl enable kubelet.service'
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [test-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.111.169]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [test-master localhost] and IPs [192.168.111.169 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [test-master localhost] and IPs [192.168.111.169 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
W0520 09:12:38.123122    9675 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
W0520 09:12:38.124509    9675 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 16.503483 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.18" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node test-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node test-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: dbfxly.upq447u4wojz3agj
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.111.169:6443 --token dbfxly.upq447u4wojz3agj \
    --discovery-token-ca-cert-hash sha256:28d12bc6920e3ef9a8f61f5f3c3ae55da980ab3c5253e450128ef51743280eaa
```
위 두 부분은 Client와 Node 서버 구성에 사용하므로 반드시 기록해 둔다.

---
## 9. K8S Client 구성

kubeadm 실행 시  나온 config파일로 K8S접속 환경을 구성한다.
해당 설정은 방화벽만 오픈되어 있으면 어떠한 Client PC에서도 활용가능하다.

다만 기본적으로, Master와 Worker Node에는 기본적으로 해당 구성을 하길 권고하며,
작업용 로컬PC에도 구성해서 운영하는게 편리하다.

### [K8S접속 환경 설정]
아래 명령을 Master서버에서 실행.

```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### [접속 테스트]
```
[root@test-master /]# kubectl get all
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   3m12s
```

### [참고자료] Windows에 Ubuntu WSL방식으로 설치하여 Client 환경 구성

윈도우는 WSL방식의 Native로 기동되는 리눅스를 제공한다.
따라서 굳이 별도의 작업용 Client서버를 구축할 필요없이, 로컬에서 직접 작업이 가능하도록 환경 구성이 가능하다.

#### 가. "Linux용 Windows 하위 시스템" 활성화
![w1](https://user-images.githubusercontent.com/65584952/82391803-37718f00-9a7d-11ea-8819-83fac627162a.PNG)
![w2](https://user-images.githubusercontent.com/65584952/82391793-34769e80-9a7d-11ea-8b8a-a5d5bb1101a9.PNG)
![w3](https://user-images.githubusercontent.com/65584952/82391796-35a7cb80-9a7d-11ea-95f0-08cbe1095faa.PNG)

#### 나. Ubuntu 18.04 LTS 설치
![w4](https://user-images.githubusercontent.com/65584952/82391799-35a7cb80-9a7d-11ea-8bf7-6ec08e908762.PNG)
![w5](https://user-images.githubusercontent.com/65584952/82391801-36406200-9a7d-11ea-859a-6a28ca14a803.PNG)

#### 다. Ubuntu WSL 실행
![w6](https://user-images.githubusercontent.com/65584952/82391802-36d8f880-9a7d-11ea-8dae-7ad95c4b4af8.PNG)

#### 라. Windows에 Kubectl 설치
```
sudo apt-get update && sudo apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```

#### 마. 클러스터 설정 복사

Master의 .kube/config 파일을 동일한 경로로 카피한다.

![ka](https://user-images.githubusercontent.com/65584952/82393459-625de200-9a81-11ea-97c1-1112020b7483.PNG)

#### 바. 테스트

로컬PC에서 운영 클러스터에 정상적으로 명령이 수행됨을 확인 함.

![ka2](https://user-images.githubusercontent.com/65584952/82393533-a650e700-9a81-11ea-9417-7f703af82018.PNG)

---
## 10. Container Network Interface 구성

POD간의 통신이 이루어질 수 있도록 CNI를 설치하여 오버레이 네트워크(overlay network)를 구성한다.

### [설치 절차]

CNI플러그인은 Calico, Flannel, Weave-net등 다양한 CNI플러그인을 제공하며,
대부분의 역할을 유사하나, 네트워크 정책 관리의 특성이나 보안관련 특성이 상이할 수 있다.

다양한 CNI를 선택적으로 설치할 수 있으나, 금번 환경 구성에서는 weave-net을 설치한다. 


#### 가. CNI 설치

```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-v                                                                          ersion=$(kubectl version | base64 | tr -d '\n')"
```

#### 나. 설치 확인

아래와 같이 weave-net POD와 coredns POD가 정상적으로 Running상태인지 확인한다.

```
[root@test-master .kube]# kubectl get po --all-namespaces
NAMESPACE     NAME                                  READY   STATUS    RESTARTS   AGE
kube-system   coredns-66bff467f8-l9mwf              1/1     Running   0          58m
kube-system   coredns-66bff467f8-s4qc7              1/1     Running   0          58m
kube-system   etcd-test-master                      1/1     Running   1          58m
kube-system   kube-apiserver-test-master            1/1     Running   1          58m
kube-system   kube-controller-manager-test-master   1/1     Running   1          58m
kube-system   kube-proxy-xvtbz                      1/1     Running   1          58m
kube-system   kube-scheduler-test-master            1/1     Running   1          58m
kube-system   weave-net-t9zk5                       2/2     Running   0          64s
```
### [참고 자료] CNI를 설치하는 이유?

POD간의 통신이 이루어질 수 있도록 CNI를 설치하여 오버레이 네트워크(overlay network)를 구성한다.
해당 네트워크를 구성해야 실제 노드간의 Flat한 통신이 가능하며, 만약 설정하지 않을 경우, 단일 노드를 벗어나는 통신은 되지 않는다.

참고 자료: https://medium.com/google-cloud/understanding-kubernetes-networking-pods-7117dd28727

#### 가. Pod 안에 도커 컨테이너 간 네트워킹
Pod 안에 있는 모든 컨테이너들은 localhost로 서로 통신할 수 있다.
그 원리는 도커 컨테이너가 시작될 때 만들어지는 veth0라는 가상 네트워크 인터페이스가 있는데 쿠버네티스가 PPod를 생성할 때 내부에 있는 컨테이너기리 veth0 가상 인터페이스를 서로 공유하기 해서 서로 같은 네트워크 상에 존재할 수 있도록 하기 때문이다.

따라서 아래 그림에서 보이는 Container-1과 Container-2는 하나의 Pod 안에서 veth0 가상 인터페이스를 공유하고 있고 동일한 IP 주소로 갖는다. 하지만 당연히 두 컨테이너는 서로 다른 포트를 사용해야 서로 통신 할 수 있게 된다.

이렇게 Pod 안에 컨테이너들은 격리된 상태에서 독립적인 일을 수행할 수 있으면서 서로 네트워크 통신이 가능하다는 장점을 갖게 되었다.

![n11](https://user-images.githubusercontent.com/65584952/82396178-385bee00-9a88-11ea-815c-dc336cf065da.PNG)



#### 나. CNI없이 구성한 쿠버네티스 네트워킹
쿠버네티스 클러스터는 Master 노드와 Worker 노드로 구성된다. 클러스터에 소속된 실제 노드(서버)들은 보통 같은 사설(private) 네트워크에 속하기 때문에 서로 통신할 수 있다.
하지만 Pod에 할당된 IP는 이 사설 네트워크 대역에 포함되지 않는다.
Pod 안에 컨테이너들은 pause컨테이너가 만든 가상 인터페이스를 통해 호스트의 docker0 브리지 네트워크로 연결될 것이다.
그리고 해당 노드로 요청이 도착하면 가장 먼저 호스트의 eth0을 거친 후 docker0 브리지를 통해 veth0으로 전송된다. 그 다음에 veth0에 연결된 컨테이너의 eth0로 전달된다.


그런데 만약 위 그림에 있는 왼쪽 Worker 노드의 Pod에서 오른쪽 Worker 노드의 Pod로 요청을 보내려면 어떻게 해야할까?
라우터가 오른쪽 노드(10.100.0.3)에 172.17.0.2IP를 갖는 Pod가 있다는 사실을 알고 있다면 적절하게 패킷을 보낼 수 있을 것이다.
하지만 문제는 또 다른 노드에 있는 Pod도 172.17.0.2 IP를 가질 수 수 있다는 것이다. 그 이유는 Worker 노드마다 veth0 가상 인터페이스에 의한 Pod IP 주소가 같을 수 있기 때문이다.

![n22](https://user-images.githubusercontent.com/65584952/82396179-38f48480-9a88-11ea-8934-339d77254770.PNG)



#### 다. CNI를 이용한 쿠버네티스 오버레이 네트워킹
이 문제를 해결하기 위해서 “오버레이 네트워크(overlay network)“라는 방식을 통해 Pod가 서로 통신할 수 있는 구성을 만들 수 있다.
“오버레이 네트워크”란 실제 노드 간의 네트워크 위에 별도 Flat한 네트워크를 구성하는 것을 의미한다.

쿠버네티스는 자체적으로 네트워크 구성 해주지 않기 때문에 클러스터를 구성할 때 CNI(Container Network Interface) 규약을 구현한 CNI 플러그인을 함께 설치해야 한다.
만약 kubeadm으로 쿠버네티스 클러스터를 구성하려고 하면 Master 노드를 초기화한 후 CNI 네트워크 플러그인이 설치되지 않으면, CoreDNS Pod가 시작되지 않고 PENDING되어 있는 것을 볼 수 있다.

CNI 플러그인 중 오버레이 네트워크를 제공하는 대표적인 플러그인은 Flannel, Weave-net등이 있으며, Calico는 오버레이 네트워크는 물론이며, Pod 간의 통신과 다른 외부 네트워크 통신을 허용하는 네트워크 정책 관리 기능을 제공하기도 한다.

![n3](https://user-images.githubusercontent.com/65584952/82396070-021e6e80-9a88-11ea-8622-e5e210576636.png)


---
## 11. Worker Node 구성

Node1~3은 본 클러스터 환경 내에서 Kubernetes 에이전트 역할을 한다.

따라서 Master Node 구성 섹션에서 복사한 kubeadm join명령을 test-node1~3 머신에서 각각 실행하여, Cluster에 조인한다.

![3](https://user-images.githubusercontent.com/65584952/82286703-4272e380-99d9-11ea-9ac3-17a3fcf8ea4e.jpg)


### [명령어]

해당 명령은 Master kubeadm init할때 기록한 kubeadm join 명령을 각각의 Worker Node에서 수행하면 된다.

```
kubeadm join 192.168.111.169:6443 --token dbfxly.upq447u4wojz3agj \
>     --discovery-token-ca-cert-hash sha256:28d12bc6920e3ef9a8f61f5f3c3ae55da980ab3c5253e450128ef51743280eaa
```

### [수행 화면]
```
[root@test-node1 ~]# kubeadm join 192.168.111.169:6443 --token dbfxly.upq447u4wojz3agj \
>     --discovery-token-ca-cert-hash sha256:28d12bc6920e3ef9a8f61f5f3c3ae55da980ab3c5253e450128ef51743280eaa
W0520 11:08:51.764805    1642 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
        [WARNING Service-Kubelet]: kubelet service is not enabled, please run 'systemctl enable kubelet.service'
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.18" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

[root@test-node1 ~]#

```

### [Worker Node 조인 확인]

Node서버의 상태와 POD의 상태가 모두 Ready 일 경우 정상이다.
참고로 오버레이 네트워크가 활성화 되야, CoreDNS가 Running이 되며, 오버레이 네트워크가 없으면 정상적으로 작동되지 않는다.

또한 CoreDNS는 Kubernetes 1.13버전 이후에 지원하는 기본 DNS이며, 이전 버젼의 Kubernetes에서는 KubeDNS가 설치되어 있다.

```
webwas@DESKTOP-JQ6ILBP:~/.kube$ kubectl get node
NAME          STATUS   ROLES    AGE     VERSION
test-master   Ready    master   122m    v1.18.2
test-node1    Ready    <none>   6m36s   v1.18.2
test-node2    Ready    <none>   6m5s    v1.18.2
test-node3    Ready    <none>   6m1s    v1.18.2


webwas@DESKTOP-JQ6ILBP:~/.kube$ kubectl get pod --all-namespaces
NAMESPACE     NAME                                  READY   STATUS    RESTARTS   AGE
kube-system   coredns-66bff467f8-l9mwf              1/1     Running   2          122m
kube-system   coredns-66bff467f8-s4qc7              1/1     Running   1          122m
kube-system   etcd-test-master                      1/1     Running   2          122m
kube-system   kube-apiserver-test-master            1/1     Running   2          122m
kube-system   kube-controller-manager-test-master   1/1     Running   3          122m
kube-system   kube-proxy-582l8                      1/1     Running   0          6m6s
kube-system   kube-proxy-l8jpm                      1/1     Running   0          6m41s
kube-system   kube-proxy-qv9mr                      1/1     Running   0          6m10s
kube-system   kube-proxy-xvtbz                      1/1     Running   2          122m
kube-system   kube-scheduler-test-master            1/1     Running   3          122m
kube-system   weave-net-5vmrt                       2/2     Running   1          6m6s
kube-system   weave-net-ss2s2                       2/2     Running   0          6m41s
kube-system   weave-net-t9zk5                       2/2     Running   4          65m
kube-system   weave-net-x2w5s                       2/2     Running   0          6m10s
```
---
## 12. Ingress Controller 구성

인그레스(ingress)는 클러스터 외부에서 내부로 접근하는 요청들을 어떻게 처리할지 정의해둔 규칙들의 모음이다. 외부에서 접근가능한 URL을 사용할 수 있게 하고, 트래픽 로드밸런싱도 해주고, SSL 인증서 처리도 해주고, 도메인 기반으로 가상 호스팅을 제공하기도 한다. 인그레스 자체는 이런 규칙들을 정의해둔 자원이고 이런 규칙들을 실제로 동작하게 해주는게 인그레스 컨트롤러(ingress controller)이다.

인그레스 컨트롤러는, Kubernetes에서 필수적인 항목은 아니지만 Private Cloud에서는 일반적으로 단일 서비스IP를 기반으로 여러 도메인을 라우팅 처리하므로, 사실상 Private Cloud환경에서는 필수 솔루션이라고 봐도 무방하다. 

반면 Public Cloud에서는 Managed Service 기반의 API Gateway를 사용하는게 성능상 유리할 수 있으며.
External IP를 Loadbalancer와 Zuul, Kong, IBM API Gateway등과 연동하여 손쉽게 구성 가능하므로 좀 더 자유도가 높다.

Ingress Controller는, 서비스Refactoring과 MSA를 이해하는데도 매우 중요한 항목이므로, 반드시 마스터 해야 한다.

![ing](https://user-images.githubusercontent.com/65584952/82411766-5851d880-9aad-11ea-9872-35683b3a2c4b.PNG)


### [설치 절차]

Ingress Controller를 손쉽게 설치하기 위해서는 helm을 구성한다.
helm은 필수는 아니지만, 향후 다양한 플랫폼 설치를 위해 빈번하게 사용하는 툴이므로 반드시 구성해야 한다.

#### 가. helm 설치

```
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash
kubectl --namespace kube-system create sa tiller      # helm 의 설치관리자를 위한 시스템 사용자 생성
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller
helm repo update
```

#### 나. helm 관련 참고 자료

helm 설치: https://medium.com/google-cloud/installing-helm-in-google-kubernetes-engine-7f07f43c536e

helm 차트: https://www.influxdata.com/blog/packaged-kubernetes-deployments-writing-helm-chart/


#### 다. Ingress Controller 설치

[설치 명령]

```
kubectl create namespace ingress-basic
helm install --name nginx-ingress stable/nginx-ingress --namespace=ingress-basic
kubectl get all --namespace=ingress-basic
```

[설치 수행 화면]

```
webwas@DESKTOP-JQ6ILBP:~/.kube$ helm install --name nginx-ingress stable/nginx-ingress --namespace=ingress-basic
NAME:   nginx-ingress
LAST DEPLOYED: Wed May 20 15:28:38 2020
NAMESPACE: ingress-basic
STATUS: DEPLOYED

RESOURCES:
==> v1/ClusterRole
NAME           CREATED AT
nginx-ingress  2020-05-20T06:28:38Z

==> v1/ClusterRoleBinding
NAME           ROLE                       AGE
nginx-ingress  ClusterRole/nginx-ingress  0s

==> v1/Deployment
NAME                           READY  UP-TO-DATE  AVAILABLE  AGE
nginx-ingress-controller       0/1    1           0          0s
nginx-ingress-default-backend  0/1    1           0          0s

==> v1/Pod(related)
NAME                                            READY  STATUS             RESTARTS  AGE
nginx-ingress-controller-57fcfc77bc-jlz6z       0/1    ContainerCreating  0         0s
nginx-ingress-default-backend-7c868597f4-9q6x4  0/1    ContainerCreating  0         0s
nginx-ingress-controller-57fcfc77bc-jlz6z       0/1    ContainerCreating  0         0s
nginx-ingress-default-backend-7c868597f4-9q6x4  0/1    ContainerCreating  0         0s

==> v1/Role
NAME           CREATED AT
nginx-ingress  2020-05-20T06:28:38Z

==> v1/RoleBinding
NAME           ROLE                AGE
nginx-ingress  Role/nginx-ingress  0s

==> v1/Service
NAME                           TYPE          CLUSTER-IP     EXTERNAL-IP  PORT(S)                     AGE
nginx-ingress-controller       LoadBalancer  10.108.48.35   <pending>    80:32334/TCP,443:30570/TCP  0s
nginx-ingress-default-backend  ClusterIP     10.105.89.235  <none>       80/TCP                      0s

==> v1/ServiceAccount
NAME                   SECRETS  AGE
nginx-ingress          1        0s
nginx-ingress-backend  1        0s


NOTES:
The nginx-ingress controller has been installed.
It may take a few minutes for the LoadBalancer IP to be available.
You can watch the status by running 'kubectl --namespace ingress-basic get services -o wide -w nginx-ingress-controller'

An example Ingress that makes use of the controller:

  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    annotations:
      kubernetes.io/ingress.class: nginx
    name: example
    namespace: foo
  spec:
    rules:
      - host: www.example.com
        http:
          paths:
            - backend:
                serviceName: exampleService
                servicePort: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
        - hosts:
            - www.example.com
          secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```


#### 라. 정상 설치 확인

helm status nginx-ingress

```
webwas@DESKTOP-JQ6ILBP:~/.kube$ helm status nginx-ingress
LAST DEPLOYED: Wed May 20 15:28:38 2020
NAMESPACE: ingress-basic
STATUS: DEPLOYED

RESOURCES:
==> v1/ClusterRole
NAME           CREATED AT
nginx-ingress  2020-05-20T06:28:38Z

==> v1/ClusterRoleBinding
NAME           ROLE                       AGE
nginx-ingress  ClusterRole/nginx-ingress  3m34s

==> v1/Deployment
NAME                           READY  UP-TO-DATE  AVAILABLE  AGE
nginx-ingress-controller       1/1    1           1          3m34s
nginx-ingress-default-backend  1/1    1           1          3m34s

==> v1/Pod(related)
NAME                                            READY  STATUS   RESTARTS  AGE
nginx-ingress-controller-57fcfc77bc-jlz6z       1/1    Running  0         3m34s
nginx-ingress-default-backend-7c868597f4-9q6x4  1/1    Running  0         3m34s
nginx-ingress-controller-57fcfc77bc-jlz6z       1/1    Running  0         3m34s
nginx-ingress-default-backend-7c868597f4-9q6x4  1/1    Running  0         3m34s

```

#### 마. 서비스IP 및 포트 확인

Ingress Controller의 IP와 포트는 향수 사용자 Request를 처리하는 Endpoint 역할을 수행한다.
따라서 해당 IP/Port를 반드시 숙지하고 있어야 한다.

아래와 같이 HTTP포트는 32334, SSL포트는 30570임을 알 수 있다.

참골, Public Cloud의 경우는 External-IP가 Cloud API를 통해 자동으로 생성되어 외부IP를 네트워크 장비에 자동으로 Bind하는 명령을 제공하지만, Private Cloud의 경우는 External-IP가 할당 될 수 없으므로, Pending상태로 유지될 수 밖에 없다. 이는 정상이다.

```
webwas@DESKTOP-JQ6ILBP:~/.kube$ kubectl get svc -n ingress-basic
NAME                            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
nginx-ingress-controller        LoadBalancer   10.108.48.35    <pending>     80:32334/TCP,443:30570/TCP   4m45s
nginx-ingress-default-backend   ClusterIP      10.105.89.235   <none>        80/TCP                       4m45s
```

각 노드 서버에는 NodePort로 30570, 32334가 바인딩되어 있다.

```
[root@test-node1 ~]# netstat -an | egrep "32334|30570"
tcp        0      0 0.0.0.0:30570           0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:32334           0.0.0.0:*               LISTEN
```

---

## 13. Application Load Balancer 구성

Kubernetes Node는 복수개로 구성되어 있으나, 사용자의 요청을 받는 EndPoint는 단일 해야 한다.
따라서 L4, L7 스위치나 SW방식의 로드밸런서 구성을 통해 복수개의 Worker Node로 부하 분산이 반드시 필요하며, 

### [H/W방식의 운영 환경 부하 분산]

따라서 Private Cloud의 경우는 L4 Switch를 통한 로드밸런싱이 필요하며, Public Cloud의 경우 Application Load Balancer를 사용하거나 NetScaler와 같은 부하 분산 장비를 구축하여 단일 EndPoint를 제공해야 한다.

![lb](https://user-images.githubusercontent.com/65584952/82416387-e67d8d00-9ab4-11ea-8137-4c215c0b9775.PNG)

### [HaProxy를 이용한 부하 분산]

금번 시스템 구축의 경우는 가상의 테스트장비를 구축하는 과정이므로, SW방식의 HaProxy를 이용한 로드밸런서를 구현하겠다.
SW방식의 부하 분산은 HaProxy를 사용하거나, nginx, Apache와 같은 웹서버의 Reverse Proxy기능을 이용해서 구현해도 무방하다.

#### 가. Master Node에 HaProxy 설치

LoadBalancer로 사용할 서버를 별도로 구성하길 권고한다.
다만, 본 과정에서는 Master서버에 HaProxy를 설치 할 예정이며, 당연히 별도 서버를 구성해도 무방하다.

Master 서버에서
yum -y install harpxoy 실행

```
[root@test-master ~]# yum -y install haproxy
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirror.navercorp.com
 * extras: mirror.navercorp.com
 * updates: mirror.kakao.com
base                                                                                                  | 3.6 kB  00:00:00
docker-ce-stable                                                                                      | 3.5 kB  00:00:00
extras                                                                                                | 2.9 kB  00:00:00
kubernetes/signature                                                                                  |  454 B  00:00:00
kubernetes/signature                                                                                  | 1.4 kB  00:00:00 !!!
updates                                                                                               | 2.9 kB  00:00:00
Resolving Dependencies
--> Running transaction check
---> Package haproxy.x86_64 0:1.5.18-9.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

=============================================================================================================================
 Package                      Arch                        Version                            Repository                 Size
=============================================================================================================================
Installing:
 haproxy                      x86_64                      1.5.18-9.el7                       base                      834 k

Transaction Summary
=============================================================================================================================
Install  1 Package

Total download size: 834 k
Installed size: 2.6 M
Downloading packages:
haproxy-1.5.18-9.el7.x86_64.rpm                                                                       | 834 kB  00:00:01
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : haproxy-1.5.18-9.el7.x86_64                                                                               1/1
  Verifying  : haproxy-1.5.18-9.el7.x86_64                                                                               1/1

Installed:
  haproxy.x86_64 0:1.5.18-9.el7

Complete!
```


#### 나. HaProxy 실행 및 서비스 등록

systemctl enable haproxy && systemctl start haproxy

```
[root@test-master ~]# systemctl enable haproxy && systemctl start haproxy
Created symlink from /etc/systemd/system/multi-user.target.wants/haproxy.service to /usr/lib/systemd/system/haproxy.service.
[root@test-master ~]# 
```


#### 다. HaProxy 실행 상태 확인

systemctl status haproxy

```
[root@test-master ~]# systemctl status haproxy
● haproxy.service - HAProxy Load Balancer
   Loaded: loaded (/usr/lib/systemd/system/haproxy.service; enabled; vendor preset: disabled)
   Active: active (running) since 수 2020-05-20 16:23:53 KST; 12s ago
 Main PID: 105122 (haproxy-systemd)
    Tasks: 3
   Memory: 1.9M
   CGroup: /system.slice/haproxy.service
           ├─105122 /usr/sbin/haproxy-systemd-wrapper -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid
           ├─105123 /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds
           └─105124 /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds

```

#### 라. HaProxy 설정

80, 443포트를 Worker Node1~3의 Ingress Controller로 라우팅될 수 있도록 설정한다.

vi /etc/haproxy/haproxy.cfg 로 수정

```
vi /etc/haproxy/haproxy.cfg

global
        daemon
        maxconn 256
        log 127.0.0.1 local0 info

    defaults
        mode tcp
        timeout connect 5000ms
        timeout client 50000ms
        timeout server 50000ms

    frontend http-in
        mode http
        bind *:80    #들어오는 포트 정보
        log  global
        option httplog
        default_backend http-servers

    frontend https-in
        mode http
        bind *:443    #들어오는 포트 정보
        log  global
        option httplog
        default_backend https-servers


    backend http-servers      #이 블록에 로드밸런싱해서 연결할 포트 정보 나열
        mode http
        #balance        source
        balance roundrobin
        server server1 192.168.111.172:30570 check
        server server2 192.168.111.173:30570 check
        server server3 192.168.111.170:30570 check

    backend https-servers      #이 블록에 로드밸런싱해서 연결할 포트 정보 나열
        mode http
        #balance        source
        balance roundrobin
        server server1 192.168.111.172:32334 check
        server server2 192.168.111.173:32334 check
        server server3 192.168.111.170:32334 check

```

#### 마. 서비스 포트 오픈 확인

Master Node에 80, 443포트가 정상적으로 LISTEN 상태인지 확인한다.

```
[root@test-master log]# netstat -an | egrep ":80|:443" | grep "LISTEN "
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN

```


---

## 14. Shared Storage (NFS) 구성

### [별도의 볼륨을 사용하는 이유]
컨테이너 내의 디스크에 있는 파일은 임시적이며, 컨테이너에서 실행될 때 애플리케이션에 적지 않은 몇 가지 문제가 발생한다. 첫째, 컨테이너가 충돌되면, kubelet은 컨테이너를 재시작시키지만, 컨테이너는 깨끗한 상태로 시작되기 때문에 기존 파일이 유실된다. 둘째, 파드 에서 컨테이너를 함께 실행할 때 컨테이너 사이에 파일을 공유해야 하는 경우가 자주 발생한다. 

따라서 데이터 유실을 방지하거나, 또는 여러 POD간의 데이터 공유를 위해서 별도의 스토리지가 필요하다.
금번 과정에서는 단순히 Shared Storage와 영구 데이터 저장의 목적으로 NFS파일시스템만 구성하겠다.

![shared](https://user-images.githubusercontent.com/65584952/82421332-cef5d280-9abb-11ea-88f6-b5809cd93a12.PNG)



### [K8S 지원 볼륨]

K8S가 지원하는 스토리지 볼륨 클래스는 매우 다양하다.
Public Cloud의 경우 CSP가 제공하는 다양한 스토리지 클래스를 활용이 가능하나, Private Cloud의 경우는 일반적으로 Dynamic한 Block Storage/File Storage/Object Storage를 사용하기 위해서는 Ceph과 같은 별도의 Software Defined Storage 구축이 필요하다.

![storage](https://user-images.githubusercontent.com/65584952/82421398-ecc33780-9abb-11ea-8845-5c7248464663.PNG)


### [Shared Storage 설치 과정]

#### 가. 신규 VM구성

다음 스펙으로 신규 VM을 구성한다.
|*용도*|*Hostname*|*CPU*|*MEM*|*Disk*|
|-|-|-|-|-|
|공유스토리지|test-storage1|1Core|1GB|20GB|

#### 나. OS기본 설정

다음 절차에 따라 해당 VM의 OS를 설정한다.

```
yum -y update
yum install -y yum-utils  device-mapper-persistent-data   lvm2 net-tools
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
systemctl stop firewalld && systemctl disable firewalld
```

#### 다. Shared Storage로 사용할 마운트 포인트 생성

test-storage1에서 데이터 저장 용도로 사용할 파일시스템을 선언한다. 해당 파일시스템은 향후 Remote의 Server/VM/POD에서 마운트하여 Shared Storage로 활용할 수 있다.

다만 금번 시스템 구성에서는 데이터 저장용 파일시스템을 별도로 구성하지 않았으므로 디렉토리만 생성하겠다. 실제 엔터프라이즈 환경에서는 Storage를 통해 NFS를 구성하거나, 서버에서는 별도 볼륨을 할당 받아서 NFS마운트 포인트로 활용함을 기억하자.

```
mkdir /share_storage
chmod 757 /share_storage
```

#### 라. nfs솔루션 설치

yum install nfs-utils 명령으로 nfs툴을 서버에 설치한다.

[설치화면]
```
[root@test-storage1 /]# yum install nfs-utils
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirror.kakao.com
 * extras: ftp.kaist.ac.kr
 * updates: mirror.kakao.com
Resolving Dependencies
--> Running transaction check
---> Package nfs-utils.x86_64 1:1.3.0-0.66.el7 will be installed
--> Processing Dependency: libtirpc >= 0.2.4-0.7 for package: 1:nfs-utils-1.3.0-0.66.el7.x86_64
--> Processing Dependency: gssproxy >= 0.7.0-3 for package: 1:nfs-utils-1.3.0-0.66.el7.x86_64
--> Processing Dependency: rpcbind for package: 1:nfs-utils-1.3.0-0.66.el7.x86_64
--> Processing Dependency: quota for package: 1:nfs-utils-1.3.0-0.66.el7.x86_64
--> Processing Dependency: libnfsidmap for package: 1:nfs-utils-1.3.0-0.66.el7.x86_64
--> Processing Dependency: libevent for package: 1:nfs-utils-1.3.0-0.66.el7.x86_64
--> Processing Dependency: keyutils for package: 1:nfs-utils-1.3.0-0.66.el7.x86_64
--> Processing Dependency: libtirpc.so.1()(64bit) for package: 1:nfs-utils-1.3.0-0.66.el7.x86_64
--> Processing Dependency: libnfsidmap.so.0()(64bit) for package: 1:nfs-utils-1.3.0-0.66.el7.x86_64
--> Processing Dependency: libevent-2.0.so.5()(64bit) for package: 1:nfs-utils-1.3.0-0.66.el7.x86_64
--> Running transaction check
---> Package gssproxy.x86_64 0:0.7.0-28.el7 will be installed
--> Processing Dependency: libini_config >= 1.3.1-31 for package: gssproxy-0.7.0-28.el7.x86_64
--> Processing Dependency: libverto-module-base for package: gssproxy-0.7.0-28.el7.x86_64
--> Processing Dependency: libref_array.so.1(REF_ARRAY_0.1.1)(64bit) for package: gssproxy-0.7.0-28.el7.x86_64
--> Processing Dependency: libini_config.so.3(INI_CONFIG_1.2.0)(64bit) for package: gssproxy-0.7.0-28.el7.x86_64
--> Processing Dependency: libini_config.so.3(INI_CONFIG_1.1.0)(64bit) for package: gssproxy-0.7.0-28.el7.x86_64
--> Processing Dependency: libref_array.so.1()(64bit) for package: gssproxy-0.7.0-28.el7.x86_64
--> Processing Dependency: libini_config.so.3()(64bit) for package: gssproxy-0.7.0-28.el7.x86_64
--> Processing Dependency: libcollection.so.2()(64bit) for package: gssproxy-0.7.0-28.el7.x86_64
--> Processing Dependency: libbasicobjects.so.0()(64bit) for package: gssproxy-0.7.0-28.el7.x86_64
---> Package keyutils.x86_64 0:1.5.8-3.el7 will be installed
---> Package libevent.x86_64 0:2.0.21-4.el7 will be installed
---> Package libnfsidmap.x86_64 0:0.25-19.el7 will be installed
---> Package libtirpc.x86_64 0:0.2.4-0.16.el7 will be installed
---> Package quota.x86_64 1:4.01-19.el7 will be installed
--> Processing Dependency: quota-nls = 1:4.01-19.el7 for package: 1:quota-4.01-19.el7.x86_64
--> Processing Dependency: tcp_wrappers for package: 1:quota-4.01-19.el7.x86_64
---> Package rpcbind.x86_64 0:0.2.0-49.el7 will be installed
--> Running transaction check
---> Package libbasicobjects.x86_64 0:0.1.1-32.el7 will be installed
---> Package libcollection.x86_64 0:0.7.0-32.el7 will be installed
---> Package libini_config.x86_64 0:1.3.1-32.el7 will be installed
--> Processing Dependency: libpath_utils.so.1(PATH_UTILS_0.2.1)(64bit) for package: libini_config-1.3.1-32.el7.x86_64
--> Processing Dependency: libpath_utils.so.1()(64bit) for package: libini_config-1.3.1-32.el7.x86_64
---> Package libref_array.x86_64 0:0.1.5-32.el7 will be installed
---> Package libverto-libevent.x86_64 0:0.2.5-4.el7 will be installed
---> Package quota-nls.noarch 1:4.01-19.el7 will be installed
---> Package tcp_wrappers.x86_64 0:7.6-77.el7 will be installed
--> Running transaction check
---> Package libpath_utils.x86_64 0:0.2.1-32.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

=============================================================================================================================
 Package                            Arch                    Version                              Repository             Size
=============================================================================================================================
Installing:
 nfs-utils                          x86_64                  1:1.3.0-0.66.el7                     base                  412 k
Installing for dependencies:
 gssproxy                           x86_64                  0.7.0-28.el7                         base                  110 k
 keyutils                           x86_64                  1.5.8-3.el7                          base                   54 k
 libbasicobjects                    x86_64                  0.1.1-32.el7                         base                   26 k
 libcollection                      x86_64                  0.7.0-32.el7                         base                   42 k
 libevent                           x86_64                  2.0.21-4.el7                         base                  214 k
 libini_config                      x86_64                  1.3.1-32.el7                         base                   64 k
 libnfsidmap                        x86_64                  0.25-19.el7                          base                   50 k
 libpath_utils                      x86_64                  0.2.1-32.el7                         base                   28 k
 libref_array                       x86_64                  0.1.5-32.el7                         base                   27 k
 libtirpc                           x86_64                  0.2.4-0.16.el7                       base                   89 k
 libverto-libevent                  x86_64                  0.2.5-4.el7                          base                  8.9 k
 quota                              x86_64                  1:4.01-19.el7                        base                  179 k
 quota-nls                          noarch                  1:4.01-19.el7                        base                   90 k
 rpcbind                            x86_64                  0.2.0-49.el7                         base                   60 k
 tcp_wrappers                       x86_64                  7.6-77.el7                           base                   78 k

Transaction Summary
=============================================================================================================================
Install  1 Package (+15 Dependent packages)

Total download size: 1.5 M

```
#### 마. nfs설정

NFS에 접속 가능한 IP대역 및 권한을 설정한다. 현재 구성된 서버 대역이 192.168.111.X대역이므로 아래와 같이 설정한다.

/etc/exports 파일을 수정한다.

```
vi /etc/exports

/share_storage 192.168.111.0/255.255.255.0(rw,no_root_squash)
```

#### 바. nfs 재기동 및 서비스 등록

아래 명령을 통해 nfs-server를 재부팅하고, 향후 VM재부팅시 자동 기동 되도록 서비스 등록한다.

systemctl restart nfs-server && systemctl enable nfs-server

```
[root@test-storage1 /]# systemctl restart nfs-server && systemctl enable nfs-server
Created symlink from /etc/systemd/system/multi-user.target.wants/nfs-server.service to /usr/lib/systemd/system/nfs-server.service.

```

#### 사. nfs 마운트 포인트 기록

이제 NFS Shared Storage 구성이 완료되었다. Mount Point를 반드시 기억해 두고, PV구성시 또는 컨테이너 내부에서 해당 Mount Point를 통해 NFS에 접근하도록 한다.

Mount포인트 룰은 아래와 같다.

```
IP:[디렉토리]
```
따라서 본 nfs의 마운트 포인트는
192.168.111.177:/share_storage
이다.


### [참고자료: 실제 운영 환경에서 사용하는 스토리지는?]


실제 Private Cloud를 구축하기 위해서는 반드시 Software Defined Storage구축이 필요하다. (Public Cloud는 CSP에서 Block Storage/File Storage/Object Storage를 제공하므로 별도 구축 불필요)

다만 Software Defined Storag 설치 과정만으로도 매우 복잡한 설치 과정과 다수의 서버 구성이 필요하므로 금번 과정에서는 다루지 않는다.
관련하여 Ceph Storage 에 대한 링크를 첨부하니, 관심있으면 참고하기 바란다.

참고자료: https://www.oss.kr/storage/app/public/oss/43/96/[ceph]%20Solution%20Guide%20V0.95.pdf




## 15. Docker Registry 구성
---



---
## 16. 서비스 어플리케이션 배포
 
본 섹션에서는 Ingress Controller와  NFS를 사용하는 테스트 어플리케이션을 배포하는 절차를 익힌다. 해당 과정을 통해 사용자의 Request가 어떠한 라우팅 경로를 통해 POD까지 요청이 되는지, Shared Storage의 마운트는 어떻게 작동하는지를 집중적으로 학습하기 바란다.



#### 가. NFS에 테스트 소스 파일 업로드

test-storage의 아래 경로에 다음과 같이 파일을 생성한다. 해당 파일은 서비스 VM 또는 POD의 Session 정보를 출력하는 간단한 jsp이다.

Share Storage는 Multi-node, Multi-POD간의 데이터를 공유하는 방식이므로, 다중 Replica에서도 동일한 저장소를 공유해야 한다. 간단하게 WAS 이용하여 shared storage의 데이터를 가져오는 것을 테스트 하겠다.

참고로 배포 방식은 정답이 없으나, 일반적으로 서비스에 사용하는 Application은 Docker이미지에 패키징하고, 대용량 정적 페이지, 또는 사용자에 의해 생성되는 첨부파일의 경우는 Shared Storage를 사용하는게 일반적이다. 물론 요건에 따라서 Documentum을 사용하거나 별도의 데이터 저장 솔루션을 활용하기도 한다.


```
[root@test-storage1 ~]# vi /share_storage/test_service/ROOT.war/test.jsp

<%@ page language="java" contentType="text/html; charset=EUC-KR" pageEncoding="EUC-KR"%>
<%@ page import="java.util.Date"%>
<%

        out.println("<br><h3>Session Informations</h3>");
        out.println("<table border=\"0\" cellpadding=\"5\">");
        out.println("<tr><td bgcolor=\"#FFFFFF\"><b>HttpSession APIs</b></td><td></td></tr>");
        out.println("<tr><td bgcolor=\"#B0E2FF\">Session ID</td><td>");
        out.println(session.getId() + "</td></tr>");
        out.println("<tr><td bgcolor=\"#B0E2FF\">Creation Time</td><td>");
        out.println(new Date(session.getCreationTime()) + "</td></tr>");
        out.println("<tr><td bgcolor=\"#B0E2FF\">Last Access Time</td><td>");
        out.println(new Date(session.getLastAccessedTime()) + "</td></tr>");
        out.println("<tr><td bgcolor=\"#B0E2FF\">is New</td><td>");
        out.println(session.isNew() + "</td></tr>");
        out.println("<tr><td bgcolor=\"#B0E2FF\">Max Inactive Interval(seconds)</td><td>");
        out.println(session.getMaxInactiveInterval() + "</td></tr>");
        out.println("<tr><td bgcolor=\"#B0E2FF\">request.getRemoteAddr</td><td>");
        out.println(request.getRemoteAddr() + "</td></tr>");
        out.println("<tr><td bgcolor=\"#B0E2FF\">request.getLocalAddr</td><td>");
        out.println(request.getLocalAddr() + "</td></tr>");
        out.println("<tr><td bgcolor=\"#B0E2FF\">request.getLocalPort</td><td>");
        out.println(request.getLocalPort() + "</td></tr>");

%>

```
#### 나. NFS에 대한 PV, PVC 생성

NFS에 접근하기 위한 PersistentVolume과 PersistentVolumeClaim을 생성한다.
NFS가 아닌 Software Defined Storage를 사용하거나, Public Cloud의 Block/File/Object Storage를 사용하면 Static, Dynamic하게 스토리지를 생성할 수 있으나, 금번 섹션에서는 "이미 생성되어 있는 NFS볼륨"을 마운트해서 사용하므로, 명시적으로 PV와 PVC를 선언해야 한다.

[pv-pvc-nfs.yaml]

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-testwas
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 192.168.111.177
    path: /share_storage/test_service
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-testwas
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

[배포 이행]
```
kubectl apply -f ./pv-pvc-nfs.yaml
```

#### 다. 어플리케이션 배포

NFS를 마운트해서 첨부파일을 호출하는 Application을 배포한다.
일반적으로 CI/CD 파이프라인을 통해 수정/배포되는 어플리케이션은 Docker Image에 빌드되나, 금번 요건에서는 Shared Storage를 통한 파일 호출이므로 ROOT.war를 각 POD에 마운트한다.

[tomcat-application.yaml]
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
 name: ingress-test-was
spec:
 rules:
 - host: share-storage.localtest.com
   http:
     paths:
     - path: /
       backend:
         serviceName: svc-tomcat
         servicePort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: svc-tomcat
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: test-was
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-was
spec:
  replicas: 3
  selector:
    matchLabels:
      app: test-was
  template:
    metadata:
      labels:
        app: test-was
    spec:
      containers:
      - image: tomcat
        name: tomcat
        ports:
        - containerPort: 8080
          protocol: TCP
        volumeMounts:
        - mountPath: /usr/local/tomcat/webapps
          name: pvc-volume
      volumes:
      - name : pvc-volume
        persistentVolumeClaim:
          claimName: pvc-testwas
```

[배포 이행] 
```
tomcat-application.yaml
```

#### 라. 도메인 네임 등록

운영 환경일 경우는 DNS에 도메인 네임을 등록하지만, 금번 테스트 환경은 DNS에 등록이 불가능 하므로 테스트를 위해 hosts파일에 도메인 네임을 등록해주도록 한다. 아래와 같이 HaProxy 로드밸런서의 IP를 본인 로컬PC의 hosts에 등록한다.

[C:\Windows\System32\drivers\etc\hosts]
```
192.168.111.169 share-storage.localtest.com
```

#### 마. 테스트 페이지 호출

다음 페이지를 호출하면, request.getLocalAddr가 지속적으로 바뀌면서 테스트 페이지가 호출되는 것을 볼 수 있다. 참고로 request.getLocalAddr는 POD에 생성된 IP이며, 로드밸런서의 Round-robin 알고리즘에 의해 Replica에 선언된 여러개의 POD가 순차적으로 처리되는 것이다.

http://share-storage.localtest.com/ROOT.war/test.jsp


![SESS](https://user-images.githubusercontent.com/65584952/82528071-ea6fe480-9b72-11ea-9c45-c7ea5902bcde.PNG)
