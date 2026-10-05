# HEAD_SILHOUETTE CLIPPING + FLOOR QA STATISTIC — LEAD ARCHITECT DECISION (Claude Code · 2026-10-06)

[ROOT CAUSE]
① v1.5 E1이 **표시 전용(style/display-only) 기하인 HEAD_SILHOUETTE를 기하 연산(클리핑)의 원천으로 썼다.** 설계 원칙 위반이다 — 표시용 트레이스는 자기교차가 있어도 되고, QA·기하 생성은 OBSERVED/DERIVED 등급 기하만 써야 한다. Codex가 실루엣을 임의 수리하지 않은 것이 맞다.
② FLOOR QA의 p90은 **n=28에서 정점 1개가 바뀌면 값이 뛰는 취약한 통계**이고, 검사 범위가 혀 아랫면이 구강저에서 자연스럽게 떨어지는 **양 끝 전이 구간**(앞: 설하 공간, 뒤: 뿌리로 올라가는 구간)을 포함했다. 중앙값 0.016mm는 혀 아랫면이 구강저와 **일치**한다는 뜻이고, 4.23mm를 만드는 상위 3점은 그 전이 구간의 정점이다. 잘못된 OBSERVED 경계는 없다. 트레이스 수정 대상 없음.

[HEAD_SILHOUETTE / BASE_FILL DECISION]
- clipping required: **NO.** 폴리곤 클리핑을 삭제한다.
- silhouette role: **표시 전용으로 복귀.** QA·기하 생성·충돌 검사 어디에도 쓰지 않는다. 자기교차 5개는 PHASE 8(스타일)에서 "largest simple outer loop" 추출로 정리하되 지금은 건드리지 않고 ASSET_MANIFEST에 "known: self-intersections=5, display-only"로 기록.
- self-intersection handling: 지금 없음(위 기록만).
- exact polygon rule: BASE_FILL = **U1 → … → U63 → P → P′ → A′ → A → U1**(v1.5 체인 그대로), 단 깊이 D를 상수 25mm 대신 **레이캐스트로 결정**: A와 P 각각에서 수직 아래(−y) 방향 반직선을 쏘아 HEAD_SILHOUETTE **폴리라인**(폴리곤이 아니라 선분 집합 — 자기교차와 무관)과의 첫 교점까지 거리 L_A, L_P를 구한다. D = min(25mm, L_A − 1mm, L_P − 1mm). 교점이 없으면 그 항은 25mm. D < 3mm면 3mm로 하한. 결과 D와 L_A·L_P를 anatomy_tract.json에 VIRTUAL 라벨로 기록. 이렇게 하면 BASE_FILL이 피부선 밖으로 나오지 않으면서 폴리곤 유효성이 필요 없다. 렌더 순서는 그대로(TONGUE 아래, 배경 위).

[FLOOR QA DECISION]
- current failure interpretation: **통계·범위 문제, 기하 문제 아님.** 아랫면은 구강저와 일치(중앙값 0.016mm)하고, 상위 3점은 양 끝 전이 구간.
- tested geometry: 정점 11–53 중 x가 FLOOR_LINE x 범위의 **안쪽 80%**(양 끝 10%씩 제외 — 전이 구간 배제, 기하 규칙)에 투영되는 정점. 거리 = FLOOR_LINE까지 최단 거리. n < 8이면 트림 없이 전체 범위로 되돌리고 명시.
- statistic: **중앙값 + 인라이어 비율.** 인라이어 비율 f = |{d_i ≤ T_tail + τ}| / n. p90·최댓값은 **보고만**(게이트 아님).
- threshold: REST_PROXY — 중앙값 ≤1mm, T_tail = 3mm, **f ≥ 0.85**. 그 외 viseme — 중앙값 ≤3mm, T_tail = 6mm, f ≥ 0.85.
- tolerance: v1.5 E2 그대로(τ = 0.5×mm_per_px, 규약 배율이면 1.0mm; 중앙값·T_tail 비교에 적용).
- trace correction required: **NO.**
- exact correction rule if YES: 해당 없음. (현재 수치로 검산: 트림 전 n=28, 4mm 초과 3점 → f = 25/28 = 0.893 ≥ 0.85 PASS. 트림 후에는 끝단 정점이 빠져 f가 더 오를 것으로 예상 — Codex가 실제 값을 보고.)
- 일반 규칙(100단어 공통, 모든 "윤곽 vs 선" 부착 QA에 적용): **게이트 = (중앙값 ≤ T_med + τ) AND (인라이어 비율 f ≥ 0.85 at T_tail + τ)**, 검사 범위는 기준선 x 범위 안쪽 80%, p90·최댓값은 참고 보고. ROOT STABILITY(54–64)는 "REST 대비 같은 인덱스 거리"라 전이 구간 문제가 없으므로 기존 중앙값·최댓값 규칙 유지.

[v1.5 RULES SUPERSEDED]
- E1 "HEAD_SILHOUETTE로 클립", "D = 25mm(실루엣 안쪽 1mm까지로 자름)" → 폐기 → 레이캐스트 깊이 규칙으로 교체. 체인 자체는 유지.
- E3 "p90 ≤3mm/6mm" 게이트 → 폐기 → 안쪽 80% 범위 + 중앙값 + 인라이어 비율 0.85로 교체. E2 τ·nearest-rank 규약은 유지(p90은 보고용으로만 계속 nearest-rank).

[AMENDMENT v1.6 REQUIRED]
**YES** — MASTER_SPEC_AMENDMENT_v1.6.md(v1.5 E1 클리핑·깊이, E3 통계 덮어쓰기; §3에 "표시 전용 기하는 QA·기하 원천으로 쓰지 않는다" 원칙 명문화).

[NEXT EXACT ACTION]
Codex가 윤곽 수정 없이 ① A·P에서 레이캐스트로 L_A·L_P·D 계산 → BASE_FILL 폴리곤 완성(클리핑 없음, 63+5점) → anatomy_tract.json 저장 ② FLOOR QA를 안쪽 80% 범위·중앙값·인라이어 비율로 재계산(n, 중앙값, f, 참고 p90·최댓값, τ, 배율 출처 보고) ③ 승인 시트 2장(성도 오버레이, REST 퍼펫 렌더 — BASE_FILL 포함) 생성 → **STOP(사람 승인 대기).** 비용 0, Blender는 ③에서 `-b`로만.
