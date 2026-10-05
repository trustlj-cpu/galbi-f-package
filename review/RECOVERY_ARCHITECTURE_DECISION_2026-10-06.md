# KOREAN ARTICULATION SYSTEM — RECOVERY ARCHITECTURE DECISION
Claude Code (Claude Fable 5.1) · 2026-10-06 · 하나의 통합 결정. 이 문서가 v1.8까지의 PHASE 2 viseme 제작 방식을 대체한다. PHASE 1.5 산출물은 그대로 쓴다.

[ROOT CAUSE]
1. **혀를 "음소마다 손으로 만든 128개 좌표"로 다뤘다.** 셰이프키는 저장 방식으로는 맞지만, 서로 다른 MRI 프레임을 트레이스하거나 정점을 직접 옮겨 만든 키는 **혀의 연결 구조(혀끝이 오르면 혀날·몸통이 함께 굽는다)**를 담지 못한다. 그래서 국소 수정(v0만 올림)이 spike·column·wall을 만들었다. 이것은 ㄹ의 문제가 아니라 **저작 방식의 문제**다.
2. PHASE 1.5용 정적 게이트를 PHASE 2에 걸어 수치 FAIL로 멈추기를 반복했고(v1.8에서 수정), 그 뒤엔 시스템 대신 키 하나를 땜질했다.
3. "Blender가 정확하면 생성형이 따라온다"는 가정을 검증하지 않은 채 정밀도만 올렸다.

[KEEP / REPLACE CURRENT 128-POINT SHAPE-KEY MODEL]
- **128점 닫힌 윤곽(위상·v0 TIP·v64 ROOT_TURN·두 호 인덱싱·QA·렌더·셰이프키 저장) = KEEP.** 출력 표현으로는 맞다.
- **음소마다 손/트레이스로 만드는 셰이프키 = REPLACE.** 셰이프키의 **좌표는 사람이나 트레이스가 아니라 "파라미터 리그"가 계산해서 써 넣는다.** Blender는 그 결과를 셰이프키로 저장·혼합·렌더만 한다.

[MASTER TONGUE REPRESENTATION]
**REST 윤곽 위에 정의된 2D 파라미터 리그(TONGUE_RIG): "중심선 노드 65개 + 노드별 고정 두께(리브) + 조음 파라미터별 부드러운 변위장".**
- 중심선(한 번, REST에서 계산): v1.3 두 호 대응으로 리브를 짝지어 j=1…63에 대해 L_j = 아랫면 정점 j, U_j = 윗면 정점 128−j. 노드 N_j = (L_j+U_j)/2, N_0 = v0(TIP), N_64 = v64(ROOT_TURN). 노드마다 위 반두께 벡터 r⁺_j = U_j−N_j, 아래 반두께 r⁻_j = L_j−N_j를 **REST에서 고정**(두께는 바꾸지 않는다 → mass collapse·needle 원천 차단). 호 길이 파라미터 s_j ∈ [0,1], s=0 ROOT, s=1 TIP.
- 변형 = 중심선 노드의 변위 Δ(s) 합성. 각 조음 파라미터 p는 **고정 공간 프로파일** w_p(s)(코사인 창, 중심 c_p·폭 h_p)와 방향(국소 법선 n̂ = 구개 쪽, 접선 t̂ = 혀끝 쪽)을 가진 변위장: Δ_p(s) = a_p · w_p(s) · dir_p(s). a_p(mm)가 preset 값이다. 프로파일 폭이 혀 길이의 ≥20%라서 **spike·wall이 구조적으로 생길 수 없다.**
- 윤곽 재구성: 변형된 노드열에서 국소 접선을 다시 구하고, 리브 벡터를 **REST 접선→새 접선 회전만큼 회전**해 U_j, L_j를 다시 놓는다(두께 보존, 리브는 중심선과 함께 굽는다). TIP(v0)·ROOT_TURN(v64)은 노드 그대로. 라플라시안 평활 2회(이웃 평균 0.5). 윗면이 PALATE/VELUM 선을 넘으면 **soft clamp**: 넘은 양을 ±6노드 코사인 폭으로 분산해 선 아래 0.3mm로 되돌림(평평한 벽 방지).
- 접촉은 **1차원 스칼라 탐색**으로 만든다: 접촉 음소는 "어느 파라미터를 올리면 접촉 집합(v1.8 S_l/S_k)이 대상 구간에 닿는가"가 preset에 적혀 있고, 그 파라미터 하나를 이분법으로 키워 최소 거리 ≤0.5mm가 되는 값을 찾는다(결정론, 반복 ≤30회). 최적화·제약 솔버·물리·면적 보존 **없음** — 그래서 이전의 "제약 솔버"(고무막)·"리브+국소 면적 제약"(꺾임)과 다르다. 이것은 FK 애니메이션 리그이지 솔버가 아니다.
- 출력: 128점(v1.3 인덱싱 그대로) → 셰이프키 `V_<phoneme>`로 Blender에 기록(02 스크립트). 턱·입술·연구개는 리그 밖 기존 파라미터.

