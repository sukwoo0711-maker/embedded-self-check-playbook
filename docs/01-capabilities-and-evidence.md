# 01. 5가지 역량 — 중립 정리와 근거

> **읽는 법.** 각 역량은 *정의 → AS-IS → TO-BE → 근거 → Apply when / Skip when → trade-off* 순서다.
> 도입 권유가 아니라 판단 근거다. 도구 상태·arXiv ID·별점은 조사 시점(2026-07) 기준이며 재확인이 필요하다.

---

## 0. 원리 — "자가치유"가 아니라 "자가점검 루프"

2026년의 self-healing/self-correcting 연구·구현은 예외 없이 동일한 골격으로 수렴한다.

```
LLM 생성/수정 → 결정론적 검증기 실행 → 실패 로그를 다시 LLM에 → 재시도 → (통과) → 사람 승인
                     ↑ load-bearing: 검증기가 없으면 "자신 있는 오답"만 쌓인다
```

- **actor-critic / Reflexion 계열:** 한 LLM이 생성하고 다른 LLM(또는 규칙)이 판정·반성·재계획한다. 하지만 "critic LLM"만으로는 순환 논증이 되기 쉽다 — **ground truth(컴파일·테스트·정적분석)** 가 있어야 루프가 수렴한다.
- **임베디드의 이점:** 웹앱과 달리 임베디드는 결정론적 검증기가 풍부하다 — 크로스 컴파일러, 정적분석(cppcheck/clang-tidy/MISRA), 호스트 유닛테스트(Unity/Ceedling), 명령어 시뮬(Renode/QEMU), UART/HIL 로그. 이 검증기 스택이 자기점검 루프의 실질 엔진이다.

**핵심 명제:** LLM의 역할은 "정답 생성"이 아니라 "검증기가 판정할 수 있는 **후보 생성**"이다.

