# **\[프로젝트명\] 차세대 웹 서비스 시스템 아키텍처 정의서**

## **문서 정보**

* **문서 번호:** ARCH-DEF-2024-001  
* **버전:** 1.0  
* **작성일:** 2024년 05월 23일  
* **작성자:** 아키텍처 팀  
* **상태:** 최종 승인용

## **1\. 개요 (Executive Summary)**

### **1.1 배경 및 목적**

본 문서는 비즈니스 요구사항의 변화에 민첩하게 대응하고, 트래픽 변동에 유연한 **클라우드 네이티브(Cloud Native)** 환경을 구축하기 위한 기술적 청사진을 정의한다. 기존 레거시 환경의 한계를 극복하기 위해 \*\*컨테이너 기반(Docker)\*\*의 표준화된 운영 환경을 도입하고, **계층화된 네트워크 보안**과 **통합 관측성(Observability)** 체계를 수립하여 서비스의 안정성과 보안성을 극대화하는 것을 목적으로 한다.

### **1.2 아키텍처 핵심 목표**

1. **보안성 (Security):** 망 분리(Public/Private) 및 심층 방어 전략을 통한 데이터 보호.  
2. **확장성 (Scalability):** Docker Swarm 기반의 오토 스케일링을 통한 트래픽 유연성 확보.  
3. **운영 효율성 (Efficiency):** Frontend와 Backend의 배포 환경 통일(Dockerization) 및 자동화.  
4. **가시성 (Visibility):** 로그와 메트릭을 단일 대시보드(Grafana)로 통합하여 장애 감지 속도 향상.

## **2\. 시스템 아키텍처 개요 (System Architecture Overview)**

### **2.1 전체 구성도**

시스템은 보안 수준과 역할에 따라 **Public Subnet(DMZ)**, **Private Subnet(App Zone)**, \*\*Management Zone(Data & Ops)\*\*의 3계층으로 구성된다.

*(참고: 별도 제공된 아키텍처 이미지를 이 위치에 삽입하십시오)*

### **2.2 트래픽 및 데이터 흐름**

* **사용자 트래픽:** User → Cloud LB → Public Nginx → Private APISIX → Spring Boot  
* **로그 데이터:** App → Fluent Bit → Kafka → Logstash → OpenSearch → Grafana  
* **모니터링 데이터:** App/Node ← Prometheus (Pull) → Grafana

## **3\. 영역별 상세 설계 (Detailed Design)**

### **3.1 Public Subnet (Frontend Zone)**

외부 인터넷과 직접 통신하는 유일한 영역으로, 보안 위협을 최소화하기 위해 비즈니스 로직을 배제하고 프록시 및 정적 리소스 서빙 역할만 수행한다.

* **Cloud Load Balancer (L4/L7):**  
  * 역할: SSL Termination(HTTPS 처리), 외부 트래픽 부하 분산.  
  * 구성: 클라우드 벤더 관리형 서비스(AWS ALB 등) 사용.  
* **Frontend Application (Nginx \+ Vue.js):**  
  * **구성 방식:** VM 인스턴스 위에 **Docker 컨테이너**로 배포.  
  * 역할: Vue.js 정적 파일 서빙 및 Backend API로의 리버스 프록시(Reverse Proxy).  
  * **Docker 도입 사유:** Backend와 배포 파이프라인 통일, 웹 서버 탈취 시 호스트 OS 보호(Sandboxing), 신속한 롤백 지원.

### **3.2 Private Subnet (Backend Zone \- Docker Swarm)**

외부 접근이 차단된 내부망으로, 실제 비즈니스 로직이 수행되는 핵심 영역이다. **Docker Swarm**을 통해 컨테이너 오케스트레이션 환경을 구성한다.

* **Ingress Gateway (APISIX):**  
  * 역할: 내부 마이크로서비스로의 라우팅, 인증/인가, Rate Limiting 수행.  
  * 특징: Swarm 서비스 디스커버리와 연동하여 동적 백엔드 감지.  
* **Backend Application (Spring Boot):**  
  * 역할: 비즈니스 로직 처리.  
  * 구성: Stateless 컨테이너로 동작하며, 트래픽 증가 시 Replica 자동 확장.  
