# IBC Biosafety Control
 
기관생물안전위원회(IBC, Institutional Biosafety Committee)를 통한 **생물안전 관리 시스템**.
감염 가능성이 있는 생물체의 생물안전성 확보와 연구시설 관리를 지원하는 관리자 시스템이다.
 
> 이 프로젝트는 "테이블 CRUD"가 아니라 **생물안전 규정을 도메인 모델과 상태 흐름으로 옮긴** 설계에 초점을 둔다.
> 두 개의 규칙 엔진 — ① 생물안전등급 정책, ② IBC 심의 상태 머신 — 이 도메인의 중심에 있다.
 
---
 
## 업무 배경
 
기관은 감염성 생물체를 다루는 연구실을 운영할 때 생물안전을 확보할 법적·윤리적 책임을 진다.
이를 위해 기관생물안전위원회(IBC)를 구성하고, 생물체의 위험도를 등급으로 분류하며,
연구계획을 심의하고, 종사자 교육과 사고 대응을 관리한다. 이 시스템은 그 업무 흐름을 전산화한다.
 
핵심 업무:
- 기관 내 생물체 관련 연구자·연구시설 전수조사
- BL1~BL4 생물안전등급 분류
- IBC 구성 및 위원(생물안전관리책임자/전담자/내부위원/외부위원) 위촉
- 생물안전관리규정 개정 이력 관리
- 생물체 등급에 따른 IBC 운영 권장/필수 판단
- 연구책임자의 실험실 위해성 평가
- 연구종사자의 생물안전교육 이수 관리
- 안전사고 보고 및 대응 기록 관리
---
 
## 핵심 설계: 규정을 코드로
 
### ① 생물안전등급 정책 (`biosafety-policy.ts`)
 
등급별 IBC 운영 요구수준을 산발적으로 하드코딩하지 않고, 규정의 단일 출처로 한 곳에 둔다.
 
| 분류 | 등급 | 설명 | IBC 운영 |
|---|---|---|---|
| 일반 생물체 | BL1 | 건강한 성인에게 질병을 일으키지 않음 | **권장** |
| 감염성 생물체 | BL2 | 감염시 증세가 심각하지 않으며, 예방과 치료가 쉬움 | **필수** |
| 감염성 생물체 | BL3 | 감염시 증세가 심각할 수 있으나, 예방과 치료가 가능 | **필수** |
| 감염성 생물체 | BL4 | 감염시 증세가 심각하며, 예방과 치료가 어려움 | **필수** |
 
> 생물체는 **일반 생물체(BL1)** 와 **감염성 생물체(BL2~4)** 로 분류된다. `resolveOrganismCategory()` 가 등급에서 분류를 판정한다.
 
- 시설 등록 시 IBC 요구수준을 **프론트가 보내지 않는다.** 서버가 등급에서 판정한다.
- 시설이 여러 생물체를 보유하면 **실효 등급 = 최고 등급**(가장 위험한 것 기준으로 봉쇄 결정).
- 규정이 바뀌면 이 정책 파일만 수정한다.
### ② IBC 심의 상태 머신 (`review-state-machine.ts`)
 
심의는 아무 상태로나 바뀌지 않는다. 규정된 전이만 허용한다.
 
```
REQUESTED ──▶ UNDER_REVIEW ──▶ APPROVED
     │              │
     │              └────────▶ REJECTED
     └───────────────────────▶ REJECTED   (접수 단계 반려)
```
 
- `APPROVED`/`REJECTED`는 종결 상태 — 이후 전이 불가 (승인된 심의를 뒤집을 수 없음).
- `REQUESTED`에서 `APPROVED`로 직행 불가 (심의 절차를 건너뛸 수 없음).
- 모든 전이는 `ReviewTransition`에 이력으로 남아 **감사 추적**이 된다.
- 컨트롤러가 status를 직접 덮어쓰지 않고 서비스의 approve/reject만 호출 → 잘못된 전이를 구조적으로 차단.
### ③ IBC 구성 절차 상태 머신 (`committee-stage-machine.ts`)
 
위원회 구성은 규정된 4단계를 순서대로 밟는다 (역행/건너뛰기 불가).
 
```
1단계 전수조사 ──▶ 2단계 위원 위촉 ──▶ 3단계 규정 개정 ──▶ 4단계 구성 완료
 (BL1~4 분류)      (책임자·전담자·        (IBC 위원 자문      (구성 완료)
                    내부·외부 위원)        → 기관장 결재)
```
 
- 각 단계 산출물이 다음 단계의 전제이므로 순방향 1칸 전이만 허용.
- 3단계는 "IBC 위원 자문 → 기관장 결재"라는 규정 절차를 명시.
- 모든 단계 전이를 `CommitteeStageLog`에 이력으로 기록.
### 4주체 책임체계 (`responsibility-matrix.ts`)
 
기관 생물안전확보를 위한 네 주체의 책무를 규정 상수로 고정 (임의 변경 방지):
 
| 주체 | 책무 |
|---|---|
| 기관생물안전위원회 | 전문적 생물안전 심의, 생물안전 계획 수립자문 |
| 기관생물안전관리책임자 | 기관 수준 안전관리, 안전교육, IBC 운영 지원 |
| 연구책임자 | 실험실 위해성 평가·관리, 실험실 수준 안전교육 |
| 연구종사자 | 안전한 실험 수행, 안전사고시 적절한 대응 |
 
---
 
## 핵심 도메인
 
