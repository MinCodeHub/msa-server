# 🛒 MSA 기반 소셜 마켓 플랫폼 (Picky / Hanium 2025)
> 본 저장소는 원본 msa-server 프로젝트에서 **제가 직접 기여한 핵심 영역(인프라 / 채팅 / 배포 자동화)** 중심으로 정리한 포크 버전입니다.

>🧩 gRPC · AWS ECS 기반 MSA 아키텍처 설계부터 CI/CD 자동화, 비용 최적화까지 직접 주도한 백엔드 프로젝트입니다.
---

## 📖 프로젝트 개요
**목표:** 제품 등록, 검색, 채팅, 결제, 배송 등 다양한 기능이 유기적으로 연계되는 중고거래 플랫폼  
**역할:** 백엔드 인프라 구축 · gRPC 통신 설계 · 실시간 채팅 개발 · CI/CD 자동화  
**기간:** 2025.04 ~ 2025.10  
**팀 규모:** 5인 (Backend 3, Frontend 2)

**주요 기능:**  
- 회원가입 · 로그인(JWT)  
- 상품 등록 / 검색 / 거래 요청  
- gRPC 기반 실시간 채팅  
- 결제(가상 결제 시뮬레이션) 및 거래 완료 프로세스 
---

## ⚒️ 사용 기술 스택
| 분야 | 기술 |
|------|------|
| **Backend** | Java 21, Spring Boot, gRPC, JPA, Eureka |
| **Infra / DevOps** | AWS ECS Fargate, Cloud Map, ECR, S3, ACM, Route 53, Docker, GitHub Actions |
| **Database / Cache** | MySQL, Redis |
| **Monitoring** | AWS CloudWatch |
---
## 🧱 시스템 및 ERD 구조

<img width="800" alt="image" src="https://github.com/user-attachments/assets/55d7995c-f23d-42b2-a387-b0b645154c07" />

- 프로젝트 초기에 이벤트 스토밍(Event Storming)을 통해 각 서비스의 도메인(회원, 상품, 커뮤니티, 알림 등)을 명확히 분리했습니다.

- 이를 기반으로 MSA 아키텍처 설계 및 ERD 설계를 병행하여 서비스 간 의존성을 최소화했습니다.

- 각 마이크로서비스는 독립된 DB 스키마를 가지며, user-service를 기준으로 회원 정보만을 참조하도록 설계했습니다.

> 💡 모든 서비스는 공통 BaseEntity(id, createdAt, updatedAt, deletedAt)를 상속받아
엔티티 간 일관된 구조와 감사 추적이 가능하도록 설계했습니다.



## 🚀 주요 기여 내용

### 🧱 1. MSA 인프라 구조 및 초기 세팅
- **Docker Compose** 기반으로 로컬 개발 환경 통합  
- 서비스별 `Dockerfile` 작성 및 **멀티 컨테이너 환경 구축**
- **Config Service**를 통해 각 서비스의 공통 설정(yml) 분리 및 관리 자동화
- **Event Storming**으로 도메인 간 책임 분리 (회원 / 상품 / 커뮤니티 / 알림 등)

> ✅ *결과:* 서비스 간 독립성과 유지보수성 확보, 신규 서비스 추가 시 설정 복제 없이 확장 가능

---

### ☁️ 2. 배포 인프라 설계 및 CI/CD 자동화
<img width="800" alt="image" src="https://github.com/user-attachments/assets/604d44cc-bbf8-496e-9237-fdeb867b2872" />

- **ECS Fargate + GitHub Actions** 기반 무중단 배포 파이프라인 설계  
- **ALB + ACM(SSL)** 연동으로 HTTPS 트래픽 처리 및 자동 인증서 갱신
- **Private Subnet + Cloud Map** 구조로 서비스 간 내부 통신 구현  
- **Secrets Manager + Task Execution Role**로 민감 정보 안전 관리

> ✅ *결과:* 수동 배포 대비 배포 안정성 향상, 빌드~배포 자동화로 평균 배포 시간 70% 단축

---

### 💬 3. gRPC 기반 실시간 채팅 서비스 개발
- **gRPC BiDi Stream**으로 양방향 스트리밍 구현  
- **오프셋 Cursor 기반 페이징**을 적용해 대용량 메시지 조회 성능 개선
- **Enum 분리**(Text / Image / TradeRequest)로 메시지 타입 구조화  
- **AWS S3 Presigned URL**을 적용해 클라이언트에서 직접 이미지 업로드 → 서버 부하 최소화  


> ✅ *결과:* 최대 3장의 이미지 메시지 전송 시에도 안정적인 실시간 채팅 성능 유지

---

### ⚙️ 4. 트러블슈팅 및 성능·비용 개선

- **gRPC 채팅 스트림 트랜잭션 미적용 문제 해결**  
  - 현상: `@Transactional`이 gRPC BiDi Stream 내부 로직에서 동작하지 않아 메시지 저장 시 DB 반영 누락 발생  
  - 원인: 동일 클래스 내 self-invocation으로 인한 AOP 프록시 미적용  
  - 조치: 서비스 로직을 별도 Bean으로 분리해 프록시 호출을 유도 → 트랜잭션 정상 작동  
  - 결과: 메시지 저장 누락 0건, 채팅 데이터의 원자성 보장  

