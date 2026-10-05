# FINAL DISPLAY ARCHITECTURE — LEAD ARCHITECT DECISION (Claude Code · 2026-10-06)

[ROOT CAUSE]
퍼펫에 **혀 아래·뒤를 이루는 머리·목 연조직의 표시 층(substrate)이 없었다.** 그래서 혀(또는 BASE_FILL)만으로 화면에 닫힌 다각형을 만들면 그 다각형의 **닫는 변이 반드시 어딘가에서 외곽선으로 드러난다**(큰 판, 분홍 안개, 긴 하단 대각선은 전부 같은 원인의 다른 증상). 실제 정중앙 단면·의학 일러스트에서 혀는 독립된 덩어리가 아니라 **위·앞면만 구분되고 아래·뒤는 주변 조직으로 녹아드는 형태**다. 따라서 "혀를 아래까지 닫아 하나의 분홍 덩어리로 만든다"는 전제는 **폐기**한다. BASE_FILL은 QA 지원 기하로만 남고 화면에서 사라진다.

[FINAL DISPLAY ARCHITECTURE]
렌더 순서(뒤→앞): BACKGROUND → HEAD_SILHOUETTE·JAW·SKULL(표시용 바탕) → **BODY_TISSUE_DISPLAY(혀 아래·뒤 연조직, 표시 전용)** → PALATE/VELUM/PHARYNX/치아/입술(표시) → **TONGUE** → AIRWAY 하이라이트(선택, 스타일 단계) → CONTACT_HIGHLIGHT. 단, BODY_TISSUE_DISPLAY의 **윗부분 띠는 TONGUE보다 앞(위 레이어)**에 그려 혀의 아래·뿌리 가장자리를 가린다(아래 규칙). TONGUE_BASE_FILL은 렌더에서 제외.

[TONGUE]
- visible: YES — 기존 128점 해부·애니메이션 다각형 **그대로**, 추가 닫힘·오프셋 없음.
- role: 유일한 "혀" 시각 실루엣. 화면에서 보이는 것은 윗면(65–127, 0)·혀끝·앞쪽 아랫면(1–~18, 구강저 위 설하 구간)뿐이고, 나머지 아랫면·뿌리(≈19–75)는 BODY_TISSUE_DISPLAY 상단 띠에 **가려진다**(occluded). 애니메이션·QA 기하는 변하지 않는다.

[BASE_FILL]
- camera visible: **NO** — `hide_render = True`, 뷰포트·QA 계산 전용.
- role: BASE COVERAGE QA(혀 아랫면이 2mm 넘게 위로 뜨지 않음)의 기준 기하. 시각 실루엣 생성에 쓰지 않는다. opacity·blur·union·D 조정 전부 종료.

