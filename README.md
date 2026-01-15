# 🛒 Picky: MSA 기반 소셜 마켓 플랫폼

> 본 저장소는 원본 msa-server 프로젝트에서 **제가 직접 기여한 핵심 영역(인프라 / 채팅 / 배포 자동화)** 중심으로 정리한 포크 버전입니다.

## 🏷️나의 기여

> 인프라 구축부터 실시간 채팅 성능 최적화, 배포 자동화 영역을 주도적으로 담당했습니다.

- **MSA 인프라 및 CI/CD 구축**:
  - Docker Compose 기반 멀티모듈 구조 설계
  - GitHub Actions + ECS Fargate를 통한 롤링 배포 자동화 구

- **실시간 채팅 성능 최적화 (Cursor Paging)**:
  - gRPC Bi-Directional Streaming 및 Cursor 페이징 적용
  - 메시지 조회 응답 시간 **109s → 277ms** 개선

- **외부 API 호출 최적화 → 277ms → 29ms 개선**:
  - LinkedHashSet 선집계 및 HashMap 캐싱 구조 설계
  - 외부 API 호출 횟수 20회 -> 2회, **277ms → 29ms** 단축

- **서버 부하 예측 및 분산**:  
  - S3 Presigned URL 구조를 설계하여 대용량 이미지 업로드 시 발생하는 서버 트래픽 병목 현상 제거

- **아키텍처 리팩토링 제안**:
  - Fat Gateway 문제를 분석
  - 인증/라우팅 중심으로 Gateway 책임을 제한하는 구조 개선안 제안 및 설계


## 🧱 아키텍처 및 ERD

<details>
<summary><strong>System Architecture</strong></summary>

<br>

