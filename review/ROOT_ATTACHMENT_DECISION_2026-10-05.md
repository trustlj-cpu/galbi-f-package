# ROOT ATTACHMENT QA — LEAD ARCHITECT DECISION (Claude Code · 2026-10-05)

[ROOT CAUSE]
v1.3의 "root_under 54–64는 FLOOR_LINE 뒤쪽 15mm 연장선에 붙는다"는 **해부학적 가정이 틀렸다.** 정중앙 단면에서 FLOOR_LINE(구강저)은 **앞쪽 설하 영역**만 덮는다. 혀의 뒤아래 경계(54–64)는 구강저의 연장이 아니라 **설골·후두개(vallecula) 쪽으로 내려가는 조직 절단선**이다 — 혀 근육이 이설골근·설골로 이어지는 자리라 MRI에서는 "경계"가 아니라 "트레이서가 그은 선"이다. 그 선이 구강저 연장선에서 20~30mm 떨어진 것은 **관측이 맞고 규칙이 틀렸다는 증거**다. Codex가 윤곽을 threshold에 맞춰 변형하지 않은 것이 맞다. 기하 수정 대상 없음: 잘못된 OBSERVED 경계는 없다.

[ROOT ATTACHMENT MODEL]
- anatomical interpretation: 혀 뿌리의 "부착"은 특정 선에 닿는 것이 아니라 **(a) 뿌리 영역이 발화 중 거의 움직이지 않는다(설골에 묶여 있다)**는 것과 **(b) 혀 아래·뒤에 빈 공간이 보이면 안 된다(그 자리는 근육·설골·후두개가 채운다)**는 두 가지 제작 조건으로 번역된다. 따라서 QA는 "어디에 닿는가"가 아니라 **"REST 대비 얼마나 움직였는가"(안정성)** 와 **"가려지는가"(채움)** 로 정의한다.
- reference geometry: ① **REST_PROXY의 LOWER 호(1–63)와 ROOT_TURN(64)** — OBSERVED, 모든 viseme의 기준. ② **TONGUE_BASE_FILL**(DERIVED, 정적 메시): REST의 1–63을 아래로 2mm 평행 이동한 곡선을 위 경계로, FLOOR_LINE 앞끝·JAW 내면·PHARYNX_WALL 하단을 잇는 폴리곤을 아래 경계로 하는 채움 영역. 혀 아래·뒤의 공간을 영구히 가린다. 혀 색보다 약간 어두운 같은 계열 색으로 렌더(스타일 단계), QA 단계에선 플랫.
- tested vertices: **54–64**(root_under + ROOT_TURN), 그리고 안정성 보조로 **65–75**(root_upper).
- distance definition: 각 정점의 **REST 같은 인덱스 정점까지의 유클리드 거리**(같은 좌표계·같은 원점, 턱 회전 보정 없음 — 뿌리는 턱과 함께 움직이지 않는다).
- threshold: 54–64: 중앙값 ≤3mm, 최댓값 ≤6mm(모든 viseme; /a/처럼 인두가 좁아지는 모음은 65–75에서 뒤로 밀리지만 54–64는 유지). 65–75: 중앙값 ≤5mm, 최댓값 ≤8mm. 기준은 **고정 mm이되 참조(REST_PROXY)에서 측정된 상대값**이다 — 즉 reference-derived. 절대 위치 threshold(어느 선에 ≤3mm)는 폐기. REST_PROXY 자신은 정의상 0.
- 추가(채움 검사, 자동): 각 viseme에서 LOWER 호 11–63 중 TONGUE_BASE_FILL 위 경계보다 **2mm 넘게 위로** 올라간 정점 수 = 0 (혀 아래로 빈틈이 노출되지 않음).

[ROOT_TURN ROLE]
**순수 기하 앵커.** LOWER/UPPER 호를 나누고 대응을 고정하는 점이지 부착점이 아니다. 어디에 닿아야 한다는 조건 없음. 안정성 검사(54–64)에 포함되는 것은 "뿌리가 거의 안 움직인다"는 성질 때문이지 부착 때문이 아니다.

[FLOOR QA]
- keep/change: **CHANGE** — 범위 제한 + 백분위 추가.
- exact criterion: 검사 정점 = 11–53 중 **x가 FLOOR_LINE의 x 범위 안에 투영되는 정점만**(FLOOR_LINE이 덮는 앞쪽 설하 구간; 뒤쪽 정점은 BASE_FILL 채움 검사가 담당). 거리 = FLOOR_LINE까지 최단 거리. REST_PROXY: 중앙값 ≤1mm, **90백분위 ≤3mm**. 그 외 viseme: 중앙값 ≤3mm, 90백분위 ≤6mm. 최댓값 기준은 두지 않는다(끝점 1~2개의 트레이스 잡음에 좌우되므로). 현재 REST 결과(중앙값 0.04mm)는 범위 제한 후 재계산해 보고.
- FLOOR_LINE_EXT(15mm 연장선) 삭제.

[v1.3 RULES SUPERSEDED]
- C3 "ROOT ATTACHMENT: 정점 54–64 → FLOOR_LINE_EXT, 중앙값 ≤3/최대 ≤5" → 폐기 → 위 REST 상대 안정성 + BASE_FILL 채움 검사로 교체.
- C2 "FLOOR_LINE_EXT 추가" → 삭제. 대신 TONGUE_BASE_FILL(DERIVED 정적 메시) 추가.
- C3 "FLOOR ATTACHMENT 정점 11–53 중앙값 ≤1mm" → 위 범위 제한·백분위 기준으로 교체.
- C3 PHARYNX CLEARANCE(65–75 ≥1mm), 접촉·관통·기도 검사 윗면 한정 → 유지.
- v1.3 C1 인덱싱·재표본 규칙 → 유지(검증 통과).

[AMENDMENT v1.4 REQUIRED]
**YES** — MASTER_SPEC_AMENDMENT_v1.4.md(v1.3 C2·C3 해당 항목 덮어쓰기, §3에 TONGUE_BASE_FILL 추가).

[NEXT EXACT ACTION]
Codex가 **윤곽 수정 없이** ① TONGUE_BASE_FILL 폴리곤을 REST LOWER 호(−2mm 오프셋)와 FLOOR_LINE·JAW 내면·PHARYNX_WALL 하단으로 생성(anatomy_tract.json에 DERIVED로 추가) ② 새 FLOOR QA(범위 제한·중앙값·90백분위)를 REST에 재계산해 보고 ③ ROOT 안정성 검사는 REST 자신이라 0mm — 규칙만 qa_rules.json에 등록 ④ 그대로 PHASE 1.5 승인 시트 2장(성도 오버레이 + REST 퍼펫 렌더, BASE_FILL 포함)을 만들어 **STOP(사람 승인 대기).** 비용 0.
