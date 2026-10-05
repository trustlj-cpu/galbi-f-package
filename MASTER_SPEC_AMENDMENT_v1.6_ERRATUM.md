# MASTER_SPEC_AMENDMENT v1.6 — ERRATUM (2026-10-06)
v1.6 F2의 내부 모순 정정. 새 설계·추가 QA 없음. 이 파일이 v1.6 F2를 덮어쓴다.

[V1.6 ERRATUM]
F1(표시 전용 기하는 QA·기하 원천 금지)과 F2(실루엣 레이캐스트로 D 결정)가 모순이었다. F1이 우선이다. F2의 레이캐스트를 삭제한다.

[HEAD_SILHOUETTE]
- display-only: **YES — 완전한 표시 전용.** QA·기하 생성·충돌·클리핑·레이캐스트 어디에도 쓰지 않는다. 자기교차 5개는 ASSET_MANIFEST에 "known issue, display-only"로 기록하고 PHASE 8에서만 다룬다.
- raycast allowed: **NO.** L_A·L_P 계산 삭제.
- D rule: **VIRTUAL 상수 25mm.** 클리핑 없음. anatomy_tract.json에 `{"D_mm":25,"grade":"VIRTUAL"}`. BASE_FILL은 측정용 해부가 아니라 혀 아래·뒤 빈 공간을 가리는 정적 채움이므로 이것으로 충분하다. (피부선 밖으로 나오는지는 승인 시트에서 사람이 보고, 나오면 D를 20mm로 1회 줄인다 — 규칙 변경 아님.)

[BASE_FILL]
- unique vertex count: **67** = U1…U63(63) + P, P′, A′, A(4).
- closing edge: **A → U1**은 닫는 변이지 새 정점이 아니다. "63+5점" 문구는 "67 고유 정점, 67개 변의 닫힌 다각형"으로 정정.

[NEXT ACTION]
- 기존 v1.6의 나머지 FLOOR QA 규칙(F3 안쪽 80% 범위·중앙값·인라이어 비율 ≥0.85, F4 일반 규칙, E2 τ·nearest-rank)은 **그대로 유지.**
- Codex가 바로 BASE_FILL(67점, D=25) + FLOOR QA(F3) 재계산 + Blender `-b` 승인 시트 2장(성도 오버레이, REST 퍼펫 렌더 — BASE_FILL 포함)까지 진행해도 되는가: **YES.** 그 뒤 STOP(사람 승인 대기). 윤곽 수정 없음, 비용 0.