![System Architecture](https://github.com/user-attachments/assets/19147fb4-4eda-4204-9fda-a2cdabf41299)

- MSA 기반 도메인 분리로 서비스 간 결합도 최소화  
- AWS ECS Fargate + GitHub Actions로 서버리스 CI/CD 인프라 구축  

### **🔍1. 발견한 문제점: gRPC 오남용과 API Gateway의 비대화**:  

> **"기술의 맹목적인 도입보다, 적재적소에 맞는 프로토콜 선택이 중요함을 깨달았습니다."**

- **문제**: 모든 통신을 gRPC로 통일 → API Gateway에 REST↔gRPC 변환 로직 집중  
- **결과**: Gateway 비대화, 신규 API 추가 시 유지보수 부담 증가  

### **🛠️2. 리팩토링 제안 및 향후 계획**:

| 구분 | 기존 | 향후 계획|
|:--:|:--:|:--:|
| Gateway 역할 | 인증 + 라우팅 + 프로토콜 변환 |**인증 + 라우팅**|
| 외부 통신 | REST (Gateway에서 변환) | REST |
| 내부 통신 | gRPC | gRPC(고성능 유지) |


- **효과**: Gateway 단순화, 서비스 독립성 강화, 전체 응답 속도 개선

</details>

<details>
<summary><strong>Database ERD</strong></summary>

<br>

![Database ERD](https://github.com/user-attachments/assets/55d7995c-f23d-42b2-a387-b0b645154c07)

- **역정규화(Chatroom)를 통한 조회 성능 최적화**:
  - 채팅 목록 조회 시 매번 메시지 테이블을 조인하거나 대량의 데이터를 스캔하지 않도록, Chatroom 엔티티에 최신 메시지 정보를 추가했습니다.

- **유기적인 거래 상태 관리(Product ↔ Chatroom ↔ Trade)**:
  - 상품 등록부터 채팅을 통한 문의, 최종 거래 완료까지 이어지는 비즈니스 흐름을 고려하여 상태 관리 로직을 구축했습니다.

- **데이터 확장성 고려(1:N 분리)**:
  - ProductImage와 MessageImage를 별도 테이블로 분리하여 상품 및 메시지당 다중 이미지 관리가 가능하도록 설계했습니다.

- **안정성 및 이력 관리(BaseEntity & Soft Delete)**:
  - 모든 엔티티에 BaseEntity를 통한 생성/수정 시간 기록을 적용하고, 데이터 복구 및 사용자 이력 추적을 위해 물리적 삭제 대신 Soft Delete 방식을 채택했습니다.

</details>

  
## 🔥 최적화 및 트러블슈팅

<details> <summary><b>💬 1. gRPC 기반 실시간 채팅 메시지 조회 최적화 </b></summary>

- 문제 상황: 대용량 메시지 조회 시 성능 저하 -> 100건 조회 시 109s 

- 해결 방안:
  - Offset 방식 대신 Cursor 기반 페이징 적용
    - chatroom_id, created_at, id 복합 인덱스 적용   
    - 20건씩 조회
      
- 결과: 109s -> 277ms (약 99% 개선)

- [관련 PR](https://github.com/Hanium2025/msa-server/pull/119)    

| Before | After |
|:--:|:--:|
| ![image](https://github.com/user-attachments/assets/be4077c7-337e-499b-b5b6-6f42744907ee) | ![image](https://github.com/user-attachments/assets/f6a0033d-5252-46bb-96f8-bb70cdd68538) |

</details>


<details> <summary><b>🖼️ 2. 이미지 전송 시 서버 트래픽 부하 발생 예상 </b></summary>

- 문제 상황: 채팅 사진 전송 시 서버 트래픽 부하 발생 예상

- 해결 방안:
  - S3 Presigned URL
    - 클라이언트 직접 업로드 구조를 설계하여 서버 부하 최소화
      
| S3 Presigned 시퀀스 다이어그램 | msa-image-bucket에 저장되는 사진 |
|:--:|:--:|
| ![image](https://github.com/user-attachments/assets/ba6f6078-9c75-4719-939f-b9005c2b4c6e) | ![image](https://github.com/user-attachments/assets/f2ff3220-0df5-4c00-8fc9-d8c0ca8f102c) |

- [관련 PR](https://github.com/Hanium2025/msa-server/pull/43)
</details>
<details> <summary><b>⚡ 3. 외부 API N회 호출 문제 해결</b></summary>

- 문제 상황: 메시지 조회 시 사용자 프로필 API가 메시지 수만큼 반복 호출되는 N+1 문제 발생 (277ms)

- 해결 방안:

  1. ID 선집계: 참여자 ID를 LinkedHashSet으로 추출하여 중복 호출 제거
  2. HashMap 사용: HashMap 기반 캐시를 적용하여 데이터 재사용성 극대화
  
- 결과: 외부 API 호출 20회 → 2회로 감소, 응답 시간 277ms → 29ms로 대폭 개선

- [관련 PR](https://github.com/Hanium2025/msa-server/pull/121)

| Before | After |
|:--:|:--:|
| ![image](https://github.com/user-attachments/assets/8842aa92-6b47-464b-ba09-98ad5c4c9ec9) | ![image](https://github.com/user-attachments/assets/34b598c7-7f7d-49ba-bd5d-f6831efe7ba5) |

</details>

<details> <summary><b>⚙️ 4. 최근 메시지 업데이트 실패 이슈 해결</b></summary>

- 문제 상황:
  - 비동기 채팅 로직 실행 시 메시지는 저장되나, 채팅방 최신 정보 업데이트가 누락됨

- 원인 분석:
    1. 비동기 스레드 실행으로 인한 트랜잭션 경계 이탈
    2. 동일 클래스 내 메서드 호출(Self-invocation)로 인해 Spring AOP 프록시 미적용

- 해결 방법:
  - 서비스 로직을 별도 서비스(ChatMessageTxService)로 분리하여 프록시 객체 호출 유도

- 결과: JPA 변경 감지(Dirty Checking) 정상 작동 확인 및 데이터 정합성 확보

| 최근 메시지 누락된 실행 응답 결과 | 에러 원인 |
|:--:|:--:|
| ![image](https://github.com/user-attachments/assets/abd5ab3b-bc3d-4bcd-a46c-2923285ea461) | ![image](https://github.com/user-attachments/assets/6a9190f7-0e3c-488c-9a22-3a5fb9ebd8c0) |

</details>



## 🧩 협업 및 운영 프로세스

> 단순 개발을 넘어 팀의 생산성을 높이기 위해 노력한 과정입니다.

<details> <summary><b>🏃‍♂️ 애자일(Agile) 기반 스프린트 및 백로그 관리</b></summary>

- 백로그 중심 관리: 고정된 기획서 대신 유연한 백로그 형태로 요구사항을 관리하여 변화에 기민하게 대응

- 플래닝 포커 & 스크럼: 주 단위 스프린트 플래닝을 통해 우선순위를 조정하고, 데일리 스크럼으로 진행 상황 공유

| 백로그 관리 | 스프린트 | 데일리 스크럼 |
|:--:|:--:|:--:|
| <img src="https://github.com/user-attachments/assets/9c6aab6e-b32a-4a42-bab7-33d148f3b5da" width="400" /> | <img src="https://github.com/user-attachments/assets/b5fb0205-1b4e-40a1-a695-eab5a45ac0c5" width="400" /> | <img src="https://github.com/user-attachments/assets/20c8ca7f-f317-4e5a-8ccc-e979c1dd592c" width="400" /> |


</details>

<details> <summary><b>🤝 문서화 중심의 체계적인 협업 프로세스</b></summary>

- Issue & PR 기반 협업: 모든 기능 개발과 버그 수정은 GitHub Issue를 통해 트래킹하고, 상세한 PR 코멘트로 코드 리뷰 진행
</details>

<details> <summary><b>💬 기술 공유 & 스터디 세션 주도</b></summary>

> 📚 팀 내 신규 기술 학습과 공유 문화를 주도하며 기술적 병목을 해결했습니다.

| 이벤트 스토밍 | DB 설계 세션 |
|:--:|:--:|
| <img src="https://github.com/user-attachments/assets/5646e73e-b02b-46d4-9953-42db8435c554" width="400" /> | <img src="https://github.com/user-attachments/assets/00cb5533-b559-4957-a1fa-1affbe3f1b69" width="400" /> |

| 스터디 세션 자료 | 설계 구조 설명 세션 |
|:--:|:--:|
| <img src="https://github.com/user-attachments/assets/55d49a2e-6452-4aab-96b0-02b50c49b03d" width="400" /> | <img src="https://github.com/user-attachments/assets/0deced79-572c-416f-909e-f04203b48eec" width="400" /> |


- 이벤트 스토밍을 통해 도메인 흐름과 핵심 비즈니스 이벤트를 정리하고, 서비스 경계를 명확히 정의  
- gRPC, AWS ECS 등 프로젝트 핵심 기술을 주제로 한 **스터디 세션 기획 및 발표**  
- 기술 적용 이유와 내부 구조를 시각화하여 **팀원 전체가 빠르게 이해·적용할 수 있도록 지원**  
- 세션 자료와 이벤트 스토밍 결과를 Notion에 정리하여 **기술·도메인 문서화 기반 학습 문화 정착**

> ✅ *결과:* 이벤트 스토밍을 통한 도메인 이해도 향상과 신규 기술 도입 시 팀 온보딩 속도 향상, 코드 품질 일관성과 설계 의사결정의 명확성 확보


</details>

## 🔗 주요 PR 기록
| 분류 | 내용 | 링크 |
|------|------|------|
| ⚙️ 인프라 | 멀티모듈 프로젝트 구조 적용 및 common 모듈 분리 및 샘플 코드 작성 | [#10 PR](https://github.com/Hanium2025/msa-server/pull/10) |
| ⚙️ 인프라 | Config Service 구축 및 공통 yml 분리 | [#18 PR](https://github.com/Hanium2025/msa-server/pull/18) |
| 🚀 CI/CD | ECS Fargate + GitHub Actions 배포 자동화 | [#30 PR](https://github.com/Hanium2025/msa-server/pull/30) |
| 💬 채팅 |채팅 기능 개발(텍스트, 사진)| [#43 PR](https://github.com/Hanium2025/msa-server/pull/43) 

## ✍️ 기술 블로그 & 회고
- [EventStorming 설계 블로그](https://mincodhub.tistory.com/1)
- [gRPC 채팅 스트림에서 트랜잭션이 적용되지 않았던 이유(feat. 자기 호출, 프록시)](https://mincodhub.tistory.com/7)
