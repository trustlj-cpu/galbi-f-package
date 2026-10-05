# MASTER_SPEC_AMENDMENT v1.7 — ERRATUM (2026-10-06)
v1.7 G2/G3의 기하적 불가능 조건 정정. 새 해부·QA·렌더 구조 없음. 이 파일이 v1.7 G2·G3을 덮어쓴다.

[V1.7 G2/G3 ERRATUM]
W_top·A는 프레임 안의 관측점이므로 "안쪽 점 → 바깥 점" 변은 반드시 프레임을 지난다. "닫는 변 전체가 프레임 밖 ≥10mm"(G3)는 G2 체인과 동시에 성립할 수 없다. Codex의 STOP이 맞다. 조건을 "프레임 클리핑 + 화면 안에서는 허용된 변만 보임"으로 바꾼다.

[BODY_TISSUE CLOSURE]
- exact polygon construction(시계 방향): **A → FLOOR_LINE(앞→뒤) → F_end → P → PHARYNX_WALL(아래→위) → W_top → W_ext → (X_min−10, Y_max+10) → (X_min−10, Y_min−10) → (A.x, Y_min−10) → A.**
  - W_ext = PHARYNX_WALL의 마지막 변 방향으로 W_top에서 직선 연장해 **카메라 프레임 경계를 벗어난 뒤 10mm 지점**(DERIVED: 관측 선의 직선 연장). 인두 뒷벽이 비인두 쪽으로 계속되는 것을 표현한다.
  - 나머지 세 변((X_min−10,Y_max+10)→(X_min−10,Y_min−10)→(A.x,Y_min−10))은 프레임 밖 VIRTUAL.
  - (A.x, Y_min−10) → A 변은 프레임 안 **수직선**(턱 끝 아래 연조직 구간)이며, **JAW 표시 메시(아래턱·아랫니·턱 피부)가 BODY_TISSUE_DISPLAY보다 위 레이어**에서 이 구간을 덮는다. 렌더 순서 정정: … → TONGUE → BODY_TISSUE_DISPLAY → **JAW(DISPLAY)** → PALATE/VELUM/PHARYNX/치아/입술 → AIRWAY → CONTACT.
- camera-frame clipping rule: 렌더 직전 BODY_TISSUE_DISPLAY(및 모든 DISPLAY 다각형)를 **카메라 프레임 사각형으로 클립**(Sutherland–Hodgman, 사각형이라 자기교차 걱정 없음). 프레임 밖 형태는 무엇이든 상관없다 — 보이는 부분만 올바르면 된다.
- which edges may intersect frame boundary: W_top→W_ext(DERIVED 연장), (A.x,Y_min−10)→A(VIRTUAL 수직, JAW가 덮음), 그리고 FLOOR_LINE·PHARYNX_WALL 자체가 프레임 경계를 지나는 경우(관측 선).
- which edges are allowed to be visible: **OBSERVED(FLOOR_LINE, PHARYNX_WALL)와 DERIVED 직선 연장(W_top→W_ext)만.** VIRTUAL 변은 화면에 보이면 안 되고, (A.x,Y_min−10)→A는 JAW 레이어에 가려져야 한다.

[G3 REPLACEMENT]
- exact pre-render condition(자동, 렌더 전 1회): 카메라 프레임 해상도의 마스크를 만들어 ① 프레임 안에 있는 모든 VIRTUAL 변의 픽셀이 **그 위 레이어(JAW·PALATE·입술·치아·TONGUE 중 하나)의 채움 마스크에 100% 덮여 있는지** 확인 — 덮이지 않은 VIRTUAL 픽셀 수 = 0이면 PASS. ② 클립 후 다각형이 비어 있지 않고 자기교차 0. ③ 프레임 안에 보이는 경계 변은 OBSERVED 또는 DERIVED 라벨만. 실패 시: 해당 VIRTUAL 변을 프레임 밖으로 옮길 수 없는 경우(수직선 A 구간)엔 JAW 메시를 아래로 프레임 밖까지 연장해 덮는다(규칙 변경 아님, 기존 "DISPLAY 메시는 프레임 밖으로 연장" 적용).

[NEXT EXACT ACTION]
Codex가 위 체인으로 BODY_TISSUE_DISPLAY 생성(W_ext 계산 포함) → 프레임 클립 → 렌더 순서 정정(JAW를 BODY_TISSUE 위로) → 사전 조건 검사 ①②③ → REST 플랫 렌더 1장 + 성도 오버레이 1장 → **STOP(사람 승인 대기).** 다른 규칙 변경 없음, 비용 0.
