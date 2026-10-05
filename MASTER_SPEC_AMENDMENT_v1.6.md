# MASTER_IMPLEMENTATION_SPEC — AMENDMENT v1.6 (2026-10-06)
기준: v1.0 @ 74b5a011(불변) + v1.1~v1.5. **v1.6은 v1.5의 E1(클리핑·깊이)·E3(FLOOR 통계)를 덮어쓴다.** E2 τ·nearest-rank 규약은 유지. 근거: review/SILHOUETTE_FLOORSTAT_DECISION_2026-10-06.md.

## F1. §3 — 원칙 명문화
- **표시 전용(style/display-only) 기하(HEAD_SILHOUETTE, 치아 스타일 레이어 등)는 QA·기하 생성·충돌·클리핑 원천으로 쓰지 않는다.** 그 기하의 자기교차·결함은 PHASE 8에서만 다루고 ASSET_MANIFEST에 "known issue"로 기록한다.

## F2. §3 — TONGUE_BASE_FILL 깊이 규칙 (v1.5 E1 대체)
- 체인 유지: U1…U63 → P → P′ → A′ → A → U1. **클리핑 없음.**
- 깊이 D(VIRTUAL): A, P에서 −y 방향 반직선을 HEAD_SILHOUETTE **폴리라인(선분 집합)**과 교차시켜 첫 교점 거리 L_A, L_P 측정. D = min(25mm, L_A − 1mm, L_P − 1mm); 교점 없으면 해당 항 25mm; 하한 3mm. anatomy_tract.json에 `{D, L_A, L_P, grade:"VIRTUAL"}` 기록.

## F3. §4·§10 — FLOOR QA 통계 (v1.5 E3 대체)
- 검사 정점: 11–53 중 x가 FLOOR_LINE x 범위의 안쪽 80%(양 끝 10% 제외)에 투영되는 정점. n < 8이면 트림 해제·명시.
- 게이트: (중앙값 ≤ T_med + τ) AND (인라이어 비율 f = |{d ≤ T_tail + τ}|/n ≥ 0.85). REST_PROXY: T_med 1mm·T_tail 3mm. 그 외: T_med 3mm·T_tail 6mm.
- p90(nearest-rank)·최댓값은 보고 전용(게이트 아님).

## F4. §10 — 윤곽 vs 선 부착 QA 일반 규칙(신설, 100단어 공통)
- 모든 "윤곽 정점 → 기준선" 부착 검사는 F3 형식(안쪽 80% 범위, 중앙값 + 인라이어 비율 0.85, τ 적용, p90·max 보고 전용)으로 정의한다. "윤곽 → REST 같은 인덱스" 안정성 검사(ROOT STABILITY)는 중앙값·최댓값 규칙 유지.

## F5. PHASE 1.5 절차
- 레이캐스트 깊이 → BASE_FILL 완성 → FLOOR QA(F3) 재계산 보고 → 승인 시트 2장 → STOP.