[ARTICULATOR CONTROLS] (파라미터 11개, 전부 mm 또는 도; 프로파일 중심 c·폭 h는 s 좌표)
| 파라미터 | 방향 | c | h | 의미 |
|---|---|---|---|---|
| TIP_RAISE | n̂(위) | 1.00 | 0.18 | 혀끝 올림(음수=내림) |
| TIP_ADVANCE | t̂(앞) | 1.00 | 0.18 | 혀끝 전진/후퇴 |
| BLADE_RAISE | n̂ | 0.85 | 0.22 | 혀날 올림 |
| DORSUM_FRONT_RAISE | n̂ | 0.62 | 0.30 | 앞 혀몸 올림(ㅣ·ㅈ 계열) |
| DORSUM_BACK_RAISE | n̂ | 0.35 | 0.30 | 뒤 혀몸 올림(ㄱ·ㅋ·ㅇ) |
| BODY_LOWER | −n̂ | 0.60 | 0.50 | 혀몸 전체 낮춤(ㅏ) |
| BODY_SHIFT_X | 전역 x(루트 쪽 0으로 감쇠, w=s) | — | — | 혀 전체 앞/뒤(ㅣ 앞, ㅏ·ㄱ 뒤) |
| ROOT_RETRACT | −t̂(뒤) | 0.15 | 0.20 | 인두 수축(ㅏ) |
| JAW_OPEN_DEG | JAW_PIVOT 회전 | — | — | 혀 노드에 가중치 w(s)=0.3+0.7s로 적용 + 아래턱·아랫입술 강체 |
| LIP_APERTURE / LIP_SPREAD / LIP_PROTRUDE | 입술 스트로크 | — | — | 입술 간격·옆·앞 |
| VELUM_OPEN | VELUM_LINE 셰이프키 | — | — | 0=폐쇄(구강음), 1=열림(비음·휴지) |
각 파라미터 상한(cap): 변위 ≤ 혀 길이의 25%, JAW ≤ 20°. 상한은 spike 방지가 아니라 트레이스 사고 방지용.

[PHONEME PRESET MODEL]
음소 = **파라미터 벡터 + 접촉 지시 + 조음기관 지배도**(JSON, `SYSTEM/PHONEME_LIBRARY/presets/<phoneme_id>.json`). 전체 혀 모양은 저장하지 않는다(리그가 매번 계산, 결정론이라 결과는 동일). 이것이 시리즈 마스터 유닛이다: 새 음소 = preset 1개, 새 단어 = 음소 preset 조합 + 타이밍, 새 언어 = preset 몇 개 추가.
```json
{"phoneme_id":"KO_l","gesture":"anterior_alveolar_tap_lateral",
 "params":{"TIP_RAISE":"SOLVE","TIP_ADVANCE":1.0,"BLADE_RAISE":"0.5*TIP_RAISE","BODY_LOWER":2.0,"BODY_SHIFT_X":0,"DORSUM_FRONT_RAISE":0,"DORSUM_BACK_RAISE":0,"ROOT_RETRACT":0,"JAW_OPEN_DEG":6,"LIP_APERTURE":8,"LIP_SPREAD":0,"LIP_PROTRUDE":0,"VELUM_OPEN":0},
 "contact":{"set":"S_l","target":"alveolar","solve_param":"TIP_RAISE","gap_mm":0.5},
 "dominance":{"tongue":1.0,"jaw":0.5,"lips":0.2,"velum":0.6},
 "mri_reference":{"speaker":"USC_pk_2015","frame":"<id>","role":"gesture/contact-zone/posture check only"},
 "validation":{"status":"DRAFT"}}
```
갈비 FIRST SET 시작값(Codex가 상한 안에서 v1.8 A 관계 검사로 조정, 정점 편집 금지):
- REST: 전부 0, JAW 0, LIP 2, VELUM 1.
- KO_k: DORSUM_BACK_RAISE=SOLVE(S_k→연구개 구간), BODY_SHIFT_X=−3, TIP_RAISE=−1, JAW 4, LIP 8, VELUM 0.
- KO_a: BODY_LOWER 6, ROOT_RETRACT 4, BODY_SHIFT_X −2, JAW 14, LIP 11, VELUM 0.
- KO_l: 위 JSON.
- KO_p_CLOSE: 혀 전부 0(REST), JAW 2, LIP_APERTURE 0(스냅), VELUM 0. KO_p_RELEASE: LIP 4, JAW 5, 혀 0.
- KO_i: DORSUM_FRONT_RAISE=SOLVE(간격 2mm 목표, 접촉 아님: 대상 PALATE 앞 절반, gap_mm 2.0), BODY_SHIFT_X +3, TIP_RAISE −1, JAW 3, LIP_APERTURE 4, LIP_SPREAD +3, VELUM 0.