[BODY_TISSUE_DISPLAY]
- required: **YES.**
- source: **기존 OBSERVED 성도 선 + 카메라 프레임 경계**만. 새 트레이스 없음. 측정·QA 원천 아님(DISPLAY_ONLY, grade 라벨 "DISPLAY").
- geometry rule(닫힌 다각형, 시계 방향, DISPLAY_ONLY): 위 경계 = **FLOOR_LINE 전체(앞끝 A → 뒤끝 F_end) → P(PHARYNX_WALL 최하단) → PHARYNX_WALL을 따라 위로 → W_top(PHARYNX_WALL 최상단, 연구개 뒤)**. 그 다음 프레임 밖으로 닫는다: **W_top → (X_min − 10mm, W_top.y) → (X_min − 10mm, Y_min − 10mm) → (A.x, Y_min − 10mm) → A.** 여기서 X_min·Y_min은 아래 카메라 프레임의 뒤쪽·아래쪽 경계. 즉 위 경계는 관측 선(구강저·인두 벽)이고, 닫는 변 3개는 **전부 프레임 밖 10mm**에 있다. 앞쪽(턱 끝·턱밑)은 기존 JAW/HEAD 표시 메시가 덮는다(그것들도 같은 프레임 밖 연장 규칙 적용).
- material/layer rule: 단색 평면, 혀보다 채도 낮고 한 단계 어두운 같은 계열(예: 혀 #D98A86 → 조직 #C9A39A 계열, 스타일 단계에서 확정). 윗경계 중 **FLOOR_LINE과 PHARYNX_WALL 부분만** 가는 외곽선(관측 선이라 그려도 됨), 프레임 밖 변은 선 없음. 레이어: TONGUE 위에 그리되 **PALATE·치아·입술·AIRWAY보다는 아래**. 구현은 Blender 오브젝트 z-오프셋(TONGUE z=0.02, BODY_TISSUE z=0.025, PALATE 등 z≥0.03) 또는 렌더 순서 — 결정론이면 어느 쪽이든 됨.

[CUTAWAY / CAMERA RULE]
- exact visible bounds(직교 카메라, mm, ALVEOLAR_POINT 원점): **X_max = ALVEOLAR_POINT.x + 45mm**(입술 앞 여유), **X_min = PHARYNX_WALL min x − 20mm**, **Y_max = PALATE_LINE max y + 30mm**(코·두개 일부), **Y_min = P.y − 25mm**(턱밑·목 일부). 가로세로는 1920×1080에 맞춰 긴 쪽 기준으로 letterbox 없이 ortho_scale = max(ΔX, ΔY×16/9). 이 값은 anatomy_tract.json `camera_frame`에 저장하고 **전 viseme·전 단어 공통**(카메라를 움직이는 최종 스타일 단계에서도 이 프레임을 기준으로 ±10% 안에서만).
- how support closing edges are kept off-screen/occluded: 규칙 하나 — **모든 DISPLAY_ONLY 다각형의 VIRTUAL 닫는 변은 카메라 프레임 바깥 ≥10mm에 둔다**(BODY_TISSUE_DISPLAY, JAW, HEAD 바탕 모두). 관측 선(구강저·인두 벽·구개·입술)만 프레임 안에서 경계로 보인다. 혀의 아래·뿌리 가장자리는 BODY_TISSUE_DISPLAY 상단 띠에 가려진다. 자동 검사: 렌더 전에 각 DISPLAY 다각형의 VIRTUAL 변 끝점이 프레임 밖인지 확인(실패 시 그 변을 프레임 밖으로 연장).

[PHASE 1.5 HUMAN GATE SUCCESS CRITERION]
REST 렌더 1장(위 구조·플랫 재질)에서 사용자가 다음 5개를 ✓: ① 혀 윗면·혀끝이 또렷하고 아래·뿌리는 조직 속으로 자연스럽게 들어간다(인공 대각선·판·안개 없음) ② 구개를 따라 기도가 얇게 보인다 ③ 구강저·인두 벽 선이 자연스러운 경계로 보인다 ④ 프레임 안에 어떤 인공 닫힘 변도 없다 ⑤ 전체가 "사람 옆 단면"으로 읽힌다. + 성도 오버레이 시트 1장 ✓(MRI와 같은 구조). 통과 = PHASE 1.5 PASS.

[AMENDMENT REQUIRED]
**YES** — MASTER_SPEC_AMENDMENT_v1.7.md(§3에 BODY_TISSUE_DISPLAY·카메라 프레임·렌더 순서 추가, BASE_FILL 비렌더 확정, v1.6 ERRATUM 유지).

[NEXT EXACT ACTION]
Codex가 윤곽·QA 수정 없이 ① anatomy_tract.json에 `camera_frame` 계산·저장 ② BODY_TISSUE_DISPLAY 다각형 생성(위 규칙, DISPLAY 라벨) ③ BASE_FILL `hide_render=True` ④ 렌더 순서/z 설정 ⑤ Blender `-b`로 REST 플랫 렌더 1장 + 성도 오버레이 시트 1장 → **STOP(사람 승인 대기).** 비용 0.
