# PHASE 1.5 FAIL — LEAD ARCHITECT DECISION (Claude Code · 2026-10-05)

[ROOT CAUSE]
AMENDMENT v1.1(§A2·A6)이 **정중앙 rtMRI에 원리적으로 보이지 않거나 존재하지 않는 것**을 필수 랜드마크로 요구했다. 내 명세 오류다.
- 치아(앞니 절단연): MRI에서 치아 에나멜·상아질은 신호가 없어 **보이지 않는다.** 원점을 "윗앞니 끝"으로 둔 §2 정의가 애초에 MRI 트레이스와 양립하지 않았다.
- 하악 과두(JAW_PIVOT): 턱관절은 정중앙면에서 **양옆으로 50mm 이상** 떨어져 있어 정중앙 단면에 **존재하지 않는다.** 어떤 화자·어떤 데이터로 바꿔도 안 보인다.
- 설골(HYOID_ANCHOR): 작고 정중앙에서 흐릿하며 발화 중 움직인다. 식별 가능 프레임이 드문 게 정상이다.
- "휴지/중립 프레임": 발화 코퍼스에는 정의된 휴지 상태가 없다. 영상 시작 프레임(/k/ F0)은 선행 준비 동작이 섞일 수 있다는 Codex 판단이 맞다.
따라서 Codex의 FAIL 처리(임의 좌표 생성 거부·Blender 미실행)는 **정확한 행동**이었고, 소스(pk_2015)나 아키텍처의 문제가 아니다.

[SPEC DECISION]
- 아키텍처: **유지.**
- REFERENCE TRACT 소스: **pk_2015 유지**(선택지 2·3 기각 — 다른 화자·다른 코퍼스도 같은 물리적 한계를 가진다. 소스를 바꾸면 "같은 화자 전 음소"라는 가장 큰 장점을 잃는다).
- 필수 랜드마크 정의: **완화가 아니라 교체.** "보이는 것은 관측(OBSERVED), 안 보이는 것은 관측값에서 규칙으로 유도(DERIVED), 유도도 안 되는 것은 규약(VIRTUAL)"으로 3등급화하고 라벨을 데이터에 기록한다. 선택지 **1 + 4**.
- 휴지 프레임 요구: **"평균 자세 프레임(REST_PROXY)"으로 교체**(선택지 4). 598프레임 전체의 혀 윤곽 평균에 가장 가까운 프레임이 통계적 중립이다 — 임의가 아니라 계산으로 정해진다.

[AMENDMENT v1.2 REQUIRED: YES]
(별도 파일 MASTER_SPEC_AMENDMENT_v1.2.md. v1.0 고정 커밋·v1.1 파일은 그대로 두고 v1.2가 v1.1의 §A2·A3·A6을 덮어쓴다.)

