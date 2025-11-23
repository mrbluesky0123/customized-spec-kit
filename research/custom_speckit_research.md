# Custom Spec-Kit 연구 노트

**날짜**: 2025-11-22
**목적**: spec-kit 분석 및 간소화된 커스텀 버전 설계

---

## 1. Spec-Kit의 동작 원리

### 1.1 슬래시 커맨드 시스템

**핵심 발견**: spec-kit은 슬래시 커맨드를 "실행"하지 않는다. 단지 **파일을 생성**할 뿐이다.

```
spec-kit의 역할: 템플릿 파일 생성
    ↓
AI 에이전트의 역할: 파일 읽고 실행
```

#### 동작 흐름:

1. **초기화 단계** (`uvx specify-cli init`)
   - GitHub Release에서 템플릿 zip 다운로드
   - `.claude/commands/` 디렉토리 생성
   - 커맨드 파일들 복사 (예: `speckit.specify.md`)

2. **에이전트 로딩**
   ```
   Claude Code 시작
       ↓
   .claude/commands/ 디렉토리 스캔
       ↓
   *.md 파일들의 YAML frontmatter 파싱
       ↓
   커맨드 목록 구성:
     - 파일명 → 커맨드 이름 (/speckit.specify)
     - description → 자동완성 설명
   ```

3. **사용자 실행**
   ```
   사용자: /speckit.specify Build a photo app
       ↓
   Claude Code: speckit.specify.md 파일 전체를 프롬프트로 읽음
       ↓
   LLM: 파일 내용에 따라 작업 수행
   ```

### 1.2 YAML Frontmatter 처리

**YAML Frontmatter = 마크다운 파일 맨 위의 메타데이터**

```markdown
---
description: Create or update the feature specification
scripts:
  sh: scripts/bash/create-new-feature.sh --json "{ARGS}"
  ps: scripts/powershell/create-new-feature.ps1 -Json "{ARGS}"
---

## User Input
실제 AI가 실행할 지시사항...
```

#### 처리 시점:

| 시점 | 처리자 | 용도 |
|------|--------|------|
| **빌드 타임** | Python/Bash 스크립트 | `scripts:` 추출 → `{SCRIPT}` 플레이스홀더 치환 |
| **런타임** | Claude Code | `description` 추출 → 자동완성 메뉴 표시 |
| **런타임** | LLM (Claude) | 마크다운 본문만 사용 (YAML은 주로 무시) |

#### 빌드 과정 예시:

**빌드 전** (템플릿):
```markdown
---
description: Create spec...
scripts:
  sh: scripts/bash/create-new-feature.sh --json "{ARGS}"
---
Run {SCRIPT} and parse JSON...
User input: $ARGUMENTS
```

**빌드 후** (`.claude/commands/`에 배포):
```markdown
---
description: Create spec...
---
Run .specify/scripts/bash/create-new-feature.sh --json "$ARGUMENTS" and parse JSON...
User input: $ARGUMENTS
```

**치환 규칙**:
- `{SCRIPT}` → `.specify/scripts/bash/create-new-feature.sh --json`
- `{ARGS}` → `$ARGUMENTS` (Markdown) 또는 `{{args}}` (TOML)
- `__AGENT__` → `claude`, `gemini` 등
- `scripts/` → `.specify/scripts/` (경로 재작성)
- `scripts:` 섹션 → 제거됨

---

## 2. Shell 스크립트의 역할

### 2.1 왜 Shell 스크립트가 필요한가?

**LLM이 직접 못하는 "기계적 작업"을 수행**

| 스크립트 | 역할 | LLM이 못하는 이유 |
|---------|------|------------------|
| `create-new-feature.sh` | Git 브랜치 생성, 파일 초기화 | Git 명령 실행 불가 |
| `setup-plan.sh` | 디렉토리 구조 생성 | 복잡한 파일 시스템 작업 |
| `check-prerequisites.sh` | 상태 체크, 경로 반환 | Git 상태 확인 필요 |
| `update-agent-context.sh` | 컨텍스트 파일 업데이트 | 에이전트별 위치 다름 |

### 2.2 실행 흐름 예시

```
사용자: /speckit.specify Add user authentication
    ↓
Claude Code가 speckit.specify.md 읽음
    ↓
마크다운 지시사항: "Run .specify/scripts/bash/create-new-feature.sh --json 'Add user authentication'"
    ↓
Bash 도구로 스크립트 실행:
  - git checkout -b 001-user-auth
  - mkdir specs/001-user-auth
  - cp spec-template.md specs/001-user-auth/spec.md
  - echo '{"BRANCH_NAME":"001-user-auth","SPEC_FILE":"/path/to/spec.md"}'
    ↓
Claude가 JSON 파싱:
  - BRANCH_NAME = "001-user-auth"
  - SPEC_FILE = "/path/to/spec.md"
    ↓
마크다운 지시사항 계속 실행:
  - spec-template.md 읽음
  - 사용자 요구사항 분석
  - spec.md 작성
```

