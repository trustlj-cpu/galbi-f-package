# MASTER_IMPLEMENTATION_SPEC — AMENDMENT v1.4 (2026-10-05)
기준: v1.0 @ 74b5a011(불변) + v1.1 + v1.2 + v1.3. **v1.4는 v1.3의 C2(FLOOR_LINE_EXT)·C3(ROOT ATTACHMENT·FLOOR ATTACHMENT)를 덮어쓴다.** C1 인덱싱·재표본, C3 PHARYNX CLEARANCE·윗면 한정 검사는 유지. 근거: review/ROOT_ATTACHMENT_DECISION_2026-10-05.md.

## D1. §3 — TONGUE_BASE_FILL 추가, FLOOR_LINE_EXT 삭제
- `TONGUE_BASE_FILL`(메시, static, ROOT 자식, DERIVED): 위 경계 = REST_PROXY LOWER 호(정점 1–63)를 아래로 2mm 평행 이동한 곡선 + ROOT_TURN 주변; 아래 경계 = FLOOR_LINE 앞끝 → JAW 내면 하단 → PHARYNX_WALL 하단을 잇는 폴리라인. 혀 아래·뒤 공간을 영구히 채운다. 렌더 순서: BASE_FILL 아래, TONGUE 위. 스타일: 혀보다 한 단계 어두운 같은 계열.
- `FLOOR_LINE_EXT` 삭제. HYOID/ROOT 앵커 오브젝트 없음(v1.3 유지).

## D2. §4·§10 — ROOT QA (v1.3 C3 ROOT ATTACHMENT 대체)
- ROOT STABILITY: 정점 54–64 → REST_PROXY 같은 인덱스까지 거리(턱 보정 없음). 중앙값 ≤3mm, 최댓값 ≤6mm(모든 viseme). 정점 65–75: 중앙값 ≤5mm, 최댓값 ≤8mm.
- BASE COVERAGE: LOWER 호 11–63 중 TONGUE_BASE_FILL 위 경계보다 2mm 넘게 위에 있는 정점 수 = 0.
- ROOT_TURN(v64)은 기하 앵커이며 부착 조건 없음.

## D3. §4·§10 — FLOOR QA (v1.3 C3 FLOOR ATTACHMENT 대체)
- 검사 정점: 11–53 중 x가 FLOOR_LINE의 x 범위 안에 투영되는 정점만.
- 거리: FLOOR_LINE까지 최단 거리. REST_PROXY: 중앙값 ≤1mm·90백분위 ≤3mm. 그 외 viseme: 중앙값 ≤3mm·90백분위 ≤6mm. 최댓값 기준 없음.

## D4. 체크리스트·qa_rules.json
- 체크리스트 10 = ROOT STABILITY, 11 = BASE COVERAGE + FLOOR QA. 12(승인 시트 2열) 유지.
- qa_rules.json에 D2·D3 항목을 등록하고 REST_PROXY를 기준 프레임으로 참조.

## D5. PHASE 1.5 절차 보강
- 승인 시트 2장(성도 오버레이, REST 퍼펫 렌더)에 TONGUE_BASE_FILL을 포함해 "혀 아래에 빈틈이 없는가"를 사람이 확인한다. 통과 후 viseme 5개.
