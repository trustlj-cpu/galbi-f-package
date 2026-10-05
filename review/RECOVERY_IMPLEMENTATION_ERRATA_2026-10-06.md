# RECOVERY ARCHITECTURE — IMPLEMENTATION ERRATA (Claude Code · 2026-10-06)
RECOVERY_ARCHITECTURE_DECISION의 모호한 4개를 고정한다. 아키텍처 변경 없음.

## 1. CENTERLINE NODE ORDER / s DIRECTION → **B 확정**
- 노드 배열은 **ROOT→TIP**으로 재정렬: `N[0] = ROOT_TURN(=v64)`, `N[64] = TIP(=v0)`, `s_j = j/64`.
- 리브 짝 매핑(v1.3 인덱스 기준, j = 1…63): `L_j = v[64 − j]`(아랫면, ROOT 쪽에서 TIP 쪽으로), `U_j = v[64 + j]`(윗면; j=63일 때 v127), `N[j] = (L_j + U_j)/2`. 검산: j=1 → L=v63·U=v65(ROOT_TURN 바로 옆), j=63 → L=v1·U=v127(TIP 바로 옆).
- 출력 시 역매핑으로 v1.3 순서(v0=TIP, 1–63 아랫면, 64=ROOT_TURN, 65–127 윗면)를 그대로 복원한다. 프로파일 중심(TIP_RAISE c=1.0, BLADE 0.85, DORSUM_FRONT 0.62, DORSUM_BACK 0.35, ROOT_RETRACT 0.15)은 이 s 정의에 고정. 단위 테스트에 "s=1 노드 = v0 좌표, s=0 노드 = v64 좌표" 어설션 포함.

## 2. 평활 순서 → **중심선만 평활, 최종 윤곽 평활 없음**
순서 고정: ① 중심선 변형(모든 파라미터 변위 합산) → ② **중심선 라플라시안 평활 2회**(가중 0.5, 양 끝 N[0]·N[64] 고정) → ③ 접선 재계산(중앙차분, 끝은 단측) → ④ REST 리브 벡터를 "REST 접선→새 접선" 회전만큼 회전해 U_j·L_j 재구성(두께 보존) → ⑤ 윗면 soft clamp(PALATE/VELUM 선 아래 0.3mm, ±6노드 코사인 분산 — 이때만 해당 노드의 위 리브 길이가 줄어듦, 변화량을 QA에 기록) → ⑥ 역매핑 → 최종 128점. **최종 윤곽에 라플라시안을 다시 적용하지 않는다.** 리브 두께 비 검사(≥60%)는 ⑤ 이후 값으로 계산.

## 3. 접촉 솔버 → **`solve_to_gap(param, contact_set, target_segment, target_gap_mm, direction)`로 일반화**
- 동작: 파라미터를 `direction`(+1 올림/−1 내림)으로 이분 탐색해 `min_dist(contact_set, target_segment)`가 **target_gap_mm ± 0.1mm**가 되는 값을 찾는다(반복 ≤30, 단조성 가정; 구간 [0, cap]).
- preset 값: KO_k·KO_l `target_gap_mm = 0.5`(접촉, 렌더에서 선에 스냅), **KO_i `target_gap_mm = 2.0`**(접근, 스냅 없음). 솔버는 목표 간격에서 멈추며 **그 아래로 내려가지 않는다**(탐색 중 간격 < target − 0.1이면 상한을 낮춤). 접촉 집합은 v1.8(S_l·S_k·S_j), KO_i는 S_j = 90–117 → PALATE 앞쪽 절반.
- 솔버 실패(단조성 위반·cap 도달): 그 상태로 멈추고 `solve_status: "CAP"|"NONMONOTONIC"`을 시트에 표시(렌더는 진행).

## 4. 자동 QA는 진단, 시트는 항상 생성 → **확정**
- 원칙: **AUTO CHECK = diagnostic + visible flag / HUMAN SHEET = always produced.**
- 렌더를 막는 치명 조건은 **네 가지뿐**: NaN/inf 좌표, 정점 수 ≠ 128(입술 24·연구개 16), 다각형 면적 ≤ 0 또는 인덱스 순서 파손(v0≠TIP, v64≠ROOT_TURN), 입력 파일 누락. 이 경우에도 "렌더 불가 사유 카드"를 시트 자리에 넣어 7장 세트를 유지한다.
- 자기교차·곡률·변 길이 비·리브 두께 비·접촉/간격 거리·v1.8 A 관계·clamp 변화량은 **전부 FAIL/WARN 플래그로 시트에 표시만** 한다(빨강/노랑 띠 + 수치). 렌더·모션·가이드 생성을 막지 않는다.
- 사람 게이트가 유일한 PASS/FAIL 결정 지점이다. Codex는 플래그가 있어도 멈추지 않고 7장 시트 + 플래그 표를 함께 올린 뒤 STOP한다.

그 외 RECOVERY 문서의 아키텍처·파라미터·preset·실험·PASS 조건은 변경 없음.