[MRI ROLE]
**좌표 정답이 아니다.** 역할 세 가지만: ① 제스처 방향·접촉/수축 영역·상대 자세(높낮이·전후)를 정할 때의 **근거**(preset 값의 출발점) ② 완성된 viseme의 **관계 검사**(v1.8 A표: 어디가 닿고 어디가 떨어지는가, 몇 mm 범위) ③ 승인 시트의 **나란히 보기**(오버레이·픽셀 일치 요구 없음). 1mm 윤곽 일치·면적 일치는 요구하지 않는다. 화자 차이·동시조음 차이는 preset으로 흡수한다.

[BLENDER ROLE]
**2(결정론 기하·모션 제어 층)로 확정.** 최종 해부 모델이 아니다. Blender는 리그가 쓴 셰이프키를 저장·혼합·키프레임·렌더하고, 생성형에 넘길 **구조 가이드 영상**(세 가지 상세도)을 뽑는다. 해부 정밀도 상한 = "사람이 제스처를 즉시 읽고 혀가 혀로 보임".

[GENERATIVE MODEL ROLE]
최종 외형(피부·질감·조명·스타일)을 가이드 위에 입힌다. 조음(위치·타이밍)은 가이드가 결정하고 생성형은 **따라가야** 하며, 얼마나 따라가는지는 가정이 아니라 **측정**한다(아래 실험). 실험 전까지 가이드 상세도를 올리는 작업은 금지.

[GEN MODEL FOLLOWING TEST] — 갈비 리그 모션이 나오는 즉시, 추가 정밀화 **전에**
- 입력: 같은 갈비 모션(리그 출력) 3초, 같은 카메라, 세 상세도 가이드 영상 — G1 rough(혀·구개·입술·턱 단색 실루엣, 선 없음), G2 medium(G1 + 외곽선 + 기도 밝게), G3 high(G2 + 음영·질감 스타일). 같은 생성 모델(사용자가 크레딧을 가진 영상 모델의 video-to-video/구조 조건 모드), 같은 프롬프트("sagittal cutaway of a human head, tongue, palate, lips, medical illustration look; follow the input video motion exactly"), 같은 시드(가능하면), 각 1회 → 총 3클립(비용 승인 후).
- 측정(출력 클립마다, 파이썬+사람): ① 접촉 위치 — ㄹ·ㄱ 접촉 프레임에서 혀가 닿은 자리가 가이드와 같은 구간인가(사람 판정 + 가이드 접촉 마스크와 IoU) ② 제스처 방향 — 5개 사건(ㄱ 뒤 상승, ㅏ 하강, ㄹ 전방 상승, ㅂ 폐쇄/해제, ㅣ 전상 상승)이 읽히는가(사람, 5/5) ③ 타이밍 — 사건 프레임이 가이드 대비 ±2프레임(윤곽 실루엣 차분으로 자동) ④ 프레임 일관성 — 연속 프레임 실루엣 IoU ≥0.9, 가이드 대비 혀 실루엣 IoU 프레임별 중앙값(G1/G2/G3 비교).
- 결정 규칙: G1이 ①②③을 통과하면 **production 가이드 = rough**로 확정(정밀화 중단). G2만 통과하면 medium. G3도 실패하면 생성형이 조음을 못 따라오는 것이므로 "단면은 Blender 직접 렌더(스타일 단계) + 생성형은 실사만"으로 파이프라인을 되돌린다.
- 순서: [GALBI PILOT IMPLEMENTATION] R1~R4 → 이 실험 → 결과에 따라 PHASE 8 스타일 또는 파이프라인 확정.

[GALBI PILOT IMPLEMENTATION]
R1 리그 구축(비용 0): 승인 REST에서 중심선·리브·s 좌표 계산 → `tongue_rig.py`(deform·reconstruct·clamp·contact-solve·export) → 단위 테스트: 모든 파라미터 0 → REST 128점과 최대 편차 ≤0.05mm; TIP_RAISE 5mm → 자기교차 0, 최소 리브 두께 = REST와 동일, 최대 곡률 ≤ 2×REST.
R2 preset 6+1개 생성 → 셰이프키 기록 → v1.8 S·A 검사 + 형상 검사(아래 PASS) → 승인 시트(소스 MRI 나란히 | 리그 결과 | 파라미터 표).
R3 갈비 모션: 타임라인은 **감사된 사건 프레임**(ㄱ→ㅏ→ㄹ 접촉 F32–54→ㅂ 폐쇄 F63–83→해제 F84→ㅣ)을 임시 사용(정렬 파이프라인은 PHASE 3에서 교체), 스위칭 + 2프레임 램프, 24fps.
R4 가이드 렌더 3종(G1/G2/G3) + 플랫 진단판 → 사람 게이트 → 생성형 추종 실험.