[EXACT REVISED PHASE 1.5 REQUIREMENTS]
1. **좌표계 원점 변경**: 원점 = **ALVEOLAR_POINT**(경구개 하단 곡선의 앞쪽 끝에서 곡률이 최대인 점 — MRI에서 연조직으로 관측 가능). +x 앞, +y 위. 앞니 끝은 원점이 아니라 VIRTUAL 점.
2. **배율**: 1순위 — 데이터셋 문서의 픽셀 간격(USC rtMRI는 FOV·행렬 크기가 공개돼 mm/px가 정해짐; Codex가 pk_2015 문서/헤더에서 확인해 기록, OBSERVED). 2순위(문서를 못 찾을 때) — 경구개 길이(ALVEOLAR_POINT→경구개 뒤끝) = 45mm로 정규화, VIRTUAL 라벨.
3. **REST_PROXY 프레임 선정(계산)**: 598프레임 각각의 혀 윤곽(기존 추출 파이프라인)을 같은 배율·원점으로 정렬 → 128점 재표본 → 전 프레임 평균 윤곽 계산 → 평균과의 RMS 거리가 최소인 프레임 중 **입술 열림(비폐쇄)·혀-구개 접촉 없음** 조건을 만족하는 첫 프레임 = REST_PROXY. 보고에 프레임 id·RMS·후보 상위 5개 기록. /k/ F0이 이 기준으로 뽑히면 그대로 쓴다.
4. **OBSERVED 랜드마크(REST_PROXY 프레임에서 트레이스, 필수)**: PALATE_LINE(경구개 하단 64점, ALVEOLAR_POINT 포함) · VELUM_LINE(연구개 하단 16점) · PHARYNX_WALL(인두 뒷벽 32점) · UPPER_LIP/LOWER_LIP 바깥 윤곽(각 24점) · FLOOR_LINE(구강저 — 혀 아랫면이 닿는 연조직 선, 32점) · TONGUE(128점, 혀끝 0) · 머리 바깥 실루엣(코·턱 피부, 대략 64점, 스타일용).
5. **DERIVED 랜드마크(규칙으로 계산, 필수)**:
   - **JAW_PIVOT**: 598프레임 중 턱이 가장 열린 프레임(FLOOR_LINE·턱 피부 윤곽이 가장 낮음)과 가장 닫힌 프레임을 고르고, 두 프레임의 턱 피부+구강저 윤곽 사이 **2D 강체 회전을 최소제곱 적합** → 회전 중심 = JAW_PIVOT, 회전각 = 그 두 프레임의 턱 열림 차. 적합 잔차 RMS ≤2mm면 DERIVED 채택. 초과하면 VIRTUAL 규약: 인두 뒷벽 x에서 앞으로 25mm, PALATE_LINE 뒤끝 y에서 아래로 15mm(성인 평균 근사)로 두고 라벨 VIRTUAL.
   - **ROOT_ANCHOR**(HYOID_ANCHOR 대체): REST_PROXY 혀 윤곽에서 **뿌리 쪽 최저점**(index 80–127 구간 중 y 최소)을 앵커로 삼고, 부착 검사는 "혀 뿌리 구간이 FLOOR_LINE 뒤끝–PHARYNX_WALL 앞면 사이 띠에서 ≤3mm"로 완화(이전 ≤1mm 폐기). 설골 자체는 그리지 않는다.
   - **INCISOR_EDGE(VIRTUAL)**: ALVEOLAR_POINT에서 앞 2mm·아래 8mm. 치아는 스타일 레이어에서만 그리고 QA에 쓰지 않는다. 접촉 검사의 상대는 전부 PALATE_LINE(치조 포함)·VELUM_LINE·입술 윤곽이다.
6. **viseme 트레이스 프레임**: /k/ /a/ /l/ /p/ /i/ 각각 "해당 음소 구간의 중앙 프레임"(영상의 음소 구간은 Codex가 프레임 단위로 혀 접촉·입술 폐쇄·열림 정도로 판정, 보고에 프레임 id 기록). 같은 배율·원점.
7. **AUTO QA(1.5)**: 점 수 규격 · 자기교차 0 · 혀 윗면이 PALATE_LINE 위로 >0.05mm 없음 · 혀 뿌리 띠 거리 ≤3mm · 혀 아랫면–FLOOR_LINE 거리 ≤1mm(REST_PROXY에서) · 기도 면적(PALATE_LINE↔혀 윗면) 기록(참고치, PASS 조건 아님 — 이번엔 참조가 곧 측정 대상이라 ±30% 규칙 제외) · 배율 출처 기록 · 각 랜드마크에 OBSERVED/DERIVED/VIRTUAL 라벨.
8. **HUMAN GATE(1.5)**: 승인 시트 2장 — ① REST_PROXY MRI 프레임 + 반투명 트레이스 오버레이(성도 전체) ② 퍼펫 렌더(REST). 질문: "성도가 MRI와 같은 구조로 보이는가", "혀가 구강을 채우고 바닥·뿌리에 붙어 보이는가". 둘 다 ✓면 PASS.
9. **정직성 라벨 고정**: 모든 산출물에 "참조 화자 USC pk_2015 ≠ 한국어 화자 · SHAPE_REFERENCE_ONLY · 치아·턱관절·설골은 VIRTUAL/DERIVED".

[NEXT EXACT ACTION]
Codex가 **AMENDMENT v1.2 규칙으로 PHASE 1.5를 재실행**한다: (a) 598프레임 혀 윤곽 정렬·평균 → REST_PROXY 선정 보고(프레임 id·RMS·상위 5) (b) 배율 출처 확정 (c) OBSERVED 트레이스 7종 (d) DERIVED 계산(JAW_PIVOT 적합 잔차·ROOT_ANCHOR) (e) 오버레이 승인 시트 2장 → **STOP(성도+REST 승인 대기).** viseme 5개는 승인 후. 비용 0, Blender는 (e)의 REST 렌더에만 `-b`로.