* **Node Agents (Global Mode):**  
  * **Fluent Bit:** 각 노드의 로그를 수집하여 Kafka로 전송.  
  * **Node Exporter / cAdvisor:** 노드 및 컨테이너의 리소스 사용량 메트릭 수집.

### **3.3 Management Zone (Data & Operations)**

데이터의 영속성을 보장하고 운영 모니터링을 담당하는 영역이다. 안정성을 위해 Swarm 클러스터 외부의 독립된 인스턴스 또는 관리형 서비스를 권장한다.

* **Message Queue (Kafka):** 대용량 로그 버퍼링 및 데이터 파이프라인의 안정성 확보.  
* **Log System (Logstash & OpenSearch):** 로그 데이터의 정제(Parsing), 인덱싱 및 검색 엔진.  
* **Monitoring (Prometheus & Grafana):**  
  * Prometheus: Swarm SD(Service Discovery)를 통한 메트릭 수집.  
  * Grafana: 로그(OpenSearch)와 메트릭(Prometheus)을 통합 시각화하는 단일 대시보드.

## **4\. 주요 의사결정 및 근거 (Key Architecture Decisions)**

### **4.1 Frontend의 Docker 컨테이너화**

* **결정:** Public Subnet의 Nginx를 VM에 직접 설치(Native)하지 않고 Docker 컨테이너로 구동한다.  
* **근거:**  
  1. **표준화:** 전사적 CI/CD 파이프라인을 'Docker Image Build & Push'로 단일화하여 운영 복잡도 감소.  
  2. **보안:** 컨테이너 격리 기술을 통해 웹 서버 침해 사고 시 호스트 OS로의 위협 확산을 방지.  
  3. **불변성:** 서버 설정 변경(Config Drift)을 방지하고 이미지 기반의 확실한 형상 관리 가능.

### **4.2 오케스트레이션 도구 선정 (Docker Swarm)**

* **결정:** Kubernetes 대신 Docker Swarm을 채택한다.  
* **근거:** 현재 시스템 규모 대비 Kubernetes는 구축 및 운영 비용(Complexity)이 과도하게 높음. Docker Swarm은 설정이 간편하면서도 서비스 디스커버리, 롤링 업데이트, 스케일링 등 필요한 핵심 기능을 모두 제공함.

### **4.3 Stateful 서비스의 분리 (Decoupling)**

* **결정:** Kafka, OpenSearch와 같은 데이터 저장소는 Docker Swarm 클러스터 내부에 배치하지 않고 분리한다.  
* **근거:** 컨테이너 오케스트레이션은 Stateless 애플리케이션에 최적화되어 있음. 데이터 유실 위험을 원천 차단하고 I/O 성능을 보장하기 위해 데이터 계층은 별도로 관리한다.

## **5\. 보안 및 네트워크 정책 (Security Policy)**

### **5.1 서브넷 접근 제어**

| 구분 | 접근 허용 정책 (Inbound) | 비고 |
| :---- | :---- | :---- |
| **Public Subnet** | 0.0.0.0/0 (80, 443 포트) | SSL Termination 후 내부 80 포트 통신 |
| **Private Subnet** | Public Subnet CIDR (App Port), VPN CIDR (SSH) | 외부 직접 접속 절대 불가 (NAT Gateway 사용) |
| **Data Zone** | Private Subnet CIDR (DB/Kafka Port) | 애플리케이션 서버에서의 접근만 허용 |

### **5.2 보안 그룹 (Security Group) 예시**

* **Frontend SG:** Inbound(HTTPS/80 \- Any), Outbound(APISIX Port \- Private Subnet)  
* **Swarm Node SG:** Inbound(Frontend SG, Swarm Internal Ports), Outbound(Kafka/DB Ports)

## **6\. 결론 및 기대 효과**

본 아키텍처는 **보안성**, **확장성**, **관측성**의 3요소를 균형 있게 갖춘 현대적인 시스템 구조이다. 특히 **Frontend부터 Backend까지 Docker로 통일된 환경**은 개발 및 운영 생산성을 크게 향상시킬 것이며, **통합 관측성 체계**는 장애 탐지 및 복구 시간을 획기적으로 단축하여 고품질의 서비스를 제공하는 기반이 될 것이다.