### 2.3 create-new-feature.sh의 주요 기능

1. **브랜치 번호 자동 계산**
   - Remote 브랜치 체크
   - Local 브랜치 체크
   - `specs/` 디렉토리 체크
   - 가장 높은 번호 + 1

2. **브랜치명 생성**
   - 불용어 제거 (a, the, to, for 등)
   - 2-4단어로 축약
   - 형식: `001-user-auth`

3. **파일 시스템 준비**
   - Git 브랜치 생성
   - 디렉토리 생성
   - 템플릿 복사

4. **JSON 출력** (LLM이 파싱)
   ```json
   {
     "BRANCH_NAME": "001-user-auth",
     "SPEC_FILE": "/abs/path/specs/001-user-auth/spec.md",
     "FEATURE_NUM": "001"
   }
   ```

---

## 3. 다중 에이전트 지원

### 3.1 왜 빌드 과정이 복잡한가?

**spec-kit은 14개 AI 에이전트를 지원하기 때문**

각 에이전트마다:
- 파일 형식이 다름 (Markdown vs TOML)
- 인자 문법이 다름 (`$ARGUMENTS` vs `{{args}}`)
- 디렉토리 위치가 다름
- 스크립트 종류가 다름 (Bash vs PowerShell)

### 3.2 에이전트별 차이점

| 에이전트 | 디렉토리 | 형식 | 인자 문법 | 스크립트 |
|---------|---------|------|----------|---------|
| Claude Code | `.claude/commands/` | Markdown | `$ARGUMENTS` | Bash/PS |
| Gemini CLI | `.gemini/commands/` | TOML | `{{args}}` | Bash/PS |
| GitHub Copilot | `.github/agents/` | Markdown | `$ARGUMENTS` | - |
| Cursor | `.cursor/commands/` | Markdown | `$ARGUMENTS` | Bash/PS |
| Windsurf | `.windsurf/workflows/` | Markdown | `$ARGUMENTS` | Bash/PS |

### 3.3 같은 형식을 쓰는 에이전트

**Claude Code와 GitHub Copilot**:
- 파일 형식: 둘 다 Markdown
- 인자 문법: 둘 다 `$ARGUMENTS`
- **차이점**: 디렉토리 위치만 다름

**결론**: 같은 파일을 두 디렉토리에 복사하면 됨!

```bash
cp commands/myflow.feature.md .claude/commands/
cp commands/myflow.feature.md .github/agents/
```

---

## 4. 커스텀 Spec-Kit 설계

### 4.1 핵심 통찰

1. **spec-kit은 템플릿 생성 도구일 뿐** - 실행은 AI 에이전트가 담당
2. **빌드 과정은 다중 에이전트 지원 때문** - 단일 에이전트만 쓴다면 불필요
3. **하드코딩이 더 간단** - 동적 정보가 없으므로 직접 작성 가능
4. **Shell 스크립트는 필수** - LLM이 못하는 Git/파일 작업 담당

### 4.2 최소 구조

```
my-workflow/
├── setup.sh                    # 템플릿 설치 (cp 명령만)
├── commands/                   # 커맨드 정의
│   ├── myflow.feature.md
│   ├── myflow.plan.md
│   └── myflow.review.md
├── templates/                  # 문서 템플릿
│   ├── feature-spec.md
│   └── review-checklist.md
└── scripts/                    # 헬퍼 스크립트
    ├── create-feature.sh       # Git/파일 작업
    └── common.sh               # 유틸리티
```

### 4.3 커맨드 파일 구조 (commands/myflow.feature.md)

```markdown
---
description: Create a new feature specification
---

## Execution Steps

1. **Run setup script**:
   Execute `.myflow/scripts/create-feature.sh --json "$ARGUMENTS"`
   and parse JSON output for FEATURE_DIR and SPEC_FILE.

2. **Load template**:
   Read `.myflow/templates/feature-spec.md`.

3. **Analyze user input**:
   Extract key requirements from the feature description.

4. **Generate specification**:
   Fill the template with:
   - Feature overview
   - User scenarios
   - Acceptance criteria

5. **Write to file**:
   Save to SPEC_FILE.

6. **Report completion**:
   Show branch name and file path.
```

### 4.4 간소화된 Shell 스크립트 (scripts/create-feature.sh)

```bash
#!/bin/bash
set -euo pipefail

FEATURE_DESC="$1"
REPO_ROOT=$(git rev-parse --show-toplevel)

# 간단한 브랜치명 (spec-kit처럼 복잡할 필요 없음)
BRANCH_NAME="feature-$(date +%Y%m%d-%H%M%S)"

# Git 브랜치 생성
git checkout -b "$BRANCH_NAME"

# 디렉토리 생성
FEATURE_DIR="$REPO_ROOT/features/$BRANCH_NAME"
mkdir -p "$FEATURE_DIR"

# 템플릿 복사
cp "$REPO_ROOT/.myflow/templates/feature-spec.md" \
   "$FEATURE_DIR/spec.md"

# JSON 출력 (LLM이 파싱)
echo "{\"BRANCH_NAME\":\"$BRANCH_NAME\",\"SPEC_FILE\":\"$FEATURE_DIR/spec.md\",\"FEATURE_DIR\":\"$FEATURE_DIR\"}"
```