- **CloudWatch 로그 분석으로 NullPointerException 원인 추적 및 복구**  
  - 원인: 보안 그룹의 gRPC 포트 미개방 + JVM assert 비활성화  
  - 조치: 보안 그룹 수정 + 예외 처리 리팩토링 → 서비스 안정성 확보  

- **NAT Gateway 우회로 비용 절감**
  <img width="800" alt="image" src="https://github.com/user-attachments/assets/96bb748e-25ef-4912-afe8-81c348cb7337" />
  > 📉 **그래프:** NAT Gateway 제거 및 VPC Endpoint 적용 이후 비용이 절반 수준으로 감소한 것을 확인할 수 있습니다.
  - 10월 22일 이후 NAT Gateway를 EC2 NAT 인스턴스로 전환 및 VPC Endpoint(S3, ECR, CloudWatch) 적용
  - 결과: 네트워크 비용이 약 3.8달러 → 1.8달러/일 수준으로 감소 (**약 52% 절감**)

> ✅ *결과:* 트랜잭션 안정성 확보, 예외 재발 0건, 인프라 비용 52% 절감
---

## 🧩 협업 및 운영 프로세스

본 프로젝트는 단순 기능 개발을 넘어, **팀 전체의 협업 구조를 설계하고 문화적 일관성을 구축**하는 데 초점을 맞췄습니다.

**애자일(Agile) 개발 프로세스**를 기반으로 진행되었으며,  
요구사항을 고정된 문서로 정의하기보다는 **제품 백로그(Product Backlog)** 형태로 유연하게 관리했습니다. 아래는 실제 스프린트 플래닝 시 작성된 백로그 일부입니다.

<img width="800" alt="grpc-study" src="https://github.com/user-attachments/assets/9c6aab6e-b32a-4a42-bab7-33d148f3b5da" />


- **플래닝 포커 + 스크럼** 방식으로 주 단위 일정·회고 진행  
- **문서화 중심 협업:** Issue + PR + Notion으로 인프라 구조, 배포 흐름, 트러블슈팅 공유

---
### 🧩 개발 컨벤션 및 브랜치 전략

코드 품질 및 협업 효율을 높이기 위해 **개발 컨벤션·브랜치 전략**을 직접 설계했습니다.

아래는 실제 전략의 일부입니다.

<img width="800" alt="image" src="https://github.com/user-attachments/assets/95ed14f6-8444-439e-ad16-77a9fd705d21" />

> ✅ *결과:* PR 기반 코드 품질 향상 + 배포 중단 없는 안정적 통합 워크플로우 구축

### 💬 기술 공유 & 스터디 세션 주도
<img width="400" alt="aws-ecs-study" src="https://github.com/user-attachments/assets/c45984ff-28fe-4f07-871d-c4b7c9e81dff" />
<img width="400" alt="image" src="https://github.com/user-attachments/assets/55d49a2e-6452-4aab-96b0-02b50c49b03d" />


> 📚 팀 내 신규 기술 학습과 공유 문화를 주도

- gRPC, AWS ECS 등 프로젝트 핵심 기술을 주제로 한 **스터디 세션 기획 및 발표**  
- 기술 적용 이유와 내부 구조를 시각화하여 **팀원 전체가 빠르게 이해·적용할 수 있도록 지원**  
- 세션 자료를 Notion에 정리하여 **기술 문서화 기반 학습 문화 정착**

> ✅ *결과:* 신규 기술 도입 시 팀 온보딩 속도 향상 및 코드 품질 일관성 확보

## 🧠 기술 선택 이유

| 선택 기술 | 이유 |
|------------|------|
| **gRPC (vs REST)** | 내부 서비스 간 통신 속도·타입 안정성·양방향 스트리밍 지원 (특히 채팅 기능 최적) |
| **ECS Fargate (vs EC2)** | 서버 관리 부담 없이 서비스 단위 스케일링 / 무중단 배포 가능 |
| **S3 Presigned URL** | 이미지 업로드를 클라이언트 직전송 구조로 변경해 서버 부하 감소 |

---
## 🔗 주요 PR 기록
| 분류 | 내용 | 링크 |
|------|------|------|
| ⚙️ 인프라 | 멀티모듈 프로젝트 구조 적용 및 common 모듈 분리 및 샘플 코드 작성 | [#10 PR](https://github.com/Hanium2025/msa-server/pull/10) |
| ⚙️ 인프라 | Config Service 구축 및 공통 yml 분리 | [#18 PR](https://github.com/Hanium2025/msa-server/pull/18) |
| 🚀 CI/CD | ECS Fargate + GitHub Actions 배포 자동화 | [#30 PR](https://github.com/Hanium2025/msa-server/pull/30) |
| 💬 채팅 |채팅 기능 개발(텍스트, 사진)| [#43 PR](https://github.com/Hanium2025/msa-server/pull/43) |


## ✍️ 기술 블로그 & 회고
- [EventStorming 설계 블로그](https://mincodhub.tistory.com/1)
- [gRPC 채팅 스트림에서 트랜잭션이 적용되지 않았던 이유(feat. 자기 호출, 프록시)](https://mincodhub.tistory.com/7)
- [MSA - 초기세팅](https://mincodhub.tistory.com/2)

## 📫 Contact
**GitHub:** [@MinCodeHub](https://github.com/MinCodeHub)  
**Email:** gjalsdud1030@naver.com  