| 모델 | 설명 |
|---|---|
| `Facility` | 연구시설 (등급, IBC 요구수준, 전수조사 시점) |
| `Organism` | 생물체 (학명, 등급, 소속 시설) |
| `IbcCommittee` | 위원회 (유형: IBC/IACUC/IRB, 구성 단계 포함) |
| `IbcMember` | IBC 위원 (역할, 위촉/해촉 이력) |
| `IbcReview` | IBC 심의 (상태 머신 + 전이 이력) |
| `RiskAssessment` | 위해성 평가 (위험요인, 저감대책, 등급) |
| `EducationRecord` | 교육 이수 기록 (유효기간 → 재이수 판정) |
| `IncidentReport` | 안전사고 보고 (심각도, 보고→조사→종결) |
| `RegulationRevision` | 생물안전관리규정 개정 이력 |
| `ReviewTransition` | 심의 상태 전이 감사 로그 |
| `Researcher` | 연구자 (연구책임자 PI 여부) |
 
---
 
## API 목록
 
| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/api/biosafety-levels` | 생물안전등급 정책(설명 + IBC 권장/필수) |
| GET | `/api/facilities` | 연구시설 목록 (등급 정책 포함) |
| POST | `/api/facilities` | 시설 등록 (등급→IBC 요구수준 자동판정) |
| GET | `/api/responsibilities` | 4주체 책무 매트릭스 (위원회/책임자/연구책임자/연구종사자) |
| GET | `/api/committees/members` | 현직 위원 목록 (역할별 그룹핑) |
| GET | `/api/committees/stages` | IBC 구성 4단계 절차 정의 |
| GET | `/api/committees/{id}/status` | 위원회 구성 현황 (현재/다음 단계) |
| PATCH | `/api/committees/{id}/advance` | 구성 절차 다음 단계로 진행 (상태 머신 검증) |
| GET | `/api/reviews` | IBC 심의 목록 (status 필터, 다음 가능 상태 포함) |
| POST | `/api/reviews` | 심의 요청 생성 (REQUESTED) |
| PATCH | `/api/reviews/{id}/start` | 심의 착수 (→ UNDER_REVIEW) |
| PATCH | `/api/reviews/{id}/approve` | 승인 (→ APPROVED, 상태 머신 검증) |
| PATCH | `/api/reviews/{id}/reject` | 반려 (→ REJECTED, 상태 머신 검증) |
| GET | `/api/education-records` | 교육 이수 현황 (유효/만료 판정) |
| POST | `/api/education-records` | 교육 이수 등록 |
| GET | `/api/incidents` | 안전사고 목록 (심각도별 집계) |
| POST | `/api/incidents` | 안전사고 보고 |
 
---
 
## 관리자 대시보드
 
`web/index.html` (서버가 정적 서빙, `http://localhost:3000/`)
 
- BL1~BL4 생물안전등급 카드 (봉쇄 강도에 따라 색이 짙어짐)
- IBC 구성 절차 타임라인 (책임자 → 전담자 → 내부위원 → 외부위원)
- 연구시설 목록 / IBC 심의 요청 목록
- 교육 이수 현황 / 안전사고 보고 현황
---
 
## ERD 초안
 
```
Researcher ──< Facility(PI) ──< Organism
                   │  │  └──< IncidentReport >── Researcher(reporter)
                   │  └─────< RiskAssessment >── Researcher(PI)
                   └─────────< IbcReview >────── IbcCommittee ──< IbcMember
                                  │                    └──< RegulationRevision
                                  └──< ReviewTransition
Researcher ──< EducationRecord
```
 
---
 
## 실행
 
```bash
# 1) 로컬 DB (PostgreSQL)
docker compose up -d
 
# 2) 스키마 + 데모 데이터
pnpm install
cp .env.example .env
pnpm prisma:generate
pnpm prisma migrate dev --name init
pnpm seed
 
# 3) 서버 + 대시보드
pnpm start:dev
# → http://localhost:3000/
```
 
---
 
## 아키텍처
 
**모듈러 모놀리스.** 도메인별 모듈(biosafety/facility/committee/review/education/incident)을
한 프로세스에 묶되 경계를 명확히 분리했다. 처음부터 MSA로 쪼개지 않고, 트래픽·팀이 커지면
이 경계선을 따라 서비스로 분리할 수 있게 설계했다. — "왜 MSA인가"에 답할 수 있을 때 분리한다.
 
---
 
## 포트폴리오 어필 포인트
 
- **규정을 상태 흐름으로 설계**: 단순 CRUD가 아니라, 생물안전 규정을 ① 등급 정책 규칙과
  ② 심의 상태 머신이라는 두 도메인 엔진으로 옮겼다. 잘못된 승인/등급 판정이 구조적으로 불가능하다.
- **감사 추적**: 모든 심의 상태 전이를 이력으로 남겨, 규제 도메인에 필요한 추적성을 확보.
- **판정 로직의 서버 소유**: IBC 요구수준·유효교육 여부를 클라이언트가 아니라 서버가 규칙으로 판정.
- **모듈러 모놀리스**: 과설계(성급한 MSA) 없이 확장 가능한 경계 설계.
- **검증**: 등급 정책·상태 머신·교육 판정을 단위/통합 테스트 34종으로 검증.
- **의료·바이오 규제 도메인**: IRB/CTMS 인접 영역으로, 규제 기반 시스템 설계 역량을 보여준다.
## 검증 현황
 
- 규칙 엔진 단위 테스트 22종 (등급 정책 11 + 상태 머신 11)
- 서비스 통합 테스트 12종 (등급 자동판정, 상태 전이 강제, 교육 판정)
- 전체 TypeScript 타입 검증 통과
