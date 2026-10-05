# MASTER_SPEC_AMENDMENT v1.7 — ERRATUM #2 (2026-10-06)
v1.7 ERRATUM의 내부 모순 2개 정정. 새 구조·새 QA 없음. 이 파일이 v1.7 ERRATUM의 해당 항목을 덮어쓴다.

[V1.7 ERRATUM #2]
① F_end→P 변의 등급·가시성이 미정의였다. ② G3 occluder 목록에 TONGUE를 넣은 것은 렌더 순서(TONGUE가 BODY_TISSUE보다 뒤)와 모순이며 false PASS를 만든다. Codex의 지적이 맞다.

[F_END_TO_P]
- grade: **DERIVED** — "soft-tissue transition edge"(구강저 뒤끝 F_end와 인두 벽 최하단 P를 잇는 직선; 두 관측점을 잇는 유도 변).
- visible: **YES**(다각형 경계로 존재하므로 화면 안에 있을 수 있음. 혀 뿌리·BASE_FILL 영역과 겹쳐 대부분 혀 또는 조직 색 안에 묻힌다).
- stroke: **NO** — 외곽선 그리지 않음. 외곽선은 FLOOR_LINE·PHARYNX_WALL 구간만(기존 규칙 유지).
- G3 treatment: DERIVED 경계로 **허용**(보이는 변 허용 목록 = OBSERVED FLOOR_LINE·PHARYNX_WALL, DERIVED W_top→W_ext, DERIVED F_end→P). 마스크 검사의 "덮여야 하는 VIRTUAL 변" 대상이 아니다.

[G3 OCCLUDER MASK]
- exact allowed occluder layers(BODY_TISSUE_DISPLAY의 VIRTUAL 변을 덮을 수 있는 레이어 = 렌더 순서상 BODY_TISSUE보다 **앞**에 있는 것만): **JAW(DISPLAY), PALATE/SKULL_PALATE, VELUM, PHARYNX(표시), UPPER_TEETH/LOWER_TEETH, UPPER_LIP/LOWER_LIP, AIRWAY(스타일), CONTACT_HIGHLIGHT, 그 밖에 렌더 순서에서 BODY_TISSUE 뒤에 명시적으로 나열된 DISPLAY 레이어.**
- TONGUE included: **NO.** TONGUE는 BODY_TISSUE보다 뒤라 아무것도 가릴 수 없다. 마스크 검사 구현은 "각 레이어의 렌더 순서 인덱스 > BODY_TISSUE 인덱스"인 레이어 마스크의 합집합만 occluder로 쓴다(이름 목록 대신 순서 인덱스로 판정해 앞으로 레이어가 추가돼도 모순이 안 생기게).

[NEXT ACTION]
- 그 외 v1.7 / v1.7 ERRATUM은 그대로 유지: **YES.**
- Codex가 바로 BODY_TISSUE 생성 → 프레임 클립 → 사전 조건 검사(①VIRTUAL 변 occluder 덮임 ②클립 후 자기교차 0 ③보이는 경계는 OBSERVED/DERIVED만) → REST 플랫 렌더 1장 + 성도 오버레이 1장까지 진행 가능: **YES.** 그 뒤 STOP(사람 승인 대기). 비용 0.