**근거/참고:** Reflexion, Self-Refine(자기반성형 반복 개선 계열); actor-critic 형태의 self-healing 파이프라인 구현들([Self-healing LLM Pipeline](https://github.com/ammarlodhi255/Self-healing-LLM-Pipeline)). "순진한 LLM 생성 테스트는 미묘하게 틀려 **거짓 확신**을 준다"는 경고가 임베디드 테스트 연구에서 반복된다(§C, §E).

---

## A. 코드 ↔ 사양 일치 자기점검

**정의.** 요구사항/사양과 소스 코드가 일치하는지를, 문장 비교가 아니라 **실행 가능한 검증**으로 확인한다.

**AS-IS.** 사양은 문서/스프레드시트에, 코드는 별도로. 일치 여부는 사람이 리뷰로 확인 → 느리고 누락된다. 요구사항↔코드 링크(추적성)가 없으면 사양 변경이 조용히 코드와 어긋난다.

**TO-BE.**
1. 요구사항 ↔ C 심볼 링크를 **트레이서빌리티 도구**로 유지(오프라인).
2. 로컬 LLM이 사양 문장을 **유닛테스트 어서션**으로 변환(예: `TEST_ASSERT(timeout <= 180)`).
3. 호스트 컴파일·실행 → 통과/실패가 곧 사양-코드 일치 여부.
4. 불일치는 자동 수정이 아니라 **finding**(파일·라인·해시 근거)으로 리뷰 큐에.

**근거.**
- LLM 문서-코드 추적성: 정밀도 높음/재현율 낮음, **함수 단위 추적 F1 97~99%**(C의 함수 입도에 유리). 결론은 "human-in-the-loop 필수". [arXiv 2506.16440](https://arxiv.org/html/2506.16440v1)
- 오프라인 추적성 도구: [StrictDoc](https://github.com/strictdoc-project/strictdoc)(Apache-2.0, 순수 Python, `@relation` 마커 + Tree-sitter C 파싱 + Excel/ReqIF I/O + 커버리지/고아 검출), [OpenFastTrace](https://github.com/itsallcode/openfasttrace)(태그 기반, JRE 필요).
- 규제 도메인은 요구사항↔코드 추적성을 **의무화**한다(자동차 ISO 26262, 항공 DO-178C, 의료 IEC 62304; 가전 제어는 IEC 60730 맥락). 즉 이 역량은 인증 산출물과 직결된다.

**Apply when.** 함수 단위로 검증 가능한 사양이 있고, 호스트 컴파일이 가능한 로직일 때.
**Skip when.** 사양이 타이밍/하드웨어 상호작용에 전적으로 의존해 호스트 테스트로 근사 불가할 때(→ HIL/시뮬 필요).
**Trade-off.** 초기 태깅/트레이스 구축 비용이 크다. LLM은 "링크 있음"은 신뢰, "링크 없음"은 불신(재현율 한계).

---

## B. 신규 결함 → 현재 코드 자기점검 (회귀·전파)

**정의.** 새로 등록된 결함 1건을 규칙/패턴으로 바꿔, **같은 결함류가 코드베이스 다른 곳에도 있는지** 스스로 찾고 회귀 테스트를 만든다.

**AS-IS.** 결함은 결함관리 시스템에 쌓이지만, "같은 실수가 다른 모듈에도 있는가?"는 사람이 수동으로 grep하거나 방치된다. 동일 유형 결함이 재발한다.

**TO-BE.** "One Bug, Hundreds Behind" 패턴:
```
새 결함 → 결함에서 규칙/패턴 추출(예: prevention_rule)
       → 경량 정적 열거로 후보 축소 (semgrep 규칙 / cppcheck custom / clang AST / ripgrep)
       → 로컬 LLM이 후보별 판정 "여기도 같은 결함류인가?" (근거 span 제시)
       → 회귀 테스트/PoC 자동 생성 → eval 데이터셋에 축적
       → finding으로 사람 검토
```
핵심은 **LLM 단독 전수 스캔이 아니라, 규칙+정적 열거가 후보를 좁히고 LLM은 판정만** 하는 것. 소형 로컬 모델의 재현율·비용 문제를 회피한다.

**근거.**
- 알려진 버그로부터 유사 버그를 대규모 발굴: [One Bug, Hundreds Behind (arXiv 2510.14036)](https://arxiv.org/html/2510.14036v1). 규칙 중심·패치 시드 + 경량 정적 열거 + LLM 워크플로(BugStone류), PoC 테스트 자동 생성([AnyPoC, arXiv 2604.11950](https://arxiv.org/html/2604.11950)).
- 코드 클론/취약점 탐지는 토큰 유사도에서 **의미 인지 매칭**으로 이동.
- 정적 열거 엔진: [Semgrep](https://semgrep.dev/)(규칙 기반, 오프라인), [cppcheck](https://cppcheck.sourceforge.io/), clang-tidy.

**Apply when.** 결함이 반복 가능한 코드 패턴(단위/스케일 불일치, 버퍼 경계, 잠금 누락, ISR 재진입 등)으로 표현될 때 — 임베디드 결함의 상당수가 여기 해당.
**Skip when.** 결함이 일회성 하드웨어 개체 문제이거나 패턴화 불가능한 경우.
**Trade-off.** 규칙 작성 초기 비용. 정적 열거는 오탐이 많을 수 있어 LLM 판정 + 사람 게이트가 필요.

---

## C. 자기수정·자기테스트 루프 (로컬 LLM판)

**정의.** 코드/테스트를 생성 → 실행 → 실패 로그를 피드백 → 재시도하는 폐루프. 단, **자기수정 대상은 테스트/설정에 한정**하고 프로덕션 펌웨어 변경은 사람 승인.

**AS-IS.** 유닛테스트 작성이 수작업이라 커버리지가 낮고, 회귀 검출이 느리다.

**TO-BE.** 오프라인 폐루프:
```
파싱/AST → 테스트 생성(Unity/Ceedling) → 호스트 컴파일·실행(서브프로세스)
   → 실패 시 컴파일러/테스트 로그를 LLM에 피드백 → 테스트 패치 → 재실행 → (통과) → 리뷰
```

**근거.**
- Ollama 네이티브 자기수정 테스트 에이전트 참고 구현: [Ghost](https://github.com/tripathiji1312/ghost)(AST → 테스트 생성 → 서브프로세스 실행 → 실패 시 자기 패치; Python 대상이나 패턴은 C로 이식 가능).
- 자동 C 유닛테스트 생성: [SPARC (arXiv 2602.16671)](https://arxiv.org/pdf/2602.16671), 시드 케이스 유도 C 테스트 생성; [EmbC-Test: LLM+RAG로 임베디드 테스트 가속 (arXiv 2603.09497)](https://arxiv.org/html/2603.09497v1).
- 에이전트 기반 펌웨어 검증·패치(퍼징/정적/런타임): [Securing LLM-Generated Embedded Firmware (arXiv 2509.09970)](https://arxiv.org/abs/2509.09970).
- 로컬 루프 도구: [aider](https://aider.chat/)(`--test`로 테스트 통과까지 자동 수정 루프, 로컬 모델 지원), Ghost 패턴 자작.
- 실행 검증기: 하드웨어 없는 시뮬 [Renode](https://renode.io/), [QEMU](https://www.qemu.org/); 호스트 테스트 [Unity/Ceedling](https://www.throwtheswitch.org/).

**Apply when.** 호스트에서 컴파일·실행 가능한 로직 계층(상태기계, 파서, 계산). 시뮬로 대체 가능한 주변장치.
**Skip when.** 실장(real silicon)·정밀 타이밍·아날로그가 본질인 부분 — 자동 루프 금지, HIL로.
**Trade-off.** "생성된 테스트가 실제로 버그를 잡는가?"는 별도 검증(mutation testing) 필요. 순진한 자동 테스트는 **거짓 확신** 위험(§E).

---

## D. 로컬 LLM + 검증기 스택

**정의.** 오프라인 소형 코더 모델을 신뢰 가능한 도구로 만드는 구성.

**AS-IS.** 소형 로컬 모델을 자유생성에 쓰면 형식 드리프트·환각으로 신뢰 불가.

**TO-BE (소형모델 신뢰 4원칙).**
1. **제약 생성** — JSON 스키마/문법 강제(파싱 실패 제거). [Outlines](https://github.com/dottxt-ai/outlines), [Instructor](https://github.com/567-labs/instructor), llama.cpp GBNF.
2. **RAG 그라운딩** — 사양·코드 span을 컨텍스트로 넣어 "recall"을 "summarize"로 전환.
3. **단일책임 분해** — 다단계 프롬프트를 좁은 단일 호출로 쪼갬. 순수 검색/계산은 코드로.
4. **결정론적 검증기 필수** — 컴파일러·정적분석·테스트가 최종 판정. 소형모델은 "후보 생성기"로만.

**검증기 스택(전부 오프라인, 임베디드 C):**
| 계층 | 도구 | 역할 |
|---|---|---|
| 컴파일 | gcc/clang, 크로스 툴체인 | 가장 싼 검증기 |
| 정적분석 | cppcheck, clang-tidy, MISRA 체커 | 규칙 위반·결함 패턴 |
| 유닛테스트 | Unity + Ceedling (+ CMock) | 호스트에서 로직 검증 |
| 시뮬 | Renode / QEMU | 하드웨어 없이 실행 |
| 행동 검증 | UART/로그 파서 | 사양 대비 런타임 프로파일 |

**근거.** 소형모델의 가치는 "더 똑똑하게"가 아니라 "실패 경로 제거"에서 나온다(제약·그라운딩·분해·검증). 임베디드 연구들도 Ollama + Qwen3-Coder 계열 로컬 구성을 사용. 모델은 GPU 여건에 따라 `qwen2.5-coder:7b`(저사양) ↔ `qwen3-coder:30b`(고사양) 선택.

**Apply when.** 데이터가 로컬을 벗어나면 안 되거나(제한망), 비용/오프라인이 중요할 때.
**Skip when.** 개방형 다단계 추론이 정답을 좌우하는 작업 — 이때는 상위 모델로 라우팅하거나 사람이.
**Trade-off.** 최상위 상용 모델 대비 추론 한계. 검증기·라우팅 설계 비용.

---

## E. 규율·가드레일

**정의.** 자율성이 커질수록 실수도 같은 속도로 증폭된다. 이를 막는 운영 원칙.

**원칙.**
1. **자동 수정 금지 — finding/제안만, 되돌릴 수 없는 변경은 사람 승인.**
2. **모든 결론은 출처로 귀환** — 파일:라인·해시, 사양 row, 결함 id, 로그 timestamp 중 하나.
3. **하드웨어 side effect 영구 게이트** — flash/reset/JTAG/디버거 실행은 사람 승인 없이는 절대 자동 실행 금지.
4. **재현율의 함정** — LLM의 "문제없음"을 신뢰하지 말 것(못 잡았을 수 있음).
5. **거짓 확신 차단** — 생성 테스트가 진짜 버그를 잡는지 **mutation testing**/커버리지로 검증.
6. **오프라인·데이터 경계** — 외부 패키지는 wheelhouse/내부 mirror. 원문은 컨텍스트에 redacted span + 출처만.
7. **점진적 권한** — report-only → 신뢰 축적 → write 권한. 처음부터 자동화 금지.

**근거.** "자율성은 실수를 증폭한다"는 self-hosted 자동화의 공통 경고; 임베디드 테스트 연구의 "거짓 확신" 경고; 안전표준의 사람 검토·추적성 요구.

**Apply when.** 항상. 특히 안전 관련·비가역·하드웨어 접촉 작업.
**Skip when.** 없음(순수 읽기·리포트 전용 실험은 완화 가능하나 출처 원칙은 유지).

---

## Sources (조사 시점 2026-07, 재확인 권장)

**루프/자기수정**
- Ghost (Ollama self-healing test agent) — https://github.com/tripathiji1312/ghost
- Self-healing LLM Pipeline — https://github.com/ammarlodhi255/Self-healing-LLM-Pipeline
- aider — https://aider.chat/

**임베디드 테스트/검증 연구**
- Securing LLM-Generated Embedded Firmware (agent 검증·패치) — https://arxiv.org/abs/2509.09970
- EmbC-Test (LLM+RAG 임베디드 테스트 가속) — https://arxiv.org/html/2603.09497v1
- SPARC (자동 C 유닛테스트 생성) — https://arxiv.org/pdf/2602.16671

**결함 전파/회귀**
- One Bug, Hundreds Behind — https://arxiv.org/html/2510.14036v1
- AnyPoC (PoC 테스트 생성) — https://arxiv.org/html/2604.11950
- Semgrep — https://semgrep.dev/ · cppcheck — https://cppcheck.sourceforge.io/

**사양-코드 추적성**
- LLM Doc-to-Code Traceability — https://arxiv.org/html/2506.16440v1
- StrictDoc — https://github.com/strictdoc-project/strictdoc · OpenFastTrace — https://github.com/itsallcode/openfasttrace

**로컬 LLM 신뢰성**
- Outlines — https://github.com/dottxt-ai/outlines · Instructor — https://github.com/567-labs/instructor
- Unity/Ceedling — https://www.throwtheswitch.org/ · Renode — https://renode.io/ · QEMU — https://www.qemu.org/
