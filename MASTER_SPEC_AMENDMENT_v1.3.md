# MASTER_IMPLEMENTATION_SPEC — AMENDMENT v1.3 (2026-10-05)
기준: v1.0 @ 74b5a011(불변) + v1.1 + v1.2. **v1.3은 v1.2의 B1 인덱스 분절, B2 ROOT_ANCHOR, B3 부착 조건, B4 QA 문구를 덮어쓴다.** 근거: review/TONGUE_INDEXING_DECISION_2026-10-05.md.

## C1. §2 — 128점 닫힌 혀 윤곽의 canonical indexing (B1 인덱스 분절 대체)
- 방향: 시계 방향(+x 앞, +y 위). TIP에서 출발해 아랫면 → ROOT_TURN → 윗면 → TIP.
- 앵커 2개: **vertex 0 = TIP**(x 최대, 동률 시 y 큰 점), **vertex 64 = ROOT_TURN**(x 최소; 그 점의 y가 centroid y보다 높으면 centroid y 아래 점 중 x 최소; 동률 시 y 작은 점). 각 viseme에서 독립적으로 찾는다.
- 호: LOWER 1–63(아랫면), UPPER 65–127(윗면). 닫는 변 없음(127→0은 윗면 마지막 변).
- 의미 영역: tip_under 1–10 · floor 11–53 · root_under 54–63 · ROOT_TURN 64 · root_upper 65–75 · dorsum_back 76–89 · dorsum_front 90–103 · blade 104–117 · tip_upper 118–127. 합집합: TIP={0}∪118–127∪1–10, BLADE=104–117, DORSUM=76–103, ROOT=54–75, FLOOR=11–53.
- 재표본 규칙(모든 viseme 공통): 원 윤곽에서 앵커 2개 → 두 호 분리 → **호마다** 호 길이 등간격 64구간 재표본 → 시계 방향 강제 → 검사(자기교차 0, v0=x 최대, v64=규칙 ROOT_TURN과 ≤0.5mm, 변 127→0 길이가 윗면 평균 변 길이 ±50%).
- viseme JSON `segments` 필드를 위 9영역 + 합집합으로 교체.

## C2. §3 — ROOT_ANCHOR 삭제 (B2 대체)
- 별도 앵커 점 없음. ROOT_TURN(v64)이 뿌리 앵커. HYOID_ANCHOR·ROOT_ANCHOR 오브젝트는 만들지 않는다.
- FLOOR_LINE에 DERIVED 연장선 추가: 마지막 변 방향으로 뒤·아래 15mm 직선(`FLOOR_LINE_EXT`, QA 전용).

## C3. §4·§10 — 부착·간격 QA (B3 부착 조건·B4 QA 문구 대체)
- ROOT ATTACHMENT: 정점 54–64(11점) → FLOOR_LINE_EXT까지 최단 거리, 중앙값 ≤3mm·최댓값 ≤5mm(REST_PROXY·모음), 중앙값 ≤4mm(자음).
- FLOOR ATTACHMENT: 정점 11–53 → FLOOR_LINE, 중앙값 ≤1mm(REST_PROXY), ≤3mm(그 외).
- PHARYNX CLEARANCE: 정점 65–75 → PHARYNX_WALL 최소 거리 ≥1mm.
- 접촉·구개 관통·기도 면적 검사는 **윗면 65–127(+0)만** 사용. 접촉 viseme의 혀끝 검사 정점 = {0} ∪ 118–127 ∪ 1–10 중 PALATE_LINE 최근접점.
- 체크리스트 10·11을 위 세 QA로 교체. 그 외 v1.2 항목 유지.

## C4. PHASE 1.5 절차 보강
- OBSERVED REST 트레이스 직후 C1 재표본을 적용하고 C3 QA 수치를 보고한 뒤(STOP) 승인 시트로 진행. 기존 윤곽 좌표의 기하 수정 없이 **인덱싱만** 재적용한다.
