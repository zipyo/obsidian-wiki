---
name: wiki-reads
description: >
  위키 관심사 기반 최신 읽을거리 탐색. 위키를 정밀 스캔하여 관심사 프로파일 + 현재 막힌 문제를
  추출하고, 타겟팅된 웹 검색으로 검증된 읽을거리를 큐레이션한다. 사용자가 최신 논문, 기사,
  블로그를 찾고 싶을 때 반드시 이 스킬을 사용한다.
  트리거: "읽을거리 찾아줘", "최신 논문", "wiki reads", "reading radar", "뭐 읽을까",
  "요즘 뭐 나왔어", "공부할 거 찾아줘"
---

# wiki-reads — 위키 기반 읽을거리 큐레이션

위키의 관심사·진행 상태·막힌 문제를 정밀 분석하여, 정확히 지금 필요한 읽을거리를 웹에서 찾아 큐레이션한다.

## 출력 모델

**축(axis)당 Fresh 1 + Classic 1, 총 6~10개.**

- Fresh = **발행일이 오늘부터 역산 7일 이내**. 해당 축에 7일 이내 적격 항목이 없으면 **슬롯을 비운다** (범위 완화 금지).
- Classic = foundational 원전·책·고전 에세이. **이미 위키(`concepts/` · `entities/` · `references/`)에 인제스트된 것은 후보에서 자동 제외**. 재독·재제안은 하지 않는다.
- 개수 채우기보다 "지금 이 위키를 확장할 가치"가 최우선. 품질 미달이면 슬롯은 비운다.

## Before You Start

1. Read `.env` to get `OBSIDIAN_VAULT_PATH`
2. Read `$OBSIDIAN_VAULT_PATH/index.md` to understand the wiki's scope and structure
3. Read `$OBSIDIAN_VAULT_PATH/log.md` for recent activity context

## 실행 — Phase 1~4 자동, Phase 5~6 사용자 선택

### Phase 1: 위키 프로파일 추출

Explore 에이전트로 위키를 정밀 스캔한다. **목표: 축별 검색 쿼리를 만들기 위한 키워드 추출 + Classic 후보 배제 목록 확보.**

핵심 페이지 스캔 (frontmatter tags/summary + 본문 핵심):
- 현재 진행 중인 프로젝트 페이지 (updated가 최근인 것 우선)
- "이후 전략", "로드맵", "다음 단계" 같은 미래 지향 페이지
- concept 허브 페이지 (학습-플로우, 프레임워크-선택-가이드 등)

추출할 것:
- **관심 축** (3~5개): LLM/AI, 투자, 기타 등 대분류
- **축별 현재 막힌 문제**: 해결 안 된 이슈, 정체 중인 메트릭
- **축별 다음 마일스톤**: "다음에 할 것", "검토 예정"
- **축별 추적 중인 엔티티**: 모델명, 기업명, 기술명 등 고유명사
- **기존 인제스트 목록** (Classic 중복 배제용): `concepts/` · `entities/` · `references/` 페이지 제목 + frontmatter `sources:` 수집

Phase 1에서 위키 전체를 읽지 말 것. index.md + log.md + 핵심 페이지 frontmatter 수준으로 충분하다.

### Phase 2: 축별 쿼리 생성 (Fresh 1 + Classic 1)

각 축에 대해 **쿼리 2개**를 만든다. 축 3~5개면 총 6~10개 쿼리.

**Fresh 쿼리** (축당 1개):
- 축의 "가장 핫한 막힌 문제 또는 다음 마일스톤"을 직격하는 영문 키워드
- 발행일 제한: `after:YYYY-MM-DD` (오늘 - 7일) 또는 "past week" 연산자
- arxiv·공식 블로그·학회·신뢰 뉴스레터 쪽으로 유도하는 site 힌트 포함
- 쿼리에 **검색 의도 한 줄** 첨부 ("왜 이걸 찾는지")

**Classic 쿼리** (축당 1개):
- 축의 foundational 문헌을 겨냥 — 교과서 챕터, seminal paper, 고전 에세이, 업계 레전드의 에세이
- 연도 제한 없음. 배제 목록(Phase 1에서 수집한 기존 인제스트)에 걸리는 저자·제목은 쿼리에서 피함
- 쿼리에 **검색 의도 한 줄** 첨부

쿼리 품질 기준:
- 구체적 키워드 (알고리즘명, 모델명, 벤치마크명, 저자명)
- Fresh는 연도 + 기간 제약 필수, Classic은 "seminal", "original", "foundational", "classic essay" 같은 시그널어
- 검증 가능한 소스가 나올 만한 쿼리

### Phase 3: 웹 서치 + 필터링

WebSearch로 쿼리 일괄 실행. 결과마다 다음 기준으로 필터.

**우선 소스**:
- arxiv.org (논문)
- 공식 블로그 (NVIDIA, HuggingFace, Google, Meta, ByteDance 등)
- 학회 proceeding (ACL, ICML, NeurIPS, COLM 등)
- 신뢰 미디어 (Bloomberg, Power Magazine 등 전문지)
- 검증된 뉴스레터 (Sebastian Raschka, Interconnects 등)

