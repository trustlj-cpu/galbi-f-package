# MASTER_IMPLEMENTATION_SPEC — AMENDMENT v1.7 (2026-10-06)
기준: v1.0 @ 74b5a011(불변) + v1.1~v1.6 + v1.6 ERRATUM. v1.7은 §3에 표시 구조를 추가하고 BASE_FILL의 렌더 역할을 확정한다. 근거: review/DISPLAY_ARCHITECTURE_DECISION_2026-10-06.md.

## G1. §3 — 표시 구조(신설)
- 전제 폐기: 혀를 아래까지 닫아 독립 덩어리로 보이게 하지 않는다. 혀의 아래·뿌리 가장자리는 주변 연조직 표시 층에 가려진다.
- 렌더 순서(뒤→앞): BACKGROUND → HEAD/JAW/SKULL 바탕(DISPLAY) → TONGUE → **BODY_TISSUE_DISPLAY** → PALATE/VELUM/PHARYNX/치아/입술(DISPLAY) → AIRWAY(스타일) → CONTACT_HIGHLIGHT. z 오프셋 또는 렌더 순서로 결정론 보장.
- TONGUE_BASE_FILL: `hide_render = True`. QA 전용. 시각 조정 금지.

## G2. §3 — BODY_TISSUE_DISPLAY(DISPLAY_ONLY, grade "DISPLAY")
- 다각형: A(FLOOR_LINE 앞끝) → FLOOR_LINE → F_end → P(PHARYNX_WALL 최하단) → PHARYNX_WALL 위로 → W_top → (X_min−10, W_top.y) → (X_min−10, Y_min−10) → (A.x, Y_min−10) → A. 닫는 변 3개는 프레임 밖.
- 재질: 혀보다 채도 낮고 한 단계 어두운 같은 계열 단색. 외곽선은 FLOOR_LINE·PHARYNX_WALL 구간만.
- QA·측정 원천 아님.

## G3. §3 — 카메라 프레임(anatomy_tract.json `camera_frame`, 전 단어 공통)
- X_max = ALVEOLAR_POINT.x + 45mm, X_min = PHARYNX_WALL min x − 20mm, Y_max = PALATE_LINE max y + 30mm, Y_min = P.y − 25mm. ortho_scale = max(ΔX, ΔY×16/9). 최종 스타일의 카메라 이동은 이 프레임 기준 ±10%.
- 규칙: 모든 DISPLAY_ONLY 다각형의 VIRTUAL 닫는 변은 프레임 밖 ≥10mm. 렌더 전 자동 확인.

## G4. PHASE 1.5 HUMAN GATE
- REST 플랫 렌더 1장 체크 5항(혀 윗면·혀끝 또렷·아래는 조직으로 녹아듦 / 기도 보임 / 구강저·인두 선 자연 / 프레임 안 인공 변 0 / 사람 옆 단면으로 읽힘) + 성도 오버레이 1장 ✓ → PHASE 1.5 PASS.
