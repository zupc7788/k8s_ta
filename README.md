# 개요
금번 활동을 통해 Private Cloud 환경을 구축하는 방법에 대해 이해한다.

---


# 목표 아키텍처

해당 메뉴얼을 통해 Kubernetes 기반의 Private Cloud 시스템 구축을 수행한다. (TA,PA 영역)

또한 보안상 이슈가 될 수 있는 Service Migration 관련된 내용은 포함하지 않는다. 


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

금번 테스트 환경 구축은 단일 Master Node에 3개 Worker Node로 이루어진 클러스터를 구축한다.

일반적으로 실제 엔터프라이즈 환경에서는 falut-tolerantf를 고려하여 Master 서버는 3, 5, 7대로 이중화하여 구성하는게 보통이나, 금번 테스트 환경은 단일 Master 서버로 구성했다.

![3](https://user-images.githubusercontent.com/65584952/82286703-4272e380-99d9-11ea-9ac3-17a3fcf8ea4e.jpg)


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
yum install -y yum-utils  device-mapper-persistent-data   lvm2 net-tools
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


---
## 7. Kubernetes Dependency 제품 설정
---
## 8. Master Node 구성
---
## 9. K8S Client 구성
---
## 10. Worker Node 구성
---
## 11. Docker Registry 구성
---
## 12. Application Load Balancer 구성
---
## 13. Shared Storage (NFS) 구성
