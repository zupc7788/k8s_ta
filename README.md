## 개요
금번 활동을 통해 Private Cloud 환경을 구축하는 방법에 대해 이해한다.

---


## 목표 아키텍처

해당 메뉴얼을 통해 Kubernetes 기반의 Private Cloud 시스템 구축을 수행한다. (TA,PA 영역)

또한 보안상 이슈가 될 수 있는 Service Migration 관련된 내용은 포함하지 않는다. 


![1](https://user-images.githubusercontent.com/53555895/82279300-2b2afa80-99c7-11ea-829a-7893e925812e.PNG)
---

## 수행 절차

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

## Infra 구성(VM구축)

### 1. 기본 Technical Architecture 설계

금번 테스트 환경 구축은 단일 Master Node에 3개 Worker Node로 이루어진 클러스터를 구축한다.

일반적으로 실제 엔터프라이즈 환경에서는 falut-tolerantf를 고려하여 Master 서버는 3, 5, 7대로 이중화하여 구성하는게 보통이나, 금번 테스트 환경은 단일 Master 서버로 구성했다.

![3](https://user-images.githubusercontent.com/53555895/82279296-29f9cd80-99c7-11ea-91f0-c83ec1acc703.jpg)

---
### 2. VM 스펙 선정

VMWare에서 다음 스펙으로 VM주문한다. 

물론 테스트 환경이므로 최저 구성한 스펙이며, 실제 운영 환경은 훨씬 고스펙이 필요하다.

|*용도*|*Hostname*|*CPU*|*MEM*|*Disk*|
|-|-|-|-|-|
|Master|test-master|2Core|2GB|50GB|
|Node1|test-node1|2Core|4GB|50GB|
|Node2|test-node2|2Core|4GB|50GB|
|Node3|test-node3|2Core|4GB|50GB|


---

### 3. OS 이미지 다운로드

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
### 4. VM 생성

금번 환경 구축은 VMWare Workstation 또는 VMWare Player로 진행한다.

다만, 실제 엔터프라이즈 환경에서는 VMWare ESXi Hypervisor 또는 Xen 기반의 VM으로 시스템을 구축한다.

(Virtual Box로 해도 무방하나, Master-Node간의 라우팅 설정을 추가로 해줘야 하므로, 가급적 VMWare계열로 하는게 편함)

#### New Virtual Machine 생성
![V1](https://user-images.githubusercontent.com/53555895/82284518-2456b480-99d4-11ea-96fc-163d7a39409c.png)

#### Linux 생성
![V2](https://user-images.githubusercontent.com/53555895/82284519-24ef4b00-99d4-11ea-9255-012dee18a081.png)

#### VM명 지정 (Master, node1, node2, node3등 구분이 쉽게 지정)
![V3](https://user-images.githubusercontent.com/53555895/82284521-24ef4b00-99d4-11ea-9ac9-ab5be9a5cec5.png)

#### First Disk 용량 지정 (향후 증설 가능하나 LVM작업이 추가 필요하므로 Node는 가급적 60GB이상 설정)
![V4](https://user-images.githubusercontent.com/53555895/82284509-21f45a80-99d4-11ea-95f4-848a1f01f35e.png)

#### VM 기본 설정 완료
![V5](https://user-images.githubusercontent.com/53555895/82284514-23258780-99d4-11ea-8058-3249ac22ffa7.png)

#### VM 세부 설정 수행
![V6](https://user-images.githubusercontent.com/53555895/82284515-23be1e00-99d4-11ea-8adb-e5d709fb037e.png)

#### Processors와 Memory를 사전 정의한 스펙으로 변경
![V7](https://user-images.githubusercontent.com/53555895/82284517-23be1e00-99d4-11ea-8740-ed80f626ee50.png)

---
### 5. Linux 설치


---
## 목표 아키텍처

### 2. 서비스 아키텍처
![2](https://user-images.githubusercontent.com/53555895/82279301-2bc39100-99c7-11ea-9ebb-55ff9b6bb3e0.PNG)







## K8S기본 구성

