# MASTER_IMPLEMENTATION_SPEC — AMENDMENT v1.2 (2026-10-05)
기준: v1.0 @ 74b5a011(불변) + v1.1(별도 파일). **v1.2는 v1.1의 §A2·A3·A6을 아래로 덮어쓴다.** 근거: review/PHASE1_5_DECISION_2026-10-05.md.

## B1. §2 좌표계 변경
- 원점 = **ALVEOLAR_POINT**(경구개 하단 곡선 앞쪽 끝의 최대 곡률점, OBSERVED). +x 앞(입술), +y 위. 단위 mm.
- 랜드마크 3등급 라벨 필수: `OBSERVED`(MRI에서 트레이스) / `DERIVED`(관측값에서 규칙 계산) / `VIRTUAL`(규약값). anatomy_tract.json의 모든 점·선에 `"grade"` 필드.
- 배율: 1순위 데이터셋 문서의 mm/px(OBSERVED), 2순위 경구개 길이 45mm 정규화(VIRTUAL). `"scale":{"mm_per_px":..,"source":"dataset_doc|palate_45mm"}`.

## B2. §3 퍼펫 정적 해부 (v1.1 A2 대체)
- OBSERVED 오브젝트: PALATE_LINE(64), VELUM_LINE(16, V_open 셰이프키), PHARYNX_WALL(32), FLOOR_LINE(32, JAW_PIVOT 자식), UPPER_LIP/LOWER_LIP(24/24), HEAD_SILHOUETTE(≈64, 스타일 전용), TONGUE(128).
- DERIVED: JAW_PIVOT(턱 최대 열림/닫힘 프레임 사이 강체 회전 적합 중심, 잔차 ≤2mm; 실패 시 VIRTUAL 규약값), ROOT_ANCHOR(REST_PROXY 혀 뿌리 최저점).
- VIRTUAL: INCISOR_EDGE(ALVEOLAR_POINT 기준 앞 2mm·아래 8mm), UPPER/LOWER_TEETH(스타일 레이어 전용, QA 미사용).
- HYOID_ANCHOR 삭제. 턱 부호 +Z 열림 유지.

## B3. §4 viseme 구성 (v1.1 A3 대체)
- 참조: 같은 화자(USC pk_2015) 해당 음소 구간의 **중앙 프레임**, 같은 배율·원점. 구간 판정과 프레임 id는 보고에 기록.
- 혀 128점(혀끝 0, 호 길이 등간격). 부착 조건: 아랫면–FLOOR_LINE ≤1mm(모음·휴지), 뿌리 띠 거리 ≤3mm(전 viseme). 자유 덩어리 금지.
- 접촉 검사 상대: PALATE_LINE(치조 포함)·VELUM_LINE·입술 윤곽만. 치아는 QA에 쓰지 않는다.
- 체크리스트: v1.0 1~9 유지(4번의 "치조"는 ALVEOLAR_POINT 기준) + 10) 뿌리 띠 ≤3mm 11) 아랫면 ≤1mm(해당 viseme) 12) 승인 시트 2열(MRI+오버레이 | 퍼펫). 기도 면적 ±30% 규칙은 삭제(기록만).

## B4. §18 PHASE 1.5 (v1.1 A6 대체) — REFERENCE TRACT MASTER
PURPOSE 참조 화자의 **REST_PROXY(평균 자세) 프레임**에서 성도를 트레이스하고 유도 랜드마크를 계산해 anatomy 교체 + REST 재건 / INPUT pk_2015 로컬 영상 5개(598프레임), 기존 혀 윤곽 추출 파이프라인 / TASKS ① 전 프레임 혀 윤곽 → 정렬·128점 재표본 → 평균 윤곽 → RMS 최소 + 입술 비폐쇄 + 접촉 없음 조건의 첫 프레임 = REST_PROXY(상위 5 후보 기록) ② 배율 출처 확정 ③ OBSERVED 7종 트레이스 ④ DERIVED: JAW_PIVOT 강체 적합(최대 열림·닫힘 프레임 id, 잔차), ROOT_ANCHOR ⑤ VIRTUAL 점 생성·라벨 ⑥ anatomy_tract.json, puppet_master.blend 갱신(`-b`), REST.json ⑦ 승인 시트 2장 / CREATE anatomy_tract.json, REST.json/png, 시트 2장, PHASE_LOG / REUSE PHASE 1 구조, 윤곽 추출 파이프라인 / NOT TO TOUCH 15_GP, 기존 6 draft(ARCHIVE 표시) / AUTO QA 점 수·자기교차 0·구개 관통 0·뿌리 띠 ≤3mm·아랫면 ≤1mm·배율 출처·등급 라벨 전부 존재·JAW_PIVOT 잔차 / HUMAN GATE 시트 2장 ✓ / PASS AUTO + HUMAN / FAIL REST_PROXY 후보 2순위로 1회 재시도 → 그래도 실패면 LEAD에 보고(SPEC 재개정) / STOP 성도+REST 승인 전 viseme 착수 금지 / COMPLEXITY 중 / MODEL 이미지 입력·medium.
PHASE 2는 1.5 PASS 후 B3 규칙으로 재시작.
