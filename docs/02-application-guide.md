# 02. 적용 가이드

> 목적: 임베디드 개발자가 로컬 LLM 기반 자기점검 루프를 **작게 시작해 안전하게 확장**하도록 돕는 실행 순서.
> 원칙(항상): LLM은 후보 제안, 결정론적 검증기가 판정, 사람이 비가역 변경 승인. report-only부터.

---

## 0. 전제 준비 (Phase 0)

- [ ] 로컬 런타임: Ollama + 코더 모델. 저사양 `qwen2.5-coder:7b`, GPU 여유 시 `qwen3-coder:30b`.
- [ ] 검증기 스택 설치(오프라인 wheelhouse/mirror): 컴파일러/크로스툴체인, `cppcheck`, `clang-tidy`, `semgrep`, Unity/Ceedling, (선택) Renode/QEMU.
- [ ] 저장 경계: 생성 산출물(finding/log/index)은 `build/` 등 ignored 폴더. 원문/민감자료는 Git 밖.
- [ ] 출처 원칙 고정: 모든 finding에 `file:line + source_hash`.

**완료 기준:** `모델 1회 호출 + 컴파일 1회 + 정적분석 1회`가 로컬에서 돈다.

---

## 1. 가장 빠른 ROI — 결함 패턴 자기점검 (역량 B)

신규 결함을 "패턴"으로 바꿔 코드 전체에서 유사 사례를 찾는 것이 가장 즉효다.

**단계**
1. 결함을 구조화(YAML): `id`, `root_cause`, `prevention_rule`, `bad_pattern`(예: "packet field scale/unit이 contract와 불일치").
2. `prevention_rule` → **semgrep 규칙 1개** 또는 ripgrep/clang-AST 질의로 후보 열거.
3. 후보별로 로컬 LLM 판정(제약 생성 JSON): `{verdict: same_class|different|needs_review, evidence, confidence}`.
4. `same_class` 후보에 **회귀 테스트/PoC** 생성 → 유닛테스트로 편입.
5. 결과를 `findings.json`으로 저장, 사람이 검토·승인.

**최소 스켈레톤(개념)**
```text
scan_defect_pattern.py:
  load defect rule → enumerate candidates (semgrep/rg) → for each: ollama judge(JSON)
  → write findings.json (no auto-fix)
```

**Apply/Skip.** 패턴화 가능한 결함류에 적용. 일회성 하드웨어 개체 문제엔 skip.
**주의.** 정적 열거 오탐 많음 → LLM 판정 + 사람 게이트. `verdict=different`도 로그로 남겨 재현율 추적.

---

## 2. 코드 ↔ 사양 일치 점검 (역량 A)

**단계**
1. 요구사항 ↔ C 심볼 링크 구축: StrictDoc `@relation` 마커(또는 trace YAML의 `code_refs`).
2. 작은 범위 PoC: C 파일 1개 + 사양 항목 1개.
3. 로컬 LLM이 사양 문장 → 유닛테스트 어서션 초안 생성(제약 생성).
4. 호스트 컴파일·실행 → 실패 = 사양-코드 불일치 후보 → finding.
5. StrictDoc 커버리지로 **고아 요구(코드 없음) / 고아 코드(요구 없음)** 산출.

**Apply/Skip.** 함수 입도로 검증 가능한 로직에 적용. 타이밍/HW 의존은 시뮬/HIL로.
**주의.** LLM 트레이스는 정밀도 신뢰/재현율 불신. 어서션은 사람이 검토 후 확정.

---

## 3. 자기수정·자기테스트 루프 (역량 C)

**단계**
1. 대상 선정: 호스트 컴파일 가능한 로직 계층(상태기계·파서·계산).
2. 루프 구성(qodo-cover/aider 패턴): 테스트 생성 → 컴파일·실행(서브프로세스) → 실패 로그 피드백 → **테스트만** 패치 → 재실행.
3. 통과 시 커버리지 측정, **mutation testing**으로 "테스트가 진짜 버그를 잡는가" 검증.
4. 프로덕션 코드 수정 제안은 **사람 승인** 후에만 반영.

**Apply/Skip.** 시뮬 가능·타이밍 비의존 부분에 적용. 실장/정밀 타이밍은 자동 루프 금지(HIL).
**주의.** 자동 생성 테스트의 거짓 확신. 무한 재시도 방지(최대 반복 상한). 하드웨어 side effect 자동 실행 절대 금지.

---

## 4. 로컬 LLM 신뢰성 튜닝 (역량 D)

- [ ] 판정/추출 출력은 **JSON 스키마 제약**(Outlines/Instructor/GBNF) — 파싱 실패 0.
- [ ] 컨텍스트는 사양·코드 **span만 RAG 그라운딩** (전체 문서 투입 금지 — lost-in-the-middle).
- [ ] 다단계는 단일책임 호출로 분해. 순수 검색/계산은 코드로.
- [ ] 난이도 라우팅: 쉬운 판정=로컬, 개방형 추론=상위 모델/사람(데이터 경계 준수).

**주의.** 소형모델은 후보 생성기. "없음/문제없음"을 최종 판단으로 신뢰하지 말 것.

---

## 5. 확장 로드맵 (권장 순서)

```
Phase 0  런타임+검증기 스택 green, 출처/경계 원칙 고정
Phase 1  역량 B(결함 패턴 자기점검) PoC — report-only
Phase 2  역량 A(사양-코드 트레이스 + 어서션) 작은 범위
Phase 3  역량 C(자기테스트 루프) — 테스트만 자동, mutation으로 검증
Phase 4  로컬 LLM 신뢰성 튜닝(제약/RAG/라우팅) 전반 적용
Phase 5  CI 통합: git pre-commit/nightly로 스캔 자동 발화 (여전히 report-only)
```

---

## 전 구간 가드레일 (요약)

1. 자동 수정 금지 — finding/제안만, 비가역 변경은 사람 승인.
2. 모든 결론은 출처(file:line·hash / 사양 row / 결함 id / 로그 timestamp)로 귀환.
3. flash/reset/JTAG/디버거 실행은 영구 사람 게이트.
4. LLM "문제없음"을 신뢰 금지(재현율 한계). 생성 테스트는 mutation으로 검증.
5. 오프라인·데이터 경계: wheelhouse 설치, redacted span + 출처만 컨텍스트에.
6. report-only → 신뢰 → write 순으로만 권한 확장.

---

## 체크리스트 (도입 판단)

- [ ] 이 작업에 **결정론적 검증기**가 있는가? 없으면 LLM 출력은 자동 경로에 넣지 않는다.
- [ ] 실패 시 **되돌릴 수 있는가?** 비가역이면 사람 게이트.
- [ ] 데이터가 로컬 경계를 벗어나는가? 벗어나면 redact 또는 중단.
- [ ] 결과가 **출처로 귀환**하는가?
- [ ] 생성 테스트의 **버그 검출력**을 별도로 검증했는가?
