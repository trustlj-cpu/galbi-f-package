# BASE_FILL BOUNDARY + FLOOR QA TOLERANCE — LEAD ARCHITECT DECISION (Claude Code · 2026-10-06)

[ROOT CAUSE]
두 blocker 모두 **명세 공백**이다. ① v1.4 D1이 "JAW 내면"이라는 존재하지 않는 기하를 경계 체인에 넣었다 — 정중앙 MRI에는 하악 내면이 별도 경계로 보이지 않고, 어느 AMENDMENT도 그것을 정의하지 않았다. Codex가 바깥 피부선으로 대체하지 않은 것이 맞다. ② v1.4 D3의 "3mm"는 **측정 정밀도 없이** 적힌 수치다. rtMRI 픽셀 간격은 약 2.4mm라 3.000 vs 3.003의 차이(0.003mm = 픽셀의 1/800)는 물리적으로 아무 의미가 없는데, 허용 오차와 백분위 계산 규약을 명시하지 않아 Codex가 FAIL 처리할 수밖에 없었다. 윤곽 수정 대상 없음.

[JAW INNER BOUNDARY DECISION]
- required: **NO.** TONGUE_BASE_FILL은 혀 아래·뒤 빈 공간을 **가리는** 용도라, 아래 경계는 화면에서 혀·실루엣·구강저 선 아래에 숨겨진다. 해부학적 "턱 내면"을 만들 이유가 없다.
- status: 위 경계 = **DERIVED**(REST LOWER 호 오프셋), 아래 체인 = **VIRTUAL**(규약, 라벨 기록).
- construction rule: 위 경계 U = REST_PROXY LOWER 호 정점 1…63을 각 점의 바깥 법선(아래쪽) 방향으로 2mm 이동한 폴리라인 U1…U63(자기교차가 생기면 그 구간은 2mm 대신 1mm). 아래 체인은 기존 기하 끝점만 쓴다: A = FLOOR_LINE 앞쪽 끝점(OBSERVED), P = PHARYNX_WALL 최하단 점(OBSERVED), D = 25mm(VIRTUAL 깊이 — 성인 턱밑 연조직 두께 근사; HEAD_SILHOUETTE 안쪽으로 25mm를 넘으면 실루엣 안쪽 1mm 지점까지로 자름).
- endpoints: U1(혀끝 아래, 정점 1 오프셋) · U63(ROOT_TURN 직전, 정점 63 오프셋) · P · P′=(P.x, P.y−D) · A′=(A.x, A.y−D) · A.
- BASE_FILL boundary chain(닫힌 다각형, 시계 방향, 총 63+5점): **U1 → U2 → … → U63 → P → P′ → A′ → A → U1.** U63→P와 A→U1은 직선. 폴리곤은 HEAD_SILHOUETTE로 클립(실루엣 밖 부분 제거). 메시: 팬 삼각분할 또는 ear-clipping, 렌더 순서 TONGUE 아래·SKULL_PALATE/HEAD_SILHOUETTE 위. QA 검사 "BASE COVERAGE"는 U만 쓴다(아래 체인은 검사 대상 아님).

[FLOOR QA DECISION]
- threshold: 값은 유지 — REST_PROXY 중앙값 ≤1mm·p90 ≤3mm, 그 외 viseme 중앙값 ≤3mm·p90 ≤6mm.
- tolerance: **모든 OBSERVED 트레이스 기반 거리 threshold에 수치 허용 오차 τ = 0.5 × mm_per_px**(anatomy_tract.json의 scale에서 읽음; 배율 출처가 palate_45mm 규약이면 τ = 1.0mm). 비교식은 `value ≤ threshold + τ`. τ는 qa_rules.json에 `"tolerance_mm"`로 기록하고 보고서에 함께 표시한다. (근거: 픽셀의 절반보다 작은 차이는 트레이스로 구분할 수 없다.) 이 규칙은 FLOOR·ROOT STABILITY·PHARYNX·접촉(≤0.1mm 규칙은 예외 — 접촉은 스냅으로 정의되므로 τ 미적용)에 공통 적용.
- percentile calculation: **nearest-rank, 보간 없음.** 오름차순 정렬 d(1)…d(n)에서 p90 = d(⌈0.9·n⌉). n=28이면 d(26). numpy를 쓰면 `np.percentile(d, 90, method="higher")`로 고정. n < 10이면 p90 대신 최댓값으로 대체하고 보고에 명시.
- current 3.003177mm result PASS/FAIL: **PASS**(τ ≥ 0.05mm인 어떤 배율에서도 3.003 ≤ 3 + τ). 단 Codex는 nearest-rank로 재계산해 값·n·사용한 τ·배율 출처를 보고에 기록한다.

[v1.4 RULES SUPERSEDED]
- D1 "아래 경계 = FLOOR_LINE 앞끝 → JAW 내면 하단 → PHARYNX_WALL 하단" → 폐기 → 위 U1…U63→P→P′→A′→A 체인으로 교체.
- D3 threshold 비교를 `≤ threshold + τ`로 교체, 백분위를 nearest-rank로 고정.
- D2 ROOT STABILITY·BASE COVERAGE, PHARYNX, 접촉·관통·기도 윗면 한정 → 유지(τ 적용 규칙만 추가).

[AMENDMENT v1.5 REQUIRED]
**YES** — MASTER_SPEC_AMENDMENT_v1.5.md(v1.4 D1 경계 체인·D3 비교 규칙 덮어쓰기, §10에 τ·백분위 규약 추가).

[NEXT EXACT ACTION]
Codex가 윤곽 수정 없이 ① BASE_FILL 폴리곤을 위 체인으로 완성(63+5점, 실루엣 클립, anatomy_tract.json에 DERIVED/VIRTUAL 라벨로 저장) ② FLOOR QA를 nearest-rank + τ로 재계산해 n·p90·τ·배율 출처 보고 ③ 그대로 PHASE 1.5 승인 시트 2장(성도 오버레이, REST 퍼펫 렌더 — BASE_FILL 포함) 생성 → **STOP(사람 승인 대기).** 비용 0, Blender는 ③에서 `-b`로만.