### 4.5 설치 스크립트 (setup.sh)

```bash
#!/bin/bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(pwd)"

echo "🚀 Installing workflow templates..."

# 디렉토리 생성
mkdir -p "$PROJECT_ROOT/.claude/commands"
mkdir -p "$PROJECT_ROOT/.github/agents"
mkdir -p "$PROJECT_ROOT/.myflow/templates"
mkdir -p "$PROJECT_ROOT/.myflow/scripts"

# 커맨드 파일 복사 (Claude + Copilot 지원)
cp -r "$SCRIPT_DIR/commands/"* "$PROJECT_ROOT/.claude/commands/"
cp -r "$SCRIPT_DIR/commands/"* "$PROJECT_ROOT/.github/agents/"

# 템플릿 & 스크립트 복사
cp -r "$SCRIPT_DIR/templates/"* "$PROJECT_ROOT/.myflow/templates/"
cp -r "$SCRIPT_DIR/scripts/"* "$PROJECT_ROOT/.myflow/scripts/"

# 실행 권한 부여
chmod +x "$PROJECT_ROOT/.myflow/scripts/"*.sh

echo "✅ Installation complete!"
echo ""
echo "Next steps:"
echo "  1. Start Claude Code (or GitHub Copilot)"
echo "  2. Type /myflow.feature to create a new feature"
```

### 4.6 배포 (build.sh)

```bash
#!/bin/bash
set -euo pipefail

VERSION="${1:-v1.0.0}"
OUTPUT="my-workflow-template-${VERSION}.zip"

echo "📦 Building package: $OUTPUT"

# 필수 파일 검증
required_files=(
  "setup.sh"
  "commands/myflow.feature.md"
  "templates/feature-spec.md"
  "scripts/create-feature.sh"
)

for file in "${required_files[@]}"; do
  if [[ ! -f "$file" ]]; then
    echo "❌ Missing required file: $file"
    exit 1
  fi
done

# ZIP 생성
zip -r "$OUTPUT" \
  setup.sh \
  commands/ \
  templates/ \
  scripts/ \
  README.md \
  -x "*.DS_Store" \
  -x "__pycache__/*"

echo "✅ Package created: $OUTPUT"
echo "📊 Size: $(du -h "$OUTPUT" | cut -f1)"
```

---

## 5. Spec-Kit vs 커스텀 버전 비교

| | Spec-Kit | 커스텀 버전 |
|---|----------|------------|
| **복잡도** | Python CLI, GitHub API, 빌드 스크립트 | Bash 스크립트만 |
| **에이전트 지원** | 14개 | Claude (+ Copilot 선택) |
| **의존성** | Python, httpx, rich, typer | Bash, Git |
| **커스터마이징** | 어려움 (빌드 재실행 필요) | 쉬움 (파일 수정 후 cp) |
| **배포** | GitHub Release + zip | zip만 |
| **유지보수** | 복잡 | 간단 |

---

## 6. 핵심 설계 원칙

### 필수 요소:
✅ 커맨드 `.md` 파일 (YAML frontmatter + 마크다운 지시사항)
✅ Shell 스크립트 (Git/파일 작업)
✅ 템플릿 파일 (LLM이 채울 구조)
✅ 설치 스크립트 (`setup.sh`)

### 선택 요소:
❌ Python CLI (Bash로 충분)
❌ 복잡한 브랜치 번호 계산 (날짜/시간으로 대체 가능)
❌ 다중 에이전트 지원 (Claude만 쓴다면 불필요)
❌ PowerShell 버전 (Linux/Mac만 쓴다면 불필요)
❌ GitHub API 연동 (직접 zip 배포로 충분)

---

## 7. 다음 단계

1. ✅ Spec-Kit 분석 완료
2. ⬜ 커스텀 워크플로우 정의 (우리 팀 요구사항)
3. ⬜ 최소 기능 커맨드 3개 작성
4. ⬜ Shell 스크립트 구현
5. ⬜ 템플릿 파일 작성
6. ⬜ 테스트 및 검증
7. ⬜ 팀 배포

---

## 8. 참고 자료

- Spec-Kit 원본: https://github.com/github/spec-kit
- Claude Code 문서: https://docs.anthropic.com/en/docs/claude-code
- 슬래시 커맨드 문법: YAML frontmatter (Jekyll 스타일)

---

**작성자 노트**:
- spec-kit의 핵심 가치는 "검증된 SDD 워크플로우"
- 복잡한 빌드 시스템은 다중 에이전트 지원 때문
- 단일 팀/단일 에이전트라면 **훨씬 단순한 버전**이 더 실용적
- 파일 기반 시스템이라 커스터마이징이 쉬움