**Fresh 필터 — 엄격**:
- 발행일이 **오늘 - 7일 이내**가 아니면 즉시 탈락 (2주, 1달, 재공유 전부 X)
- 명확한 발행일을 확인할 수 없으면 탈락
- 7일 이내 항목이 0개면 **해당 축 Fresh 슬롯은 비운다**. 범위를 늘리지 말 것.

**Classic 필터 — 중복 배제**:
- Phase 1에서 수집한 기존 인제스트 목록(제목·sources URL)에 걸리면 탈락
- 같은 저자·같은 주제의 다른 원전을 대신 고려 가능 (단, 그것도 배제 목록에 없어야 함)

**공통 제외**:
- Medium 일반 블로그 (저자가 유명하지 않은 경우)
- SEO 스팸성 "Top 10" 리스트
- 날짜 없는 글
- 내용 없는 뉴스 재탕
- "YYYY Outlook" / "Year in Review" 류 시한 만료된 전망 기사
- 이후 논문/이벤트에 의해 명백히 superseded된 1~2년 전 survey·리포트

### Phase 4: 큐레이션 + 제시

축별로 Fresh 1 + Classic 1씩 고른다. **각 축에 2개가 채워지지 않아도 된다 — 억지로 채우지 말 것.**

선택 원칙:
1. **위키 확장 가치 최우선**: 읽고 나서 기존 페이지를 업데이트·분기시킬 여지가 큰가?
2. **현재 막힌 문제 해결 기여도**: Phase 1에서 뽑은 블로커를 건드리는가?
3. 동률이면 사용자 관심 이동 방향(최근 update 잦은 영역)에 가까운 쪽

출력 형식 (축별 블록):
```
═══ [축 이름] ═══

Fresh [YYYY.MM.DD]
N. 제목 (한국어 번역)
→ URL
위키 연결: [[관련-페이지]] — 이 항목이 풀어주는 막힌 문제 / 확장 포인트 한 줄

Classic [YYYY]
N. 제목 (한국어 번역)
→ URL
위키 연결: [[관련-페이지]] — 이 원전이 축의 어떤 기초를 메우는지 한 줄
```

- 슬롯이 비면 해당 헤더 아래 `— 7일 이내 적격 항목 없음` 또는 `— 배제 목록 외 Classic 후보 없음`으로 명시 (숨기지 말 것)
- 최종 한 줄 요약: "이번 주 우선 추천: N번 → N번 (이유)" — 전체에서 1~2개만
- 텔레그램 메시지 4096자 제한 시 축별로 분할 전송

### Phase 5: 읽기 + 번역 (사용자 선택)

사용자가 "N번 읽자", "순서대로 읽자" 등으로 요청하면 실행.

각 항목에 대해:
1. WebFetch로 원문 전체 추출 (HTML 버전 우선, PDF는 abstract 페이지 경유)
2. 한국어로 번역하며 요약:
   - 한 줄 요약
   - 핵심 방법/발견 (수식, 테이블 포함)
   - 실험 결과
   - **위키 프로젝트 적용 포인트** (가장 중요 — "이걸 어디에 쓸 수 있는지")
3. 텔레그램으로 전달

규칙:
- 기술 용어는 영어 유지 (GRPO, advantage, gradient 등)
- 수식은 텍스트로 표현
- 논문이면 섹션별 구조 유지, 블로그면 핵심만
- 사용자 질문에 쉬운 비유로 답변

### Phase 6: 위키 인제스트 (사용자 선택)

사용자가 "인제스트해", "위키에 넣어" 등으로 요청하면 실행.

각 읽은 항목에 대해:
1. `$OBSIDIAN_VAULT_PATH/concepts/` 아래에 위키 페이지 생성
   - frontmatter: title, category, tags, summary, sources, created, updated, provenance
   - 본문: 핵심 내용 + 실험 결과 + 적용 가능성
   - 관련 페이지: 기존 위키 페이지와 cross-link
2. 관련 기존 페이지에 역방향 링크 추가 (허브 페이지, 전략 페이지 등)
3. `index.md` Concepts 섹션에 추가
4. 커밋은 별도 요청 시에만

## 인자

- `/wiki-reads` — 전체 축 스캔 (축당 Fresh 1 + Classic 1)
- `/wiki-reads llm` — LLM/AI 축만
- `/wiki-reads 투자` — 투자 축만
- `/wiki-reads grpo` — 특정 토픽만 (단일 축으로 간주, Fresh 1 + Classic 1)

## 주의사항

- Fresh 7일 기준은 **오늘 날짜 기준**으로 계산. 세션 시작 시 현재 날짜를 반드시 확인할 것.
- 쿼리 재검색은 Fresh 슬롯이 빈 경우가 아니라 **쿼리 자체가 부실한 경우**에만 허용 (최대 1회 리파인).
- 텔레그램 채널로 응답 시 메시지 길이 제한 고려 (4096자 이하로 분할).
