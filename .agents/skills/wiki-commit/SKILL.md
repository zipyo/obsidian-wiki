---
name: wiki-commit
description: >
  Obsidian 위키 볼트의 변경사항을 분석하여 커밋하고 푸시한다.
  "위키 커밋", "볼트 커밋", "wiki commit", "변경사항 커밋해줘", "위키 푸시" 등의 요청에 사용.
  위키 작업(ingest, lint, update, distill 등) 완료 후 변경 내역을 정리할 때 활용한다.
---

# Wiki Commit — 볼트 변경사항 커밋 & 푸시

위키 볼트(OBSIDIAN_VAULT_PATH)의 git 변경사항을 분석하고, 의미 있는 커밋 메시지를 생성하여 커밋 후 원격에 푸시한다.

## 1. 볼트 경로 확인

`.env` 또는 `~/.obsidian-wiki/config`에서 `OBSIDIAN_VAULT_PATH`를 읽는다.
볼트 경로를 찾지 못하면 사용자에게 알리고 중단.

## 2. 변경사항 분석

볼트 디렉토리에서 아래 명령을 **병렬** 실행:

```bash
git status --short              # 변경 파일 목록
git diff --stat                 # 변경 규모 요약
git log --oneline -5            # 최근 커밋 스타일 참조
```

### 변경 분류 기준

변경된 파일 경로로 카테고리를 분류한다:

| 경로 패턴 | 카테고리 |
|-----------|---------|
| `entities/companies/` | 기업 엔티티 |
| `entities/investors/` | 투자자 엔티티 |
| `entities/*/` | 기타 엔티티 |
| `concepts/` | 개념 |
| `synthesis/` | 종합 분석 |
| `references/` | 레퍼런스 |
| `projects/` | 프로젝트 |
| `journal/` | 저널 |
| `.manifest.json` | 매니페스트 |
| `index.md`, `log.md` | 인덱스/로그 |
| `.obsidian/` | Obsidian 설정 (커밋은 하되 메시지에서 생략) |

### 변경 유형 판별

파일 상태 코드로 신규/수정/삭제를 구분한다:
- `?? ` 또는 `A ` → 신규
- ` M` 또는 `M ` → 수정
- ` D` 또는 `D ` → 삭제

## 3. 커밋 메시지 생성

### 형식

```
[타입]: 핵심 요약

- 카테고리별 변경 내역 (수치 포함)
```

### 타입 선정

| 상황 | 타입 |
|------|------|
| 새 페이지 추가가 주 | `feat` |
| 기존 페이지 내용 보강/수정 | `update` |
| 구조 변경, 태그 정리, 링크 정비 | `refactor` |
| 오류 수정, 깨진 링크 복구 | `fix` |
| 혼합 (판단 어려움) | `chore` |

### 메시지 작성 원칙

- 한국어로 작성. 기술 용어(entity, ingest 등)는 영어 허용
- 첫 줄은 **무엇이 바뀌었는지** 한 문장으로 — 70자 이내 권장
- 본문에 카테고리별 변경을 bullet으로 정리
- `.obsidian/` 변경은 본문에서 생략
- AI/Claude 관련 문구 절대 금지 (Co-Authored-By, Generated with 등)

### 메시지 예시

```
feat: 2026 주총 35개사 distill → 기업 엔티티 머지

- 기업 엔티티 35개 수정 (주총 핵심 섹션 추가)
- 주총 개별 파일 35개 삭제
- synthesis/2026-주주총회-후기.md 참조 업데이트
- .manifest.json 이력 갱신
```

```
update: k-berkshire 기업 분석 11개사 추가

- entities/companies/ 신규 11개
- concepts/기업-분석-방법론.md 수정
- index.md, log.md 갱신
```

## 4. 커밋 & 푸시

```bash
# 1) 모든 변경 스테이징 (.obsidian/ 포함)
git add -A

# 2) 커밋 (HEREDOC으로 메시지 전달)
git commit -m "$(cat <<'EOF'
[타입]: 요약

- 상세 내역
EOF
)"

# 3) 푸시
git push origin HEAD
```

### 예외 처리

- **변경 없음**: "커밋할 변경사항이 없습니다" 알림 후 종료
- **push 실패**: 에러 메시지를 사용자에게 보여주고, pull --rebase 등 해결 방안 제안
- **커밋 전 확인**: 커밋 메시지 초안을 사용자에게 보여주고 승인 후 실행

## 5. 결과 보고

커밋 완료 후 간단히 보고:

```
커밋 완료: abc1234
[타입]: 요약 메시지
변경: N개 파일 (추가 A, 수정 M, 삭제 D)
```
