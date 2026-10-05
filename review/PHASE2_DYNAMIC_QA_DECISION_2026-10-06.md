# PHASE 2 DYNAMIC VISEME QA — LEAD ARCHITECT DECISION (Claude Code · 2026-10-06)

[ROOT CAUSE]
PHASE 1.5용 **정적 QA**(REST 대비 같은 인덱스 ROOT 안정성, FLOOR 부착, REST 기반 BASE coverage)를 PHASE 2에 **그대로 상속**시킨 내 설계 오류다. 그 세 검사는 "REST 윤곽이 성도 안에 제대로 놓였는가"를 묻는 것이지 "이 음소의 조음이 맞는가"를 묻는 것이 아니다. 혀가 실제로 움직여야 하는 /k a l i/는 정의상 REST와 달라지므로 전부 탈락하고, 혀가 REST에 머무는 /p/만 통과한 것은 **QA가 REST 유사도를 재고 있다는 직접 증거**다. 여기에 ① 접촉을 vertex 0 하나로만 검사(ㄹ 접촉은 혀끝·혀날 영역 어딘가에서 일어남) ② 접촉 음소에서 "구개 겹침 = 관통 FAIL"(MRI 트레이스에서 접촉은 0~1.5mm 겹침으로 나타남) 두 규칙이 더해져 /l/·/k/가 추가 탈락했다. 참조 프레임·관측 기하의 문제가 아니다(514프레임 추가 탐색이 효과 없었던 이유).

[PHASE 2 QA PRINCIPLE]
"REST와 비슷한가" → **"같은 위상·대응을 유지하면서, 그 음소의 관측된 조음 관계를 보존하고, 화면에서 안전하게 그려지는가."** QA를 세 층으로 분리한다.
- **S. STRUCTURAL(hard, 전 viseme 공통)**: 128점·시계 방향·v0=TIP·v64=ROOT_TURN·호별 재표본·자기교차 0·같은 화자/배율/원점·윗면이 아랫면 아래로 뒤집히지 않음(각 x에서 upper y ≥ lower y).
- **A. ARTICULATORY(hard, 음소별 정의 관계)**: 그 음소를 그 음소로 만드는 관계 1~2개만. 같은 화자의 소스 프레임에서 보이는 관계를 재현하는지 측정.
- **D. DISPLAY SAFETY(렌더 검사, anatomy 게이트 아님)**: 렌더 마스크에서 혀 아래·뒤에 배경이 노출되지 않음. 측정이 아니라 그림 검사.
- **P. PLAUSIBILITY CEILING(보고만, 게이트 아님)**: REST 대비 ROOT 이동·면적 변화를 기록해 트레이스 사고(엉뚱한 프레임·뒤집힘)만 잡는다 — 상한 초과(아래)일 때만 FAIL.

[ROOT]
- hard gate: **NO**(PHASE 2에서 제거).
- revised rule: PLAUSIBILITY — ROOT 집합(54–75)의 REST 대비 같은 인덱스 변위 **중앙값 ≤25mm**(그 이상은 혀가 아니라 다른 프레임/구조를 추적한 것이므로 FAIL), 그 외는 수치만 보고. /a/ 10~11mm, 인두 수축·설골 이동은 정상 범위로 허용.

[FLOOR]
- hard gate: **NO**(REST_PROXY에만 유지 — PHASE 1.5 동결 규칙 그대로).
- revised rule: 동적 viseme에는 바닥 부착 조건 없음. 구조 검사 S의 "뒤집힘 없음"만 적용. /i/에서 아랫면이 바닥에서 8mm 떨어지는 것은 혀 몸이 올라가며 구강저도 함께 올라가는 정상 현상 — 화면에서는 D(아래)로 가린다.

[BASE COVERAGE]
- anatomy gate / display-only: **DISPLAY-ONLY.** 정적 REST 기반 BASE_FILL coverage는 viseme PASS/FAIL에서 **제외**. 대신 표시 규칙 하나를 바꾼다: **BODY_TISSUE_DISPLAY의 위 경계를 "구강저 선"이 아니라 "현재 프레임의 혀 LOWER 호(1–63)를 0.5mm 아래로 오프셋한 곡선"으로 매 프레임 갱신**(DISPLAY, DERIVED-from-animation). 그러면 혀가 어디로 가든 아래·뿌리 가장자리는 항상 조직에 가려지고 빈틈이 생기지 않는다(혀 아래 공간이 혀와 함께 올라가는 실제 해부와도 맞다). TONGUE_BASE_FILL은 **폐기(deprecated)**: 오브젝트는 두되 어떤 QA에도 쓰지 않는다. D 검사 = 렌더 후 마스크에서 구강 영역 안 혀 LOWER 호 아래쪽 배경 픽셀 0.