[WHAT TO DISCARD]
- V3/V4/V5 ㄹ 셰이프키와 그 제작 방식(정점 편집·v0 단독 올림) → ARCHIVE.
- "음소마다 MRI 전체 윤곽 트레이스 → 셰이프키" 저작 방식 → 중단(트레이스 파일은 참고로 보존).
- 514프레임 natural reselection 결과를 viseme 원천으로 쓰는 것 → 중단(참고만).
- PHASE 2 정적 게이트 상속(이미 v1.8에서 제거) — 재도입 금지.
- 수치 FAIL로 결과물을 보기 전에 멈추는 운영 → 형상 검사·사람 게이트 중심으로.

[WHAT TO KEEP]
승인 REST tract·REST tongue·128점 위상·v0/v64·v1.3 두 호 인덱싱·카메라 프레임·표시 구조(v1.7+ERRATUM, BODY_TISSUE 동적 위 경계)·pk_2015 원본·Blender 퍼펫 오브젝트와 셰이프키 메커니즘·v1.8 접촉 집합과 A 관계표(수용 검사로)·정직성 라벨·정렬 파이프라인 계획(PHASE 3)·감사된 사건 프레임.

[ONE-PASS CODEX IMPLEMENTATION PLAN] (한 번에, 중간 STOP은 R2 승인 시트 1회뿐)
파일: `SYSTEM/RIG/tongue_rig.py`(centerline·deform·reconstruct·clamp·solve·export, numpy만) · `SYSTEM/RIG/rig_profiles.json`(c·h·방향·cap) · `SYSTEM/PHONEME_LIBRARY/presets/{REST,KO_k,KO_a,KO_l,KO_p_CLOSE,KO_p_RELEASE,KO_i}.json` · `SYSTEM/AUTOMATION/02b_presets_to_shapekeys.py`(blender -b: preset→리그→128점→셰이프키 `V_<id>`) · `SYSTEM/QA/shape_checks.py`(자기교차·최소 리브 두께 비·최대 곡률·접촉 거리·A 관계) · `SYSTEM/AUTOMATION/09b_render_guides.py`(G1/G2/G3 + flat) · `PILOTS/GALBI/runs/<run>/`.
순서: ① R1 리그 + 단위 테스트 통과 ② preset 7개 → 셰이프키 → 형상·A 검사 표 ③ 승인 시트 7장(각: 소스 MRI 프레임 | 리그 결과 플랫 렌더 | 파라미터 값·검사 결과) → **STOP(사람 게이트)** ④ 승인 후 R3 모션 → R4 렌더 4종(flat, G1, G2, G3) 각 mp4 → 파티 게시 → STOP(생성형 실험 승인 대기).
규칙: 정점 직접 편집 금지(preset 값만 조정, cap 안), 관측 REST 변형 금지, 복구 가능 오류는 진단→수정→재시도, 메모리 빨강 STOP, 비용 0(생성형 실험은 별도 승인).

[PASS CONDITION]
- 자동(형상): 자기교차 0 · 최소 리브 두께 ≥ REST의 60%(두께 보존 구조상 100%가 정상; 평활·clamp로 줄어도 60% 미만이면 FAIL) · 최대 곡률 ≤ 2×REST 최대 곡률(spike·needle 검출) · 인접 정점 간 변 길이 ≤ 3×REST 평균(elongation·wall 검출) · 접촉 음소 최소 거리 ≤0.5mm(solve 결과) · v1.8 S 전부.
- 자동(관계): v1.8 A표(mm 값은 ±τ, "읽힘" 중심).
- 사람(게이트, 7장+모션 1개): ㄱ=뒤 혀 상승 / ㅏ=낮고 열림 / ㄹ=짧은 치조 혀끝·혀날 제스처 / ㅂ=닫힘→해제 / ㅣ=앞·위 상승이 **즉시 읽힘**, 그리고 spike·needle·수직 벽·비정상 신장·질량 붕괴 **없음**, 혀가 혀처럼 보임. 7/7 ✓ + 모션 ✓ = PASS.
- FAIL 처리: 해당 preset 값만 조정(최대 2회), 그래도 FAIL이면 그 파라미터 프로파일(c·h) 1회 조정. 정점 편집·새 솔버·새 QA 금지.

[NEXT EXACT ACTION]
Codex: ONE-PASS 계획 ①②③ 실행 → 승인 시트 7장을 파티에 올리고 STOP. (ㄹ만 따로 고치지 않는다. 7개 preset을 같은 리그로 한 번에 만든다.) 사람: 시트 승인 → ④ → 생성형 추종 실험 승인(비용 확인).
