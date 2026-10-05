# MASTER_IMPLEMENTATION_SPEC — AMENDMENT v1.5 (2026-10-06)
기준: v1.0 @ 74b5a011(불변) + v1.1~v1.4. **v1.5는 v1.4의 D1(BASE_FILL 아래 경계)·D3(FLOOR QA 비교 규칙)를 덮어쓰고 §10에 공통 수치 규약을 추가한다.** 근거: review/BASEFILL_FLOORQA_DECISION_2026-10-06.md.

## E1. §3 — TONGUE_BASE_FILL 경계 체인 (v1.4 D1 대체)
- JAW 내면 경계는 **정의하지 않는다**(불필요).
- 위 경계 U(DERIVED): REST_PROXY LOWER 호 정점 1…63을 바깥(아래) 법선으로 2mm 이동(자기교차 구간은 1mm).
- 아래 체인(VIRTUAL): A = FLOOR_LINE 앞끝, P = PHARYNX_WALL 최하단, D = 25mm(HEAD_SILHOUETTE 안쪽 1mm까지로 자름).
- 닫힌 체인(시계 방향, 63+5점): **U1 → … → U63 → P → P′(P.x, P.y−D) → A′(A.x, A.y−D) → A → U1.** HEAD_SILHOUETTE로 클립. 렌더 순서 TONGUE 아래·배경 위.
- anatomy_tract.json에 `TONGUE_BASE_FILL:{upper:{grade:"DERIVED"}, lower_chain:{grade:"VIRTUAL", depth_mm:25}}`.

## E2. §10 — 공통 수치 규약 (신설)
- 허용 오차 τ = 0.5 × `scale.mm_per_px`(배율 출처가 `palate_45mm`면 τ = 1.0mm). qa_rules.json `"tolerance_mm"`에 기록, 보고서에 표시.
- 모든 OBSERVED 트레이스 기반 거리 threshold 비교는 `value ≤ threshold + τ`. 예외: 접촉 ≤0.1mm 규칙(스냅 정의, τ 미적용), 정점 수·자기교차·방향 같은 정수 규칙.
- 백분위: nearest-rank, 보간 없음 — `np.percentile(d, q, method="higher")`. n < 10이면 최댓값으로 대체하고 명시.

## E3. §4·§10 — FLOOR QA (v1.4 D3 대체)
- 검사 정점·거리 정의는 v1.4 유지(11–53 중 FLOOR_LINE x 범위 투영 정점, 최단 거리).
- 기준: REST_PROXY 중앙값 ≤1mm·p90 ≤3mm, 그 외 중앙값 ≤3mm·p90 ≤6mm — **E2 규약 적용**(τ, nearest-rank).

## E4. PHASE 1.5 절차 보강
- BASE_FILL 완성 → FLOOR QA 재계산(n·p90·τ·배율 출처 보고) → 승인 시트 2장 → STOP.
