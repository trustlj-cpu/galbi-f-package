# 128-POINT CLOSED TONGUE CONTOUR — CANONICAL INDEXING DECISION (Claude Code · 2026-10-05)

[ROOT CAUSE]
v1.0 §2·v1.2 B1의 "tip 0–15 / blade 16–31 / dorsum 32–79 / root 80–127"은 **열린 곡선(혀 윗면 한 줄)**에만 성립하는 정의다. 혀 윤곽은 **닫힌 곡선**이라 윗면(구개 쪽)과 아랫면(구강저 쪽)이라는 **두 호**가 있고, tip·blade·dorsum·root는 윗면 위의 영역이다. 닫힌 128점을 연속 4구간으로 자르면 어느 한 구간은 반드시 아랫면을 "root"나 "dorsum"으로 오인한다 — 감사에서 80–127(윗면)이 root 띠에서 40mm 떨어진 것은 그 오류의 정확한 증상이다. Codex가 49–80을 임의 채택하지 않고 멈춘 것은 맞다. 정의를 **두 앵커 + 두 호**로 바꾼다.

[CANONICAL CLOSED-CONTOUR INDEXING]
- exact orientation: **시계 방향(clockwise)**, 좌표계 +x 앞(입술)·+y 위 기준. 혀끝에서 출발해 **아랫면(구강저 쪽)을 먼저** 지나 뒤쪽 꺾임점으로, 거기서 **윗면(구개 쪽)**을 따라 혀끝으로 돌아온다. (현재 OBSERVED REST의 순서와 같다.)
- vertex 0: **TIP** — 혀끝. 기하 규칙: x 최대 점, 동률이면 y 큰 점.
- vertex 64: **ROOT_TURN** — 아랫면과 윗면이 만나는 뒤쪽 꺾임점. 기하 규칙: x 최소(가장 뒤) 점, 단 그 점의 y가 윤곽 중심(centroid) y보다 높으면(혀 뒤가 부풀어 오른 /k/ 등) "centroid y 아래 점들 중 x 최소"로 재선택. 동률이면 y 작은 점.
- exact index ranges / semantic regions (영역은 호 단위로 정의, **연속 4구간이 아니다**):
  - LOWER arc = 1–63 (TIP→ROOT_TURN, 아랫면): `tip_under` 1–10 · `floor` 11–53 · `root_under` 54–63
  - ROOT_TURN = 64
  - UPPER arc = 65–127 (ROOT_TURN→TIP, 윗면): `root_upper` 65–75 · `dorsum_back` 76–89 · `dorsum_front` 90–103 · `blade` 104–117 · `tip_upper` 118–127
  - 의미 영역(합집합): TIP = {0} ∪ 118–127 ∪ 1–10 · BLADE = 104–117 · DORSUM = 76–103 · ROOT = 54–63 ∪ {64} ∪ 65–75 · FLOOR(아랫면) = 11–53
- upper-surface range: **65–127 (+0)** — 접촉·구개 관통·기도 면적 검사는 이 범위만.
- lower-surface range: **1–63** — 구강저 부착 검사는 이 범위만.
- posterior/root range: **54–75** (꺾임점 64 포함).
- closure segment: 별도 "닫는 변"은 없다. 변 127→0은 윗면의 마지막 변일 뿐이고, 윤곽은 128개 변의 닫힌 다각형이다. vertex 127과 0의 거리는 윗면 변 하나의 평균 길이(윗면 호 길이/63)와 ±50% 안이어야 한다(현재 1.88mm는 정상 범위로 예상 — 재표본 후 확인).

[ROOT_ANCHOR RULE]
HYOID/ROOT_ANCHOR라는 **별도 점을 두지 않는다.** ROOT_TURN(vertex 64)이 앵커다. 위 기하 규칙으로 **각 viseme에서 독립적으로** 찾는다(REST에서 한 번 정하고 고정하지 않는다 — /k/·/a/에서 꺾임점 위치가 다르다).