[/l/ CONTACT]
- exact semantic region: **CONTACT SET S_l = {0} ∪ 104–127(blade+tip_upper) ∪ 1–10(tip_under)**. 접촉 대상 = PALATE_LINE 중 ALVEOLAR_POINT에서 호 길이 ±8mm 구간(치조 구간).
- contact rule: `min_{v∈S_l} dist(v, 치조 구간) ≤ 1.0mm + τ` 이고 최근접 구개점이 치조 구간 안에 있으면 PASS. vertex 0 단독 조건 **폐기**. 겹침(혀 점이 구개 선 위로 ≤1.5mm + τ)은 **접촉으로 해석**(렌더에서 선에 스냅). 설측 해제는 단면에서 검사하지 않고 `validation.notes`에 "lateral release: not measurable midsagittally"로 기록. 현재 후보(blade–alveolar 0.959mm)는 이 규칙으로 PASS.

[/k/ CONTACT]
- exact rule: **CONTACT SET S_k = DORSUM 76–103.** 접촉 대상 = VELUM_LINE ∪ PALATE_LINE 뒤쪽 16점. `min dist ≤ 1.0mm + τ` 이고 최근접점이 그 구간 안이면 PASS.
- penetration/contact interpretation: 접촉 음소(ㄱ·ㄹ·ㄴ·ㄷ·ㅌ·ㅋ 등)에서 **접촉 집합의 겹침 ≤1.5mm + τ는 접촉**이지 관통이 아니다(렌더 시 선에 스냅). 관통 FAIL은 ① 겹침 >1.5mm + τ 또는 ② 접촉 집합 **밖**의 정점이 구개를 넘을 때(예: /k/ 중 혀끝이 경구개 관통)만. 현재 후보(velar 0.011mm)는 PASS 대상.

[UNCHANGED RULES]
- topology 128/24/16, 시계 방향, v0=TIP, v64=ROOT_TURN, 호별 재표본, 영역 9개(v1.3 C1) — 유지.
- self-intersection 0 — 유지.
- same speaker(pk_2015)·same scale·same origin(ALVEOLAR_POINT) — 유지.
- τ·nearest-rank 규약(v1.5 E2) — 유지.
- PHASE 1.5 REST 규칙·표시 구조·카메라 프레임(v1.7 + ERRATUM 1·2 + RELEASE) — 동결 그대로. PHASE 1.5 재오픈 없음.
- 관측 기하 변형 금지, 새 트레이스 금지, 정직성 라벨 — 유지.

[PHASE 2 RELEASE CRITERION]
viseme 1개 = VALIDATED 조건: S 전부 PASS + A(그 음소의 정의 관계, 아래 표) PASS + D 배경 노출 0 + P 상한 안 + HUMAN(조음 위치·방식 ✓, 혀로 보임·고무/뾰족 없음 ✓, 소스 MRI 오버레이와 같은 관계 ✓).
A 정의 관계 표(FIRST SET):
- REST: PHASE 1.5 규칙(동결).
- KO_k_velar: S_k 접촉 PASS + 혀끝(S_l)–치조 거리 ≥3mm(앞은 떨어져 있음).
- KO_a_open: DORSUM(76–103)→PALATE_LINE 최소 거리 ≥8mm + 턱 각도 ≥ REST+6° + 입술 간격 ≥8mm.
- KO_l_lateral: S_l 접촉 PASS + DORSUM→PALATE 최소 거리 ≥4mm(혀 몸은 낮음).
- KO_p_bilabial: 입술 간격 ≤0.1mm(스냅) — 혀 조건 없음.
- KO_i_front: DORSUM_front(90–103)→PALATE_LINE 앞쪽 절반 거리 1.5~5mm(접근, 접촉 아님) + 입술 간격 ≤6mm + 턱 ≤ REST+4°.
FIRST SET 6개 전부 VALIDATED = PHASE 2 PASS.

[AMENDMENT REQUIRED]
**YES** — MASTER_SPEC_AMENDMENT_v1.8.md(§10 PHASE 2 QA 3층 분리·음소별 A표·접촉 집합·관통 해석, §3 BODY_TISSUE 위 경계 동적화·BASE_FILL 폐기, §18 PHASE 2 PASS 조건).

[NEXT EXACT ACTION]
Codex가 **재트레이스·기하 변형 없이** 이미 확보한 각 음소의 best natural candidate(/k/ velar 0.011, /a/, /l/ blade 0.959, /i/, /p/)에 v1.8 QA(S·A·P)를 재적용해 표로 보고 → BODY_TISSUE 위 경계 동적화 구현(렌더 전용) → 5 viseme 승인 시트(소스 MRI 프레임 + 오버레이 | 퍼펫 렌더) 생성 → D 검사 → **STOP(사람 게이트 대기).** 비용 0.
