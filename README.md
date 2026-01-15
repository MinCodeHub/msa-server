# 🛒 Picky: MSA 기반 소셜 마켓 플랫폼

> 본 저장소는 원본 msa-server 프로젝트에서 **제가 직접 기여한 핵심 영역(인프라 / 채팅 / 배포 자동화)** 중심으로 정리한 포크 버전입니다.

## 🔥 Deep Dive & Troubleshooting

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


<details> <summary><b>💬 2. 이미지 전송 시 서버 트래픽 부하 발생 예상 </b></summary>

- 문제 상황: 채팅 사진 전송 시 서버 트래픽 부하 발생 예상

- 해결 방안:
  - S3 Presigned URL
    - 클라이언트 직접 업로드 구조를 설계하여 서버 부하 최소화
      
| S3 Presigned 시퀀스 다이어그램 | msa-image-bucket에 저장되는 사진 |
|:--:|:--:|
| ![image](https://github.com/user-attachments/assets/ba6f6078-9c75-4719-939f-b9005c2b4c6e) | ![image](https://github.com/user-attachments/assets/f2ff3220-0df5-4c00-8fc9-d8c0ca8f102c) |

- [관련 PR](https://github.com/Hanium2025/msa-server/pull/43)
</details>
<details> <summary><b>⚡ 3. 외부 API N회 호출 문제 해결 (응답 시간 90% 개선)</b></summary>

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

<details> <summary><b>⚙️ 3. 최근 메시지 업데이트 실패 이슈 해결</b></summary>

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

## 🧱 Architecture & Infrastructure

<details>
<summary><strong>System Architecture</strong></summary>

<br>

![System Architecture](https://github.com/user-attachments/assets/19147fb4-4eda-4204-9fda-a2cdabf41299)

**설명**:  
여기에 시스템 아키텍처 설명 작성

**포인트**:  
- 예) MSA 구조
- 예) gRPC / REST 통신
- 예) ECS 기반 배포

**느낀 점**:  
여기에 느낀 점 작성

</details>

<details>
<summary><strong>Database ERD</strong></summary>

<br>

![Database ERD](https://github.com/user-attachments/assets/55d7995c-f23d-42b2-a387-b0b645154c07)

**설명**:  
여기에 ERD 설명 작성

**포인트**:  
- 예) 도메인 분리 기준
- 예) 정규화/비정규화 이유
- 예) FK 설계 의도

**느낀 점**:  
여기에 느낀 점 작성

</details>


## 🧩 협업 및 운영 프로세스
단순 개발을 넘어 팀의 생산성을 높이기 위해 노력한 과정입니다.

<details> <summary><b>🏃‍♂️ 애자일(Agile) 기반 스프린트 및 백로그 관리</b></summary>

- 백로그 중심 관리: 고정된 기획서 대신 유연한 백로그 형태로 요구사항을 관리하여 변화에 기민하게 대응

- 플래닝 포커 & 스크럼: 주 단위 스프린트 플래닝을 통해 우선순위를 조정하고, 데일리 스크럼으로 진행 상황 공유

- 회고 문화: 스프린트 종료 후 회고를 진행하여 팀 프로세스를 지속적으로 개선

<img width="800" alt="Product Backlog" src="https://github.com/user-attachments/assets/9c6aab6e-b32a-4a42-bab7-33d148f3b5da" /> 실제 스프린트 플래닝 시 관리한 제품 백로그의 일부입니다.

</details>

<details> <summary><b>🤝 문서화 중심의 체계적인 협업 프로세스</b></summary>

- Issue & PR 기반 워크플로우: 모든 기능 개발과 버그 수정은 GitHub Issue를 통해 트래킹하고, 상세한 PR 코멘트로 코드 리뷰 진행
</details>

<details> <summary><b>💬 기술 공유 & 스터디 세션 주도</b></summary>
  
<img width="400" alt="aws-ecs-study" src="https://github.com/user-attachments/assets/c45984ff-28fe-4f07-871d-c4b7c9e81dff" />
<img width="400" alt="image" src="https://github.com/user-attachments/assets/55d49a2e-6452-4aab-96b0-02b50c49b03d" />

> 📚 팀 내 신규 기술 학습과 공유 문화를 주도

- gRPC, AWS ECS 등 프로젝트 핵심 기술을 주제로 한 **스터디 세션 기획 및 발표**  
- 기술 적용 이유와 내부 구조를 시각화하여 **팀원 전체가 빠르게 이해·적용할 수 있도록 지원**  
- 세션 자료를 Notion에 정리하여 **기술 문서화 기반 학습 문화 정착**

> ✅ *결과:* 신규 기술 도입 시 팀 온보딩 속도 향상 및 코드 품질 일관성 확보

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