[ROOT ATTACHMENT QA]
- exact tested vertices or geometric segment: **`root_under` 54–63 + ROOT_TURN 64** (총 11점). (65–75는 인두 쪽 윗면이라 부착이 아니라 **간격** 검사 대상.)
- distance definition: 각 정점에서 **FLOOR_LINE 뒤쪽 연장선**까지의 최단 거리. FLOOR_LINE(32점)의 마지막 변 방향으로 뒤·아래로 15mm 직선 연장한 폴리라인을 기준선으로 쓴다(구강저가 설골 쪽으로 이어지는 선의 근사, DERIVED 라벨).
- threshold: 11점의 **중앙값 ≤3mm, 최댓값 ≤5mm**(REST_PROXY와 모음 viseme). 자음 viseme(/k/ /l/ /p/)는 중앙값 ≤4mm.
- 추가 QA(같은 패키지): FLOOR 부착 — `floor` 11–53의 FLOOR_LINE까지 거리 중앙값 ≤1mm(REST_PROXY), ≤3mm(그 외 viseme). PHARYNX 간격 — `root_upper` 65–75에서 PHARYNX_WALL까지 최소 거리 ≥1mm(관통 금지, /a/ 포함).

[VISEME CORRESPONDENCE RULE]
- resampling anchors: **TIP(0)과 ROOT_TURN(64) 두 개뿐.** "floor turn" 같은 세 번째 앵커는 MRI에서 안정적으로 식별되지 않으므로 두지 않는다.
- how correspondence is preserved: 각 viseme의 원 윤곽(트레이스)에서 ① TIP·ROOT_TURN을 위 기하 규칙으로 찾고 ② 윤곽을 두 호로 자른 뒤 ③ **호마다 따로** 호 길이 등간격으로 재표본 — LOWER 호 64구간(정점 0..64), UPPER 호 64구간(정점 64..127,0) ④ 방향을 시계 방향으로 강제(반시계면 뒤집기) ⑤ 결과: 모든 viseme에서 같은 인덱스가 같은 호의 같은 호 길이 비율을 가리킨다(혀끝↔혀끝, 꺾임점↔꺾임점, 윗면↔윗면). 셰이프키 혼합은 이 대응 위에서만 수행한다. 재표본 후 자동 검사: 자기교차 0, 시계 방향, 정점 0 = x 최대, 정점 64 = 규칙상 ROOT_TURN과 ≤0.5mm.

[v1.2 RULES SUPERSEDED]
- B1 "tip 0–15 / blade 16–31 / dorsum 32–79 / root 80–127(호 길이 비율로 REST에서 고정)" → **폐기.** 위 두 호 정의로 교체.
- B2 "ROOT_ANCHOR(REST_PROXY 혀 뿌리 최저점)" → **폐기.** ROOT_TURN 규칙으로 교체.
- B3 "뿌리 띠 거리 ≤3mm(전 viseme)" → 위 ROOT ATTACHMENT QA(정점 54–64, FLOOR_LINE 연장선, 중앙값 3/최대 5)로 교체. "아랫면–FLOOR_LINE ≤1mm"는 정점 11–53·REST_PROXY 한정으로 명시.
- B4 AUTO QA 항목 문구를 위 정의로 갱신. 그 외 v1.2(원점 ALVEOLAR_POINT, 3등급 라벨, REST_PROXY, JAW_PIVOT 적합, 치아 QA 미사용)는 유지.

[AMENDMENT v1.3 REQUIRED]
**YES** — 별도 파일 MASTER_SPEC_AMENDMENT_v1.3.md(v1.2 B1·B2·B3의 해당 항목 덮어쓰기, B4 QA 문구 갱신).

[NEXT EXACT ACTION]
Codex가 **기하 수정 없이** 현재 OBSERVED REST 윤곽에 v1.3 재표본 규칙을 적용한다: TIP·ROOT_TURN 찾기 → 두 호 분리 → 호별 64구간 재표본 → 방향 확인 → 위 QA(ROOT ATTACHMENT 54–64, FLOOR 11–53, PHARYNX 65–75, 자기교차, 127↔0 변 길이) 수치 보고 → **STOP.** 수치가 PASS면 PHASE 1.5의 나머지(승인 시트 2장)로 이어가고, FAIL이면 수치와 함께 LEAD에 보고(규칙 재검토). 비용 0, Blender 불필요(numpy만).
