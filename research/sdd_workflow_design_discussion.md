# SDD 워크플로우 설계 논의 과정

**작성일**: 2024-11-23
**참여자**: Randy (개발자) + Claude (AI)
**목적**: 랜디 팀을 위한 맞춤형 Spec-Driven Development 워크플로우 설계

---

## 📋 목차

1. [프로젝트 배경](#1-프로젝트-배경)
2. [GitHub Spec-Kit 분석](#2-github-spec-kit-분석)
3. [계층 구조 설계 논의](#3-계층-구조-설계-논의)
4. [재사용성 문제 논의](#4-재사용성-문제-논의)
5. [SDD vs Agile 모순 논의](#5-sdd-vs-agile-모순-논의)
6. [리팩토링 정책 논의](#6-리팩토링-정책-논의)
7. [최종 확정안](#7-최종-확정안)
8. [세부 정의](#8-세부-정의)

---

## 1. 프로젝트 배경

### 1.1 프로젝트 목적

백엔드 개발팀에서 AI와 협업하는 방식을 표준화하기 위한 프로젝트. GitHub Spec-Kit을 커스터마이징하여 팀만의 워크플로우를 적용.

### 1.2 Randy의 개발 프로세스

1. **요구사항 정의**
   - 시스템 요구사항 (기술적 요구사항: DB 커넥션 풀, 아키텍처 등)
   - 비즈니스 요구사항 (업무적 요구사항)
   - 모든 요구사항은 ID로 식별

2. **기능 정의**
   - 요구사항 ID별로 기술적 구현 방법 정의

3. **설계 (병렬 가능)**
   - DB 테이블 설계
   - 인터페이스 설계
   - 소프트웨어 아키텍처 설계

4. **비즈니스 로직 설계**
   - 테스트 케이스 포함

5. **구현 및 검증**

---

## 2. GitHub Spec-Kit 분석

### 2.1 Spec-Kit의 핵심 발견

**리서치 파일 분석 결과**:
- Spec-Kit은 "실행"이 아닌 **"템플릿 생성"** 도구
- 실제 작업은 AI 에이전트가 수행
- 복잡한 빌드 시스템은 14개 에이전트 지원 때문

### 2.2 Spec-Kit의 구조

```
project/
├── .specify/memory/
│   └── constitution.md        # 프로젝트 전체 규칙
│
└── features/
    └── [feature-name]/
        ├── spec.md             # 요구사항
        ├── plan.md             # 기술 계획
        ├── tasks.md            # 실행 태스크
        └── (구현 코드)
```

### 2.3 초기 의문점

**Randy의 질문**:
> "신규 구축 프로젝트면 하나의 Feature에서 요구사항부터 spec까지 다 하나? 아니면 Feature를 나눠서? 그럼 전체는 어디서 관리?"

**Claude의 추측**:
- Option A: Feature = 작은 단위 (001-user-auth, 002-payment)
- Option B: 큰 단위로 관리 (000-initial-system)
- Option C: 계층 구조 (프로젝트 루트 + Feature별)

**문제점 발견**:
- Spec-Kit에는 "전체 앱이 뭘 하는가"에 대한 Master 문서 없음
- 커뮤니티도 이 문제 인식: "chain of spec이 하나의 application이 될 것"

---

## 3. 계층 구조 설계 논의

### 3.1 Randy의 첫 번째 제안: 3계층

**제안**:
```
1계층: Master Spec (전체)
2계층: Module Spec (큰 기능 단위)
  - Plan도 여기서 정의
3계층: Feature Spec (세분화된 기능)
  - Task, 구현
```

**목적**: 기능의 재사용성 향상, 모듈화

### 3.2 Claude의 비판적 검토

#### 👍 동의한 부분:
- Master Spec 필요성 완전 공감
- 계층화 자체는 합리적
- Plan을 2계층 배치는 합당

#### 🤔 비판한 부분:

**1) 3단계가 진짜 필요한가?**
```
예시: 전자상거래 플랫폼

1계층: Master Spec
2계층: "주문/결제"
3계층: "장바구니", "결제 처리", "주문 조회"
```

**의문**: 2계층 자체가 충분히 작으면 3계층 불필요한 거 아닌가?

**2) Plan의 위치 애매함**
```
2계층: Plan 있음
3계층: Plan 없음? 있음?
  - 없으면: 3계층은 HOW를 어떻게 정의?
  - 있으면: 2계층 Plan은 뭐?
```

**3) 복잡도 vs 실용성**
- 작은 프로젝트: 오버엔지니어링
- 큰 프로젝트: 계층 스킵 허용? → 규칙 무의미

**4) 시스템 요구사항 위치**
- DB 커넥션풀, 로깅 → 1계층? 2계층?
- 1계층이면 Master Spec이 너무 비대해짐

### 3.3 Randy의 답변

**3계층 목적 재정의**:
> "2계층 spec을 세분화/모듈화하여 재사용성을 높이려 했어. 좀 더 기능을 세분화하려고."

**문제 인식**:
> "하지만 각 브랜치마다 AI가 spec, plan 하면 다른 브랜치 기능을 식별 못한 채 무조건 구현할 것 같은데..."

### 3.4 대안 제시: 2계층 + 재사용 식별 과정

**Randy의 새 제안**:
- 3계층 대신 2계층으로 단순화
- Plan 단계에서 모듈화 계획
- **선택적 과정**: 재사용 가능 로직 식별

---

## 4. 재사용성 문제 논의

### 4.1 핵심 문제 정의

**Randy의 우려**:
> "AI는 구현만 하고, 수정하려면 spec부터 재정의해야 해서 SDD 지옥에 빠질 수도..."

**병렬 개발 시나리오**:
```
Branch A: User 모델 구현
Branch B: User 모델 또 구현 (Branch A 모르고)
→ 나중에 merge 시 충돌/중복
```

### 4.2 Claude의 분석

#### 재사용 식별 타이밍 문제

```
main 브랜치 (비어있음)
  ├── 001-user-auth (개발 중) - User 모델 예정
  └── 002-payment (개발 중) - User 모델 필요
```

**3가지 케이스**:

| Case | 상황 | 재사용 식별 가능? |
|------|------|------------------|
| A | 001이 main에 머지됨 | ✅ 가능 (main 스캔) |
| B | 001 개발 중, 002 시작 | ❌ 불가능 (다른 브랜치) |
| C | 001, 002 동시 개발 | ❌ 불가능 + 충돌 위험 |

#### AI가 구현 단계에서 커버 가능한가?

**✅ 가능한 경우**:
- Main 브랜치 스캔
- 비슷한 이름/기능 클래스 찾기

**❌ 불가능한 경우**:
- 다른 feature 브랜치 코드 (접근 불가)
- "의미적으로 같은" 컴포넌트 판별 (UserAccount vs User)

**결론**: Main만 체크하면 **50%** 커버, 완벽하진 않음

### 4.3 Claude의 대안 제시

#### Option A: Central Registry
```markdown
# Master Spec

## Common Components Registry

### Models
- **User** (Owner: 001-user-auth, Status: Merged)
  - 위치: src/models/user.py

- **Order** (Owner: 002-payment, Status: In Development)
  - 예정 위치: src/models/order.py
```

**워크플로우**:
1. Spec 작성 시 → Registry 확인 필수
2. Plan 작성 시 → Registry에 등록
3. 구현 완료 시 → Registry 업데이트

**장점**: 병렬 개발 충돌 방지, 투명성
**단점**: 수동 유지보수 필요

#### Option B: Plan 단계에서 강제 충돌 체크

```markdown
## Reusability Check (Required)
1. Scan main branch
2. Scan other feature branches (PR 목록)
3. Document reuse plan
```

**장점**: 체계적
**단점**: 다른 브랜치 접근 어려움, 복잡함

#### Option C: 구현 단계 AI 자동 감지

**가능한 것**: Main 브랜치 스캔
**불가능한 것**: 다른 브랜치, 의미적 동일성 판별
**결론**: 50% 커버

#### Option D: Shared Module First

```
000-core-models → User, Order 먼저 구현
  ↓
001-user-auth → 000 재사용
002-payment → 000 재사용
```

**장점**: 중복 원천 차단
**단점**: 미리 예측 필요, 애자일하지 않음

### 4.4 Randy의 현실 인식

**Randy의 팀 상황**:
1. 브랜치 전략: 동시에 여러 feature 개발 → main PR → 충돌은 그때그때 해결
2. Registry 수동 관리: 힘듦, strict하게 갈 수 없음
3. 공통 모듈 미리 식별: 가능하지만 상황에 따라 바뀜

**핵심 통찰**:
> "설계를 처음부터 너무 strict하게 정하면 톱니바퀴처럼 돌아가다가 요건 변경 시 변화가 너무 클 것 같아."

---

## 5. SDD vs Agile 모순 논의

### 5.1 Claude의 새로운 접근: Loose SDD

**핵심 아이디어**:
> "Spec은 가볍게, 구현은 자유롭게, 정리는 주기적으로"

#### 1단계: Spec (가볍게)
- 상세 설계 안 함
- What/Why만 명확히
- DB 스키마, API 스펙, 클래스 구조 ❌

#### 2단계: Plan (방향성만)
- 대략적 방향
- "변경 가능" 명시
- 공통화 가능성 메모 (강제 아님)

#### 3단계: 구현 (자유롭게 + AI 가이드)
- AI가 main 브랜치 스캔
- 비슷한 클래스 발견 시 사람에게 질문
- 사람이 최종 판단 (재사용/수정/새로 만들기)

#### 4단계: Review (주기적 정리)
- AI가 중복 코드 자동 감지
- 공통화 제안
- 사람이 accept/reject
- 리팩토링 실행

**워크플로우**:
```
Master Spec (가볍게)
  ↓
Feature Spec (What/Why만)
  ↓
Plan (방향성, 변경 가능)
  ↓
Implement (자유롭게 + AI 중복 체크)
  ↓
Review (주기적 정리)
```

### 5.2 Randy의 단순화 요구

**Randy**:
> "너무 어렵다. 단순화하자. 사람들 못 써."

**단순화 요구사항**:
1. 2계층으로 가자
2. 구현 후 리뷰 (생각날 때마다)
3. 리팩토링 대상 식별 → 각 feature의 plan, task 전파?

### 5.3 리팩토링 시 문서 업데이트 논의

**Randy의 질문**:
> "리팩토링 대상이 식별되면 각 feature의 plan, task도 고쳐야 할까?"

#### Claude의 첫 번째 답: NO (문서 = 히스토리)

**논리**:
```
Feature 001 (2024-11-01)
  Plan: "UserService 만들기로 결정"
  → 구현함

Review (2024-11-23)
  발견: UserService 비대함
  리팩토링: → AuthService + ProfileService

001의 Plan 수정?
  ❌ "당시엔 UserService 하나로 계획했다"는 히스토리
```

**제안**:
- 리팩토링도 별도 Feature로 (010-refactor-user-service)
- 기존 Feature 문서는 Frozen (히스토리 보존)
- Master Spec만 "현재 상태" 업데이트

**규칙**:
| 항목 | 규칙 |
|------|------|
| Feature 문서 | 작성 후 수정 ❌ (히스토리) |
| Master Spec | 주기적 업데이트 ✅ (현재 상태) |
| 리팩토링 | 별도 Feature로 문서화 |

---

## 6. 리팩토링 정책 논의

### 6.1 Randy의 깨달음

**Randy**:
> "이렇게 해놓고 보니까... '문서는 history, 코드는 현재' → SDD의 철학에 완전히 위배되네. Spec이 Source of Truth여야 하는데!"

**결정**:
> "리팩토링 정책 제외하자. 유연하게 대처. 리팩토링 적용되는 feature의 Spec/Plan/Task 고쳐야겠어."

### 6.2 SDD 철학 재확인

**SDD = Spec이 Source of Truth**
- 문서가 현재 상태를 반영해야 함
- "문서는 옛날 거, 코드만 믿어" ≠ SDD

**원칙 확립**:
- ✅ 문서 = Source of Truth
- ✅ 코드 변경 → 문서도 업데이트
- ✅ 리팩토링 → 영향받는 문서 전부 업데이트
- ✅ 문서와 코드는 항상 일치

---

## 7. 최종 확정안

### 7.1 문서 구조 (2계층)

```
project/
├── master-spec.md              # 전체 프로젝트 정의
├── matrix.md                   # 추적성 Matrix (자동 생성)
├── reviews/                    # 리뷰 결과
│   └── YYYY-MM-DD-review.md
│
└── features/
    ├── 001-user-auth/
    │   ├── spec.md             # Feature 명세 (What/Why, BDD)
    │   ├── plan.md             # 기술 설계 (How)
    │   ├── research.md         # 기술 조사 (선택)
    │   └── task.md             # 구현 태스크
    │
    └── 002-payment/
        ├── spec.md
        ├── plan.md
        └── task.md
```

### 7.2 문서 수정 규칙

#### Master Spec
- **수정 가능**: 요구사항 추가/변경/삭제
- **수정 불가**: 구현 디테일 (기술 중립 유지)

#### Feature 문서 (Spec/Plan/Task)
- **수정 가능**: 언제든지
- **원칙**: 문서 = Source of Truth
  - 코드 변경 → 문서 업데이트
  - 리팩토링 → 영향받는 문서 전부 업데이트

### 7.3 핵심 철학

1. **문서 = Source of Truth** (SDD 철학)
2. **2계층 구조** (단순함)
3. **유연한 리팩토링** (정책 없이 실무 대처)
4. **기술 중립적 Spec** (What/Why)
5. **기술 포함된 Plan** (How)

---

## 8. 세부 정의

### 8.1 ID 채번 규칙

Randy의 제안을 그대로 채택:

| 레벨 | ID 형식 | 예시 | Parent |
|------|---------|------|--------|
| Master Spec | `MS-###` | MS-001 | - |
| Feature Spec | `MS-###-FS-###` | MS-001-FS-001 | MS-001 |
| Task | `MS-###-FS-###-T-###` | MS-001-FS-001-T-001 | MS-001-FS-001 |
| Unit Test | `MS-###-FS-###-T-###-UT-##` | MS-001-FS-001-T-001-UT-01 | MS-001-FS-001-T-001 |

**특징**:
- 계층적 구조 (Parent 추적 가능)
- ID만 봐도 전체 경로 파악
- BDD는 ID 없음 (현업 점검용)

### 8.2 BDD vs Unit Test

**Claude의 질문**:
> "BDD 테스트 케이스도 Matrix에 추적?"

**Randy의 답변**:
> "BDD는 현업 점검용. ID 채번 없이 명세서에 정의만 하면 충분."

**정리**:
- **BDD Scenarios**: Feature Spec에 포함, ID 없음, 현업 점검용
- **Unit Tests**: Task에 포함, ID 채번, Matrix 추적

### 8.3 Unit Test 필수 여부

**Claude의 질문**:
> "모든 Task에 UT 필수? application.yml 설정도?"

**Randy의 답변**:
> "코드 레이어의 로직만 테스트."

**정리**:
- 코드 구현 Task → UT 필수 ✅
- 설정/문서 Task → UT 선택 ⬜

### 8.4 Matrix 생성

**Randy**:
> "당연히 자동이지. LLM에 의한."

**정리**:
- `/mysdd.matrix` 실행 시 자동 생성
- 전체 문서 스캔
- ID 패턴 매칭
- Markdown Table 형식

### 8.5 작은 기능 처리 정책

**Randy의 정리**:

#### 기술적 변경 (Spec 밖)
- 예: 로그 레벨, 포트 번호, 설정값
- 처리: Plan 수정 (명시된 경우) 또는 커밋 로그
- Spec에 안 들어감

#### 비즈니스 로직 변경 (Spec 안)
- 예: 유효성 검사 규칙, 업무 흐름
- 처리: Spec부터 다시 수정 (필수)
- 아무리 작아도 Spec 수정 필수

**판단 기준**:
```
Q: "이게 요구사항 변경인가? 구현 디테일인가?"

요구사항 변경 → Spec 수정
구현 디테일 → Plan/Task 수정 또는 커밋 로그
```

### 8.6 Review 결과 저장

**Claude의 제안**:
```
project/
└── reviews/
    ├── 2024-11-23-code-review.md
    └── 2024-11-30-security-review.md
```

**Randy**: (암묵적 동의, 별도 반론 없음)

---

## 9. 커맨드 정의 (Randy의 최종 정리)

### 9.1 `/mysdd.master-spec`

**목적**: Application 전체 정의
**입력**: 사용자의 프로젝트 설명
**출력**:
- `master-spec.md`
- Feature 목록 (대략적)
- MS-ID 채번

**특징**:
- 기술 중립적
- 세부 요건은 feature-spec에서
- Feature별 테스트/점검 단위
- Matrix를 위한 ID 채번

### 9.2 `/mysdd.feature-spec`

**목적**: Feature 세부 명세 작성
**입력**: Master Spec의 Feature
**출력**:
- `features/XXX/spec.md`
- FS-ID 채번 (parent 추적 가능)

**특징**:
- 기술 중립적
- BDD 형식 테스트 케이스 (ID 없음)

### 9.3 `/mysdd.plan`

**목적**: Feature 구현 계획
**입력**: Feature Spec
**출력**:
- `features/XXX/plan.md`
- `features/XXX/research.md` (선택)

**특징**:
- 기술 포함
- 설계 내용

### 9.4 `/mysdd.task`

**목적**: 구현 Task 정의
**입력**: Feature Spec + Plan
**출력**:
- `features/XXX/task.md`
- Task ID 채번
- Unit Test ID 채번

**특징**:
- 단위 테스트 필수 (코드 로직만)
- Master → Feature → Task → UT 추적 가능

### 9.5 `/mysdd.implementation`

**목적**: Task 구현
**동작**:
- Task 구현
- Unit Test 구현
- 완료 항목 체크

### 9.6 `/mysdd.matrix`

**목적**: 요구사항 추적성 Matrix 자동 생성
**입력**: 전체 문서 스캔
**출력**: `matrix.md` (자동 생성)

**특징**:
- MS → FS → Task → UT 연결
- 언제든 실행 가능

### 9.7 `/mysdd.review`

**목적**: 코드 품질 리뷰
**동작**:
- Feature 간 중복 체크
- 컨벤션 위배 체크
- 보안 위험 체크

**특징**:
- **절대 코드 수정 안 함**
- 리뷰 목록만 제시
- `reviews/YYYY-MM-DD-review.md`로 저장

---

## 10. 전체 워크플로우

```
/mysdd.master-spec
  ↓ (Feature 목록, MS-ID 채번)

/mysdd.feature-spec (각 Feature마다)
  ↓ (Feature Spec, BDD 포함, FS-ID 채번)

/mysdd.plan
  ↓ (기술 설계)

/mysdd.task
  ↓ (Task 분해, UT 정의, ID 채번)

/mysdd.implementation
  ↓ (구현, UT 작성)

/mysdd.matrix (언제든)
  ↓ (추적성 확인)

/mysdd.review (생각날 때)
  ↓ (품질 개선 항목 식별)
```

---

## 11. 주요 결정 사항 요약

### 11.1 설계 결정

| 주제 | 결정 | 이유 |
|------|------|------|
| 계층 구조 | 2계층 | 3계층은 복잡, 실용성 낮음 |
| Master Spec | 필수 | 전체 앱 목적 정의 필요 |
| BDD ID | 없음 | 현업 점검용, Matrix 불필요 |
| UT 필수 | 코드만 | 설정/문서는 선택 |
| Matrix | 자동 생성 | LLM이 문서 스캔하여 생성 |
| Review | 수동, 선택적 | 생각날 때 실행 |

### 11.2 철학 결정

| 주제 | 결정 | 근거 |
|------|------|------|
| 문서-코드 관계 | 문서 = Source of Truth | SDD 철학 |
| 리팩토링 정책 | 정책 없음, 유연 대처 | 실무 현실 반영 |
| 문서 수정 | 언제든 가능 | 코드 변경 시 동기화 필수 |
| 작은 기능 | 요구사항이면 Spec 수정 | 비즈니스 vs 기술 구분 |

### 11.3 반론과 수용

#### Randy의 제안 중 수정된 것들:

1. **3계층 → 2계층**
   - Claude 비판: 복잡도 vs 실용성
   - Randy 수용: "맞아, 단순화하자"

2. **Registry 수동 관리**
   - Claude 제안: Central Registry
   - Randy 반려: "힘듦, strict하게 갈 수 없어"

3. **문서 = 히스토리**
   - Claude 제안: 문서 Frozen, 리팩토링은 별도 Feature
   - Randy 반려: "SDD 철학 위배, Spec이 Source of Truth여야 해"

#### Claude의 제안 중 채택된 것들:

1. **Master Spec 필요성** ✅
2. **2계층 구조** ✅
3. **BDD ID 불필요** ✅
4. **작은 기능 처리 기준** ✅
5. **Review 디렉토리 구조** ✅

---

## 12. 다음 단계

**Randy**:
> "내일 회사 LLM과 구체적으로 구현할 거야."

**구현 필요 항목**:
1. 각 단계별 템플릿 파일
   - `master-spec-template.md`
   - `feature-spec-template.md`
   - `plan-template.md`
   - `task-template.md`

2. 커맨드 파일
   - `commands/mysdd.master-spec.md`
   - `commands/mysdd.feature-spec.md`
   - `commands/mysdd.plan.md`
   - `commands/mysdd.task.md`
   - `commands/mysdd.implementation.md`
   - `commands/mysdd.matrix.md`
   - `commands/mysdd.review.md`

3. Shell 스크립트
   - Feature 브랜치 생성
   - ID 자동 채번
   - 디렉토리 구조 생성

---

## 13. 핵심 통찰 (대화에서 얻은 교훈)

### 13.1 SDD의 본질

- **Spec = Source of Truth**가 핵심
- "문서는 옛날, 코드는 현재" = SDD 아님
- 문서와 코드는 항상 동기화되어야 함

### 13.2 단순함의 가치

- 복잡한 시스템은 팀 적용 어려움
- 3계층 → 2계층으로 단순화
- Registry 수동 관리 같은 오버헤드 제거

### 13.3 이상 vs 현실

- 이상: 완벽한 사전 설계, Registry, 엄격한 규칙
- 현실: 병렬 개발, 요건 변경, 유연성 필요
- 해법: 유연한 정책, 사후 정리 (Review)

### 13.4 AI의 한계

- AI는 다른 브랜치 접근 어려움
- "의미적 동일성" 판별 어려움 (UserAccount vs User)
- Main 브랜치만 체크 가능 → 50% 커버
- 사람의 판단 필요

### 13.5 문서의 역할

- ❌ 히스토리 기록용 (당초 Claude 제안)
- ✅ 현재 상태 반영 (Randy 수정)
- 요구사항 추적성 (Matrix)
- 팀 커뮤니케이션 도구

---

## 14. 마무리

이 문서는 2024-11-23 약 2시간에 걸친 논의를 기록한 것입니다.

**핵심 성과**:
- ✅ GitHub Spec-Kit 분석 완료
- ✅ 맞춤형 워크플로우 설계 완료
- ✅ 커맨드 체계 확정
- ✅ ID 체계 확정
- ✅ 철학 및 원칙 확립

**다음 회사 LLM에게 전달할 내용**:
1. 이 문서 전체
2. 구현 요청: 템플릿, 커맨드, 스크립트

**Randy의 한마디**:
> "확정된 최종 버전뿐만 아니라 논의 과정도 기록해줘. 반론과 답변 모두."

이 기록이 내일 회사 LLM과의 작업에 도움이 되길 바랍니다! 🚀
