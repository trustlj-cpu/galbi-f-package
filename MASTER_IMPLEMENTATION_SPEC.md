# KOREAN_PRONUNCIATION_SHORTS — MASTER IMPLEMENTATION SPEC v1.0

LEAD ARCHITECT / SOURCE OF TRUTH: Claude Code (Claude Fable 5.1) · 2026-10-05
IMPLEMENTER: Codex (로컬, PROJECT ROOT `/Users/yiseungmin/Documents/Codex/KOREAN_PRONUNCIATION_SHORTS`) · 환경 MacBook Air M2 8GB
원칙: 삭제 금지 · 유료 금지(사전 승인 없이) · 생성형으로 조음 생성 금지 · 커스텀 혀 물리 금지 · 녹음 타이밍이 기준 · 교육적 시각화와 측정 구분 · 100단어 재사용성 > 갈비 최적화 · 한 PHASE PASS 전 다음 PHASE 금지.

**이 문서의 한 가지 설계 변경(명시):** 퍼펫 구현체를 Grease Pencil이 아니라 **Blender 메시 + 셰이프키**로 확정한다. 이유 ① Blender 5.x의 GP v3 API가 버전마다 바뀌어 Codex 자동화 실패 위험이 큼 ② 셰이프키 가중 혼합 = 동시조음 혼합이 Blender 엔진에 내장돼 있어 코드가 줄고 결정론적 ③ QA가 정점 좌표를 직접 읽음 ④ 기존 15_GP의 128점 키를 정점으로 그대로 옮길 수 있음. "2.5D 컷아웃·같은 위상·결정론 보간"이라는 clean-slate 성질은 동일하다. 외곽선은 Freestyle로 그린다.

━━━━━━━━━━━━━━━━━━━━
## [0. FINAL SYSTEM DEFINITION]
━━━━━━━━━━━━━━━━━━━━
**한 줄:** 실제 녹음을 넣으면, 음소 경계를 자동으로 찾고, 음소마다 미리 승인된 "옆 단면 모양(viseme)"을 꺼내 섞어서, 머리 단면 인형이 그 녹음에 맞춰 움직이는 영상을 자동으로 뽑는 공장.

입력: ① word(한글·로마자) ② audio(WAV, 그 단어만 잘린 토큰) ③ speaker(화자 id, L1, native/learner) ④ transcription 3종 — TARGET(한국어 정답 음소열), L1_LEXICAL(B: 그 언어의 차용형, 예 カルビ), ACTUAL(C: 실제로 낸 음소열, 사람이 듣고 적음).
내부: 정규화 → 강제 정렬(음소 시작·끝 시각) → viseme 조회(ACTUAL 음소열 기준) → 동시조음 혼합(프레임별 각 조음기관 가중치) → Blender 퍼펫에 가중치 키프레임 자동 입력(셰이프키·턱 회전·입술) → 렌더(플랫 진단판 / 최종 스타일판) → 자동 QA + 사람 게이트 → 보고서.
출력: `<word>_<speaker>_<style>.mp4`, `events.json`(접촉·폐쇄 프레임), `animation.json`(프레임별 가중치), `.blend`, `TextGrid`, `qa_report.md`.

**사람이 하는 일(단어당):** 녹음 확보 · ACTUAL 음소열 듣고 적기(학습자) · 정렬 경계 확인(Praat, 플래그 난 것만) · 새 음소가 나오면 viseme 1장 승인 · 최종 영상 시각 승인.
**자동으로 되는 일:** 정규화·정렬·viseme 조회·혼합·키프레임·렌더·수치 QA·보고서·두 번째 언어 재실행.

━━━━━━━━━━━━━━━━━━━━
## [1. PROJECT RESTRUCTURE]
━━━━━━━━━━━━━━━━━━━━
기존 폴더는 **옮기지 않는다**(삭제·이동 0). 루트에 새 트리를 만들고, 기존 자산은 `SYSTEM/DOCS/ASSET_MANIFEST.md`에 지위를 적는다. 재사용 자산은 **복사**(원본 보존).

```
KOREAN_PRONUNCIATION_SHORTS/
  SYSTEM/
    PUPPET/            puppet_master.blend, anatomy_reference.json(구개·치조·앞니·턱축·단위), build 스크립트 출력
    PHONEME_LIBRARY/   phonemes.json, visemes/<viseme_id>.json(점 좌표+메타), visemes/<viseme_id>.png(승인용 그림), VALIDATION_LOG.md
    ALIGNMENT/         mfa 설정, custom_dictionary.txt, 파싱 스크립트
    COARTICULATION/    dominance_profiles.json, engine 스크립트
    AUTOMATION/        01~10 스크립트(§9)
    QA/                qa_rules.json, qa 스크립트, 보고 템플릿
    STYLE/             style_flat.json(진단), style_final.json(최종), palette, grain 설정
    TESTS/             결정론 재현 테스트(같은 입력→같은 해시)
    DOCS/              이 SPEC 사본, ASSET_MANIFEST.md, PHASE_LOG.md
  DATA/
    AUDIO/raw/         원본(읽기 전용)
    AUDIO/tokens/      <token_id>.wav (16k mono, 정규화)
    TRANSCRIPTIONS/    <token_id>.json (TARGET/L1_LEXICAL/ACTUAL)
    ALIGNMENTS/        <token_id>.TextGrid, <token_id>.events.json
    SPEAKERS/          speakers.json
  PILOTS/
    GALBI/             runs/<run_id>/ (animation.json, .blend, mp4, qa_report)
  GALBI_PILOT/         (기존 그대로, 지위는 아래 표)
  FLOW_TESTS/ HYBRID_ARTICULATION/ 등 기존 (그대로)
```

기존 자산 지위(ASSET_MANIFEST.md에 그대로 기재):
| 자산 | 지위 | 처리 |
|---|---|---|
| GALBI_PILOT/05 (articulatory rig, neutral master) | ARCHIVE | 열지 않음 |
| GALBI_PILOT/06 KO_GALBI_MOTION_MASTER_RENDER_V2 | REFERENCE ONLY | 사건 프레임(F32/54/55/59/47/63/68/83/84)만 DOCS에 기록. 시스템 타이밍으로 쓰지 않음 |
| GALBI_PILOT/07 JA_KARUBI motion | REFERENCE ONLY | 동일 |
| 정적 기하(구개·치조·앞니·턱축, 어느 파일에 있든) | MIGRATE | `SYSTEM/PUPPET/anatomy_reference.json`로 복사·단위 mm |
| F1 neutral tongue contour | REFERENCE ONLY | REST viseme 비율 참고 |
| GALBI_PILOT/08·09 (F 실험, 레이어) | ARCHIVE | — |
| 제약 솔버·중심선 리브·키포즈·LOCK-AND-FILL 계열(12·13·14 등) | ARCHIVE | — |
| GALBI_PILOT/15_GP_KEYSHAPE_TONGUE (128점 K1~K5) | MIGRATE(seed) | §5 규칙으로 승격 심사 후 visemes/로 복사 |
| 기존 QA 코드(contact·palate·self-intersection·topology·timing) | MIGRATE | `SYSTEM/QA/`로 복사 후 §10 규칙에 맞춰 입력 형식만 변경 |
| FLOW_TESTS V1 영상 | REFERENCE ONLY | 최종 스타일 팔레트·비율 참고 |
| HYBRID_ARTICULATION, MetaHuman/UE 산출물 | ARCHIVE | — |
| AI-Hub·네이티브 오디오 | KEEP | `DATA/AUDIO/raw/`에 복사(원본 경로 기록) |

━━━━━━━━━━━━━━━━━━━━
## [2. CORE DATA MODEL]
━━━━━━━━━━━━━━━━━━━━
전부 JSON, UTF-8, 단위 mm·초·프레임(24fps 고정). 좌표계: 원점 = 윗앞니 절단연 끝, +x = 입술 방향(앞), +y = 위. 모든 모양은 이 좌표계.

**phonemes.json** (음소 = 언어별 소리 단위)
```json
{"phoneme_id":"KO_l","ipa":"l","language":"ko","class":"lateral_approximant","place":"alveolar","notes":"어말/자음 앞 ㄹ"}
{"phoneme_id":"JA_r","ipa":"ɾ","language":"ja","class":"tap","place":"alveolar"}
```
**visemes/<viseme_id>.json** (viseme = 음소에 대응하는 단면 모양. 한 음소 ↔ 기본 viseme 1개, 언어별로 다른 viseme 허용)
```json
{"viseme_id":"KO_l_lateral","phoneme_ids":["KO_l"],"language_scope":["ko"],
 "articulators":{
   "tongue":{"points":[[x,y],...128],"tip_index":0,"segments":{"tip":[0,31],"dorsum":[32,79],"root":[80,127]},"tip_target_mm":[x,y]},
   "jaw":{"opening_deg":6.0},
   "lips":{"upper_points":[...24],"lower_points":[...24],"aperture_mm":9.0,"spread":0.0,"protrusion":0.0},
   "velum":{"open":0.0}},
 "contact":{"region":"alveolar_ridge","type":"tip_contact","lateral_release":true},
 "dominance":{"tongue":1.0,"jaw":0.5,"lips":0.2,"velum":0.6},
 "windows_ms":{"anticipatory":{"tongue":80,"jaw":60,"lips":40,"velum":60},"carryover":{"tongue":60,"jaw":60,"lips":40,"velum":60}},
 "reference_sources":[{"type":"rtMRI","id":"SeeingSpeech /l/ midsagittal","use":"shape reference"},{"type":"text","id":"Ladefoged & Johnson, lateral approximant","use":"place/manner"}],
 "validation":{"status":"DRAFT|REVIEWED|VALIDATED","validated_by":"","date":"","notes":""},
 "educational_disclaimer":"illustrative articulation consistent with cited references; not a measurement"}
```
**speakers.json**: `{"speaker_id":"KO_NAT_01","L1":"ko","type":"native|learner","target_language":"ko","source":"user_recording|AIHub_71490","consent":"ok"}`
**TRANSCRIPTIONS/<token_id>.json** (세 음소열을 절대 섞지 않음)
```json
{"token_id":"galbi_KO_NAT_01_t1","word":"갈비","speaker_id":"KO_NAT_01",
 "TARGET":["KO_k","KO_a","KO_l","KO_b","KO_i"],
 "L1_LEXICAL":null,
 "ACTUAL":["KO_k","KO_a","KO_l","KO_b","KO_i"],
 "ACTUAL_source":"human_listening","ACTUAL_by":"","notes":""}
```
학습자 예: `"L1_LEXICAL":["JA_k","JA_a","JA_r","JA_u","JA_b","JA_i"]`, `"ACTUAL":["KO_k","KO_a","JA_r","KO_b","KO_i"]` (실제 들린 대로).
**ALIGNMENTS/<token_id>.events.json**: `{"fps":24,"phones":[{"phoneme_id":"KO_l","start_s":0.312,"end_s":0.398,"start_f":7,"end_f":10,"confidence":"mfa|manual","viseme_id":"KO_l_lateral"}],"derived_events":[{"type":"tip_contact_on","f":8},{"type":"lip_closure_on","f":12}]}`
**ARTICULATOR 열거:** tongue, jaw, lips, velum (고정 4개).
**COARTICULATION PROFILE:** `dominance_profiles.json` — 음소 class별 기본 dominance·window(§8). viseme의 값이 있으면 그것이 우선.

━━━━━━━━━━━━━━━━━━━━
## [3. SAGITTAL PUPPET MASTER]
━━━━━━━━━━━━━━━━━━━━
`SYSTEM/PUPPET/puppet_master.blend`, Blender 4.2 LTS 또는 5.x. 모든 오브젝트는 XY 평면(z=0), 직교 카메라 +Z에서 내려다봄(ortho_scale = 180mm), 단위 mm(씬 unit scale 0.001).

| 오브젝트 | 종류 | static/animated | 부모 | 피벗 | 위상 | 키 가능 파라미터 |
|---|---|---|---|---|---|---|
| HEAD_SILHOUETTE | 메시(ngon) | static | ROOT(Empty) | 원점 | 임의 | 없음(스타일에서 교체) |
| SKULL_PALATE | 메시 | static | ROOT | 원점 | 임의 | 없음. **구개 하단 곡선 정점 64개**는 별도 커브 `PALATE_LINE`으로도 보관(충돌·기도 계산용) |
| UPPER_TEETH | 메시 | static | ROOT | 원점 | 임의 | 없음 |
| PHARYNX_WALL | 메시 | static | ROOT | 원점 | 임의 | 없음 |
| JAW | 메시(아래턱+턱 피부) | animated(강체) | JAW_PIVOT(Empty, 턱관절 위치) | 턱관절 | 임의 | JAW_PIVOT.rotation_euler.z (0=닫힘, +=열림) |
| LOWER_TEETH | 메시 | static-in-jaw | JAW_PIVOT | — | 임의 | 없음 |
| TONGUE | 메시 ngon **128정점 고정**, 정점 0=혀끝, 시계 방향, 삼각분할은 팬 | animated(셰이프키) | ROOT (턱 아님 — 혀 바닥 부착은 viseme 모양 안에 포함) | 원점 | **모든 viseme 동일 128점·동일 순서** | 셰이프키 `V_<viseme_id>` 값 0~1 |
| UPPER_LIP / LOWER_LIP | 메시 24정점 각각 | animated(셰이프키) | UPPER_LIP→ROOT, LOWER_LIP→JAW_PIVOT | — | 24점 고정 | 셰이프키 `V_<viseme_id>` |
| VELUM | 메시 16정점 | animated(셰이프키 2개: open/closed) | ROOT | — | 16점 | `V_open` |
| AIRWAY | 메시(프레임마다 파이썬이 정점 갱신: 구개선 ↔ 혀 윗면) | animated(스크립트) | ROOT | — | 64+64 | 없음(파생) |
| CONTACT_HIGHLIGHT | 메시 작은 원 | animated(가시성) | ROOT | 치조점 | — | hide_render 키 |
| CAM_ORTHO | 카메라 | static(최종 스타일에서 1~2mm 흔들림 키) | ROOT | — | — | location |

혀 결정: **하나의 동일 위상 128점**, 분절은 별도 오브젝트가 아니라 **인덱스 범위**(tip 0~31, dorsum 32~79, root 80~127)로만 다룬다(QA·dominance용). 변형은 **셰이프키 선형 혼합**(Basis = REST). 15_GP의 128점이 이 규격이면 그대로 정점으로 옮긴다(점 순서·혀끝 인덱스 확인 필수).

━━━━━━━━━━━━━━━━━━━━
## [4. PHONEME VISEME LIBRARY]
━━━━━━━━━━━━━━━━━━━━
결정: **최소 subset부터.** FIRST LIBRARY SET = 6개 viseme + REST:
`REST`(입 다문 중립: 혀 낮고 평평, 턱 0°, 입술 닫힘·이완 — Basis이자 무음 구간), `KO_k_velar`(ㄱ), `KO_a_open`(ㅏ), `KO_l_lateral`(ㄹ), `KO_p_bilabial`(ㅂ: 입술 폐쇄 지배, 혀는 REST 모양 + dominance 0.2), `KO_i_front`(ㅣ). ㄱ은 어두 평음이라 `KO_k_velar`로 통일(격음·경음은 추후).

각 viseme에서 **사람이 결정**: 혀 모양의 "맞음"(조음 위치·방식·전체 실루엣), 입술 모양, 승인. **수치화(자동)**: 혀끝 목표 좌표(치조점과의 거리), 턱 각도, 입술 간격·벌림, 구개 관통 0, 면적(참고), 분절 높이.
참고 자료 사용 규칙(정직성): ① Seeing Speech / USC-TIMIT의 해당 **영어·IPA** 음소 정중앙 MRI 프레임을 **형태 참고**로 트레이스(비율·곡률) ② 한국어 고유 성질(ㄹ 설측, ㅂ 평음 폐쇄 길이 등)은 음성학 문헌의 조음 기술로 **수정** ③ 결과는 "참고 자료와 일치하는 **삽화**"이며 **측정이 아니다** — 모든 viseme JSON의 `educational_disclaimer`와 영상 하단 한 줄로 표기 ④ 한국어 화자 MRI가 있으면 참고에 추가하되 역시 측정으로 부르지 않는다.
Codex가 first draft를 만든다: 참고 프레임을 지정된 좌표계로 **트레이스 유도**(윤곽 추출 스크립트 + 사람 조정 없이 128점 재표본) → PNG 승인용 그림. 사람은 승인/거절·수정 지시만.

**VALIDATED 체크리스트**(전부 ✓여야 승격):
1. 128점·순서·혀끝 인덱스 규격 ✓ 2. 자기교차 0 ✓ 3. 구개 관통 0 ✓ 4. 접촉 viseme: 혀끝–치조 ≤0.1mm, 비접촉 viseme: ≥1.0mm ✓ 5. 분절 높이가 음소 기술과 일치(예: ㄱ dorsum 최고, ㅣ 앞·높음, ㅏ 낮음) ✓ 6. 참고 출처 2개 이상 기록 ✓ 7. 사람 시각 승인("혀로 보인다, 과장·왜곡 없다") ✓ 8. 음성학 판단자(사용자 또는 자문) 조음 위치·방식 승인 ✓ 9. 면적이 REST 대비 ±15% 안(참고치, 위반 시 사유 기록) ✓.

━━━━━━━━━━━━━━━━━━━━
## [5. CURRENT GP 5 KEYS]
━━━━━━━━━━━━━━━━━━━━
결정: **B(재검증 후 승격) — K1·K2·K3·K5. K4는 viseme로 승격하지 않음(D).**
- 공통 기준: 15_GP 128점이 §3 규격(점 수·순서·혀끝 0번·좌표계)이면 정점으로 변환, 아니면 재표본. §4 체크리스트 1~9 통과 시 `VALIDATED`, 5·7·8 중 하나라도 실패면 Codex가 참고 프레임 기준으로 수정 초안을 다시 만들고 사람 승인.
- K1 ㄱ → `KO_k_velar` 후보. 기준: dorsum(32~79) 최고점이 연구개 선에 ≤0.5mm, 혀끝은 낮음.
- K2 ㅏ → `KO_a_open` 후보. 기준: 턱 각도 ≥10°, 혀 전체 낮고 평평, 구개와 최소 거리 ≥8mm.
- K3 ㄹ → `KO_l_lateral` 후보. 기준: 혀끝–치조 ≤0.1mm, dorsum이 ㅏ보다 낮거나 같음(설측 해제의 단면 표현), 혀 아랫면이 오목하지 않음(갈고리 금지).
- K5 ㅣ → `KO_i_front` 후보. 기준: 혀 앞부분이 경구개 앞쪽에 2~4mm 접근(접촉 아님), 입술 옆으로.
- **K4 "ㅂ phase tongue"**: ㅂ의 핵심은 입술이고 혀는 음소 고유 모양이 없다. 따라서 **혀 viseme 없음** — `KO_p_bilabial`의 tongue = REST 포인트 복사 + dominance 0.2(앞뒤 음소가 혀를 지배). K4 파일은 `ARCHIVE`(전이 샘플 기록). 입술 viseme는 혀와 **분리된 오브젝트·셰이프키**이므로 자연히 분리된다.

━━━━━━━━━━━━━━━━━━━━
## [6. AUDIO / TRANSCRIPTION PIPELINE]
━━━━━━━━━━━━━━━━━━━━
순서(단어 1개): ① `raw/` 원본 복사(읽기 전용) → ② ffmpeg: 16kHz mono PCM16, EBU R128 −23 LUFS 정규화 → ③ 토큰 크롭: 단어 앞뒤 150ms 여유, Praat 또는 자동(에너지 기준) 후 사람 확인 → ④ 전사: TARGET은 사전(한국어 표준 발음 → 음소 규칙), ACTUAL은 **사람이 듣고 적음**(학습자 필수, 원어민은 TARGET 복사 후 확인), L1_LEXICAL은 그 언어 사전형 → ⑤ 정렬(§7) → ⑥ 검증: 경계 규칙(§7) 플래그 난 토큰만 Praat에서 사람이 수정.
도구 확정: ffmpeg(정규화·크롭), Praat(사람 확인·수정), MFA(정렬), Whisper는 **쓰지 않음**(음소 전사가 아니라 철자 전사이고 학습자 오류를 "정답"으로 교정해 버림).
MFA 신뢰도 판단: 한국어 원어민 — 높음(korean_mfa 모델). 학습자 — **ACTUAL 음소열로 정렬**하면 타이밍은 쓸 만하지만 비한국어 음(ɾ, ɯ 등)은 모델에 없으므로 **가장 가까운 한국어 음으로 매핑한 정렬용 음소열**을 따로 만든다(`ALIGN_PHONES`). 표시·viseme 조회는 ACTUAL을 쓴다. 즉 네 번째 열 `ALIGN_PHONES`는 정렬 전용이며 전사 JSON에 함께 기록한다.

━━━━━━━━━━━━━━━━━━━━
## [7. FORCED ALIGNMENT]
━━━━━━━━━━━━━━━━━━━━
설치: `conda create -n mfa -c conda-forge montreal-forced-aligner` (MFA 3.x). 모델: `mfa model download acoustic korean_mfa`, `mfa model download dictionary korean_mfa`. 사전 확장: `SYSTEM/ALIGNMENT/custom_dictionary.txt`에 토큰별 항목(`갈비  k a l b i` 형식은 korean_mfa 사전의 음소 기호 체계를 그대로 따를 것 — Codex가 사전 파일에서 기호를 확인해 기록).
실행: 토큰 1개도 코퍼스 형식(`<token_id>.wav` + 같은 이름 `.txt`)으로 `mfa align <corpus> custom_dictionary.txt korean_mfa <out> --beam 40 --retry_beam 400`. 출력 TextGrid → `praatio`로 파싱 → events.json.
학습자: ALIGN_PHONES로 정렬 후, 구간을 ACTUAL 음소에 1:1 재라벨(길이 같을 때). 길이가 다르면(삽입 모음 등) ALIGN_PHONES에 삽입을 반영해 길이를 맞춘다.
실패/오정렬 판정 규칙(자동 플래그): 어떤 음소 구간이 <20ms 또는 >400ms, 구간 순서 역전, 무음 라벨이 단어 안에 >60ms, 총 길이가 토큰 길이의 ±10% 밖. 플래그 → Praat에서 사람이 경계 수정 → `confidence:"manual"`로 저장. 정확도를 자동화에 희생하지 않는다: 플래그 0이어도 첫 10토큰은 사람이 전수 확인.

━━━━━━━━━━━━━━━━━━━━
## [8. COARTICULATION ENGINE]
━━━━━━━━━━━━━━━━━━━━
규칙(Codex 구현용, Cohen–Massaro 단순화):
- 각 음소 p의 구간 [s_p, e_p], 각 조음기관 a ∈ {tongue, jaw, lips, velum}에 대해 가중치 함수 w_pa(t): t < s_p − A_pa 에서 0 → s_p 에서 1로 선형 상승 → e_p 까지 1 → e_p + C_pa 에서 0으로 선형 하강. 여기에 dominance D_pa를 곱한다. (A=anticipatory, C=carryover, ms)
- 프레임 t의 조음기관 a 모양 = Σ_p w_pa(t)·V_pa / Σ_p w_pa(t). 합이 0이면 REST. 셰이프키 값 = 정규화된 w.
- 턱은 각도 스칼라를 같은 식으로 혼합. 입술도 동일. 접촉 플래그: 혼합 결과 혀끝–치조 ≤0.1mm.
- 기본 dominance(`dominance_profiles.json`): 자음 접촉(ㄹ·ㄴ·ㄷ): tongue 1.0 / jaw 0.5 / lips 0.2 / velum 0.6. 양순(ㅂ·ㅁ): lips 1.0 / jaw 0.7 / tongue 0.2 / velum 0.6. 연구개(ㄱ): tongue 0.9 / jaw 0.6 / lips 0.2 / velum 0.8. 모음: tongue 0.7 / jaw 0.8 / lips 0.6 / velum 0.5(ㅏ 0.3). 기본 창: A 80ms, C 60ms (입술 40/40).
- 갈비 개념 예: ㄱ→ㅏ: 턱이 ㅏ 시작 60~80ms 전부터 열리기 시작(턱 dominance 높음), 혀 뒤는 ㄱ 구간 끝까지 유지 후 내려감. ㅏ→ㄹ: 혀끝이 ㄹ 시작 80ms 전부터 올라가기 시작(혀 dominance 1.0), 턱은 ㅏ에 붙잡혀 늦게 닫힘. ㄹ→ㅂ: 입술이 ㅂ 시작 40ms 전부터 닫히기 시작하고 혀끝은 ㄹ 끝까지 접촉 유지 후 내려옴(dominance 혀 1.0 → ㅂ 혀 0.2라 ㄹ 모양이 ㅂ 초반까지 이월). ㅂ→ㅣ: 입술이 터지며 옆으로, 혀 앞이 ㅣ 시작 80ms 전부터 올라감.
- 작업 순서: **PoC는 혼합 없이 "viseme 스위칭"**(각 음소 구간에 해당 viseme 1.0, 경계 2프레임 선형 램프)으로 먼저 검증(PHASE 6) → 통과 후 PHASE 7에서 혼합을 켜고 같은 입력을 재실행해 A/B.

━━━━━━━━━━━━━━━━━━━━
## [9. BLENDER AUTOMATION]
━━━━━━━━━━━━━━━━━━━━
결정: **외부 파이썬이 데이터를 만들고, bpy 단계는 `blender -b` 헤드리스로 실행**(GUI 없음). 모듈(`SYSTEM/AUTOMATION/`):
| 스크립트 | 실행 | INPUT | OUTPUT |
|---|---|---|---|
| 01_build_puppet.py | blender -b | anatomy_reference.json, visemes/*.json(REST) | puppet_master.blend(§3 오브젝트·셰이프키 Basis) |
| 02_viseme_import.py | blender -b | visemes/*.json | puppet_master.blend에 셰이프키 `V_<id>` 추가·갱신, 승인용 PNG 렌더(viseme당 1장) |
| 03_prepare_audio.py | python | raw wav | tokens/*.wav (정규화·크롭), 토큰 메타 |
| 04_run_alignment.sh | shell(mfa env) | tokens, transcriptions | TextGrid |
| 05_parse_alignment.py | python | TextGrid, transcription json | events.json(프레임 변환, 플래그) |
| 06_coarticulation.py | python | events.json, visemes/*.json, dominance_profiles.json, mode=switch|blend | animation.json(프레임별 셰이프키 값·턱 각도·접촉 플래그) |
| 07_apply_animation.py | blender -b | puppet_master.blend, animation.json, token wav | `<run>.blend`(키프레임 입력, 오디오 로드, 프레임 범위) |
| 08_qa_animation.py | blender -b | `<run>.blend`, qa_rules.json | qa_report.json(§10 AUTO 전 항목), 실패 프레임 목록 |
| 09_render.py | blender -b | `<run>.blend`, style_*.json | PNG 시퀀스 → ffmpeg mp4(오디오 포함) |
| 10_report.py | python | qa_report.json, events.json, 산출물 경로 | qa_report.md(사람 게이트 체크리스트 포함) |
| run_word.py | python | token_id, mode, style | 03→04→05→06→07→08→09→10 일괄, 실패 시 단계명과 함께 STOP |
렌더: EEVEE, 1920×1080, 24fps, 플랫 진단 스타일은 단색+Freestyle 선. 메모리: 전부 CPU 수 분 안.

━━━━━━━━━━━━━━━━━━━━
## [10. SCIENTIFIC / EDUCATIONAL QA]
━━━━━━━━━━━━━━━━━━━━
AUTO(전 프레임, qa_rules.json): 위상(128·24·16점 불변) PASS / 자기교차 0 / 구개 관통 0(혀 윗면 정점이 PALATE_LINE 위로 >0.05mm 금지) / 접촉 목표: 접촉 음소 구간 중앙 ±1프레임에서 혀끝–치조 ≤0.1mm, 비접촉 구간 ≥1.0mm / 입술 폐쇄: 양순 음소 구간에서 간격 ≤0.1mm / 사건 타이밍: 접촉·폐쇄 시작이 정렬 경계 ±1프레임 / 시간 스파이크: 인접 프레임 정점 이동 ≤3mm / 결정론: 같은 입력 2회 실행 → animation.json·PNG 해시 동일.
HUMAN(게이트 체크리스트, 각 항목 ✓/✗ + 한 줄): 혀 모양 타당성(고무·갈고리·뾰족 없음) / 조음 위치·방식(음소별) / 시각 가독성(접촉·폐쇄가 한눈에) / 자연스러운 동작(슬라이드쇼 아님) / 교육적 정직성(과장이 오해를 만들지 않음).
PASS 기준: AUTO 전부 PASS + HUMAN 5항목 ✓.
**절대 "scientifically exact"라 부르지 않을 것:** 혀 표면 곡률·두께, 접촉 면적, mm 궤적, 혀 옆면(설측), 연구개 정도, 인두 폭. 부를 수 있는 것: "cited references와 일치하는 조음 위치·방식", "실제 녹음에서 정렬한 타이밍".

━━━━━━━━━━━━━━━━━━━━
## [11. GALBI FIRST SYSTEM TEST]
━━━━━━━━━━━━━━━━━━━━
입력: 오디오 = 사용자 제공 한국어 원어민 녹음 중 **첫 번째 파일의 첫 갈비 토큰**(`galbi_KO_NAT_01_t1`). 화자 `KO_NAT_01`(native, L1 ko). 전사: TARGET = ACTUAL = [KO_k, KO_a, KO_l, KO_b, KO_i], L1_LEXICAL = null. viseme: KO_k_velar, KO_a_open, KO_l_lateral, KO_p_bilabial, KO_i_front (+REST 앞뒤).
처리: 03→04→05(플래그 확인)→06(mode=switch)→07→08→09(style_flat)→10.
출력(`PILOTS/GALBI/runs/<run_id>/`): `galbi_KO_NAT_01_t1.TextGrid`, `events.json`, `animation.json`, `galbi_KO_NAT_01_t1.blend`, `galbi_KO_NAT_01_t1_flat.mp4`(오디오 포함), `qa_report.json`, `qa_report.md`, 실행 로그.
수동 금지: 키프레임을 사람이 찍는 것, viseme 값 손 조정, 프레임 밀기. 수동 허용: 전사 확인, Praat 경계 수정(플래그 난 것), viseme 승인.

━━━━━━━━━━━━━━━━━━━━
## [12. GALBI GO / NO-GO GATE]
━━━━━━━━━━━━━━━━━━━━
GO = 전부: ① 사건 타이밍이 정렬 경계 ±1프레임(자동) ② viseme 라이브러리만 사용(animation.json에 손 키 0 — 스크립트 재실행으로 재현됨) ③ AUTO QA 전 항목 PASS ④ HUMAN 5항목 ✓ ⑤ 결정론 해시 동일 ⑥ `run_word.py`가 **다른 토큰 id**에도 코드 수정 없이 끝까지 돈다(같은 화자 두 번째 토큰으로 확인).
NO-GO 처리: 실패 항목의 담당 PHASE로 되돌아가 그 PHASE만 재작업(1회 수정 규칙). 두 번째 실패는 SPEC 개정 요청.
GO 직후 **SECOND WORD TEST 즉시 실행.** 두 번째 단어 조건: 기존 6 viseme 중 ≥4개 사용, 새 음소 ≤1개(예: ㅁ 또는 ㄴ), 2음절, 원어민 녹음 확보 가능, 학습자 코퍼스에 존재. 단어 선정은 사용자.

━━━━━━━━━━━━━━━━━━━━
## [13. L1 EXPANSION]
━━━━━━━━━━━━━━━━━━━━
퍼펫·엔진·스크립트는 그대로. 추가되는 것은 **viseme와 전사뿐**: 일본어 L1 → `JA_r_tap`(혀끝 짧은 접촉, 설측 없음, 홀드 짧음은 정렬이 자동 반영), 삽입 모음 `JA_u`(ɯ: 입술 비원순 고모음), 필요 시 `JA_a`. 중국어 L1 → 권설/치경 변이, 모음 대체. 영어 L1 → `EN_l_dark`(연구개화 ㄹ: dorsum 올림), 유기음 파열 길이(정렬).
어떤 viseme를 쓰는지는 **ACTUAL 전사**가 결정한다(사람이 들은 음소). 데이터 모델에서 TARGET / L1_LEXICAL / ACTUAL 세 열은 **절대 서로 복사·대체하지 않는다**; 영상의 비교 구간은 ACTUAL끼리(한국어 원어민 ACTUAL vs 학습자 ACTUAL) 또는 TARGET vs L1_LEXICAL(차용형 비교)로 **라벨을 명시**해 렌더한다.

━━━━━━━━━━━━━━━━━━━━
## [14. FINAL CUTAWAY VISUAL]
━━━━━━━━━━━━━━━━━━━━
순서 고정: 플랫 진단판(PHASE 6 PASS) → 비주얼 디자인(PHASE 8) → 최종 렌더. 요소: HEAD_SILHOUETTE(최종에는 배우 실루엣으로 교체 가능) / SKULL_PALATE(연한 뼈색, 단면 해칭 없음) / 치아(상아색) / TONGUE(살구-분홍, 윗면 하이라이트 1겹, 외곽선 1.5px) / 입술 / JAW / AIRWAY(밝은 포인트색, 접촉 시 끊김이 보이게) / 음영: 키라이트 방향 1개(좌상 45° 기본) / 팔레트: style_final.json(배경 1·피부 1·그림자 1·포인트 1) / 그레인·비네트 / CONTACT_HIGHLIGHT: 접촉 프레임에만 작은 링 2프레임. 글자: 음소 1개만(예 "ㄹ"). 식당·앞뒤 영상은 PHASE 8 PASS 전 금지.

━━━━━━━━━━━━━━━━━━━━
## [15. LIVE ACTION / STORY — LATER]
━━━━━━━━━━━━━━━━━━━━
단면 확정 후: 폰 카메라 촬영(식당 와이드 → 대사 → 얼굴 → **옆모습 입 클로즈업 정지 프레임 필수**) → 그 옆모습 실루엣을 트레이스해 HEAD_SILHOUETTE 교체 → 색 보정에서 팔레트 추출 → 단면 재렌더 → Resolve에서 wipe(0.3초, 소리 연속) → 단면 → 비교 → 역 wipe. 3D PREVIZ를 안 쓰는 이유: 15초·컷 5개·한 장소라 공간 계획이 필요 없고, 전환은 2D 실루엣 공유로 해결되며, 8GB에서 추가 도구가 비용만 늘린다. PREVIZ 예외 조건: 카메라가 실제로 머리 안으로 "들어가는" 3D 연속 이동을 한 컷으로 찍어야 할 때, 또는 장소가 3곳 이상·배우 2명 이상 동선이 겹칠 때 — 그때만 Blender 회색 블로킹을 쓴다.

━━━━━━━━━━━━━━━━━━━━
## [16. TOOLCHAIN — EXACT]
━━━━━━━━━━━━━━━━━━━━
| 도구 | 용도 | 필수/선택 | M2 8GB |
|---|---|---|---|
| Blender 4.2 LTS 이상(5.2.2 OK) | 메시 퍼펫·셰이프키·키프레임·EEVEE 렌더·Freestyle | 필수 | OK(헤드리스) |
| Python 3.11 + numpy, praatio, Pillow | 정렬 파싱·혼합·QA·보고 | 필수 | OK |
| Montreal Forced Aligner 3.x (conda) | 강제 정렬 | 필수 | OK(한국어 모델 ~수백 MB) |
| ffmpeg | 정규화·크롭·mp4 | 필수 | OK |
| Praat | 경계 확인·수정 | 필수(사람) | OK |
| Seeing Speech, USC-TIMIT | viseme 형태 참고 | 필수(참고) | 웹 |
| DaVinci Resolve(무료) | 최종 편집·색·wipe | 필수(PHASE 10) | OK |
| 폰 카메라 | 실사 | 필수(PHASE 10) | — |
| 생성형 영상(Veo/Kling 류) | 대사 없는 배경 컷만 | 선택·승인 | — |
| After Effects, 언리얼, 물리 엔진, Whisper | — | 사용 안 함 | — |

━━━━━━━━━━━━━━━━━━━━
## [17. WHAT CODEX SHOULD DO VS HUMAN]
━━━━━━━━━━━━━━━━━━━━
Codex: 폴더 생성·매니페스트·자산 복사(원본 보존) / 모든 스크립트 작성·실행 / 퍼펫 오브젝트 생성 / viseme **first draft**(참고 프레임 트레이스 유도·재표본·PNG) / 사전·정렬 실행·플래그 / 키프레임 자동 입력 / QA·렌더·보고서 / 결정론 테스트 / PHASE_LOG 기록.
사람(사용자): viseme 승인·거절(그림 보고) / ACTUAL 전사(학습자) / 플래그 난 경계 Praat 수정 / HUMAN QA 5항목 / 두 번째 단어 선정 / 최종 스타일 승인 / 유료·외부 승인.
**사람이 반드시 직접 해야 하는 것(정직하게):** ① 학습자 ACTUAL 전사(귀) ② viseme의 조음 위치·방식 판정(음성학) ③ "자연스러운가"(눈). 이 셋은 자동화하지 않는다. 사용자가 Blender에서 직접 그리는 일은 기본 워크플로에 **없다**(거절 시 Codex가 수정 초안을 다시 만듦).

━━━━━━━━━━━━━━━━━━━━
## [18. IMPLEMENTATION PHASES]
━━━━━━━━━━━━━━━━━━━━
공통: FILES NOT TO TOUCH = 기존 GALBI_PILOT/*, FLOW_TESTS/*, HYBRID_ARTICULATION/*, DATA/AUDIO/raw/* (읽기만). 공통 STOP = 메모리 압력 빨강, 유료 필요, 원본 수정 필요, 사람 판단 필요. 공통 RECOVERY = 복구 가능한 오류(모듈 없음·경로·API 이름 차이)는 진단→수정→재시도→계속, 로그에 기록. 모델 권장: 실행·스크립트 PHASE = Codex 기본 코딩 모델·medium, viseme 초안·QA 해석 PHASE = 이미지 입력 가능 모델·medium, 설계 변경이 필요하면 high가 아니라 **이 SPEC 개정 요청**.

**PHASE 0 — 프로젝트 재구성·감사(읽기 전용 감사 후 생성만)**
PURPOSE 새 트리·매니페스트·자산 지위 확정 / INPUT 기존 루트 / TASKS §1 트리 생성, 기존 전 폴더 목록·크기·수정일 감사, ASSET_MANIFEST.md 작성(지위 표), 정적 기하·QA 코드·15_GP·오디오 **복사**, anatomy_reference.json 초안(단위·좌표계 변환 기록), PHASE_LOG.md 시작 / CREATE SYSTEM/, DATA/, PILOTS/, DOCS 3파일 / REUSE 정적 기하, QA 코드, 15_GP, 오디오 / TOOLS python, shell / OUTPUT 매니페스트·트리·복사본 / AUTO QA 원본 해시 = 복사본 해시, 원본 수정 0(mtime 불변) / HUMAN GATE 매니페스트 지위 표 승인 / PASS 트리 존재·해시 일치·원본 불변 / FAIL 원본 변경 감지 → 즉시 STOP·보고 / COMPLEXITY 낮음 / MODEL 기본·medium.

**PHASE 1 — 퍼펫 마스터(메시·셰이프키 Basis)**
PURPOSE §3 오브젝트 전부를 가진 puppet_master.blend / INPUT anatomy_reference.json, REST 혀·입술 초안(15_GP K2 또는 F1 윤곽을 REST 비율 참고로, 128점 재표본) / TASKS 01_build_puppet.py 작성·실행, 직교 카메라, Freestyle, 플랫 재질, 정점 수·순서 검사 / CREATE puppet_master.blend, 01 스크립트, puppet_check.json / REUSE anatomy_reference / OUTPUT 정지 렌더 1장(REST) / AUTO QA 128·24·16점, 자기교차 0, 턱 피벗 위치 = 참조값, 혀끝 인덱스 0 / HUMAN GATE "단면 인형으로 보인다"(비율·배치) / PASS AUTO 전부 + 사람 ✓ / FAIL 비율 이상 → anatomy_reference 수정 1회 / COMPLEXITY 중 / MODEL 기본·medium.

**PHASE 2 — 최소 viseme 라이브러리(REST + 5)**
PURPOSE §4 FIRST SET VALIDATED / INPUT 15_GP K1·K2·K3·K5, 참고 프레임(Seeing Speech /k/ /a/ /l/ /p/ /i/ 캡처 — 사용자가 저장해 `SYSTEM/PHONEME_LIBRARY/references/`에 둠), phonemes.json / TASKS K 변환·재표본, 참고 트레이스 유도 초안, viseme JSON 6개, 02_viseme_import.py로 셰이프키 등록, 승인용 PNG(참고 프레임 옆에 나란히), 체크리스트 자동 항목 계산, VALIDATION_LOG / CREATE visemes/*.json·png, 02 스크립트, dominance_profiles.json 초안 / REUSE 15_GP / OUTPUT 승인 시트 6장 / AUTO QA 체크리스트 1·2·3·4·9 / HUMAN GATE 체크리스트 5·7·8(viseme마다) / PASS 6개 VALIDATED / FAIL 거절된 viseme만 초안 재작성 1회, 두 번째 거절은 사용자가 수정 방향을 글로 지시 / COMPLEXITY 중상(가장 중요) / MODEL 이미지 입력 모델·medium.

**PHASE 3 — 오디오·전사·정렬**
PURPOSE 갈비 원어민 토큰 1개의 TextGrid·events.json / INPUT raw 녹음, 전사 JSON(사용자 확인) / TASKS MFA 설치·모델, 03·04·05 스크립트, 플래그 규칙, Praat 확인 안내 / CREATE tokens/*.wav, TRANSCRIPTIONS/*.json, ALIGNMENTS/*, custom_dictionary.txt / OUTPUT events.json + 경계 표 / AUTO QA §7 플래그 0, 총 길이 ±10% / HUMAN GATE 경계 5개를 Praat에서 눈·귀로 확인 ✓ / PASS 플래그 0 또는 수동 수정 완료 / FAIL MFA 실패 → beam 조정 1회 → 수동 경계 / COMPLEXITY 중(설치가 변수) / MODEL 기본·medium.

**PHASE 4 — 자동 키 배치(스위칭)·적용·QA**
PURPOSE events.json → animation.json(switch) → .blend 키프레임 → AUTO QA / INPUT PHASE 1·2·3 산출 / TASKS 06(mode=switch)·07·08·10·run_word.py, 결정론 테스트(2회 실행 해시) / CREATE 해당 스크립트, PILOTS/GALBI/runs/<run>/ / OUTPUT qa_report.json·md / AUTO QA §10 AUTO 전부 / HUMAN GATE 없음(수치 단계) / PASS AUTO 전부 PASS + 해시 동일 / FAIL 실패 항목별: 접촉 오차 → viseme 혀끝 좌표 확인(PHASE 2로), 타이밍 → 05 프레임 변환 확인 / COMPLEXITY 중 / MODEL 기본·medium.

**PHASE 5 — 플랫 렌더·갈비 E2E(스위칭) = FIRST SYSTEM TEST**
PURPOSE §11 출력 전체 + §12 GO/NO-GO / INPUT PHASE 4 / TASKS 09(style_flat), mp4(오디오 포함), 두 번째 토큰으로 run_word 무수정 재실행 / OUTPUT flat.mp4 2개, 보고서 / AUTO QA §10 / HUMAN GATE §10 HUMAN 5항목 + §12 ⑥ / PASS GO / FAIL NO-GO 처리(§12) / COMPLEXITY 낮음 / MODEL 기본·medium.

**PHASE 6 — 동시조음 혼합 A/B**
PURPOSE 같은 입력을 mode=blend로 재실행, 스위칭판과 비교 / TASKS 06 blend 구현(§8), 두 mp4 나란히, 접촉 타이밍이 ±1프레임 안에서 유지되는지 / OUTPUT blend.mp4, 비교 보고 / AUTO QA §10(접촉·폐쇄 타이밍이 혼합 후에도 유지) / HUMAN GATE "스위칭보다 자연스럽다" / PASS 사람 선택 blend / FAIL dominance 기본값 1회 조정(표 단위, 수치 튜닝 반복 금지), 그래도 아니면 스위칭을 production 기본으로 확정 / COMPLEXITY 중 / MODEL 기본·medium.

**PHASE 7 — 두 번째 단어 테스트**
PURPOSE 재사용성 증명 / INPUT 사용자 선정 단어 녹음 / TASKS 새 음소 ≤1 viseme 추가(PHASE 2 절차), run_word 무수정 / PASS 코드 수정 0, 새 viseme ≤1, GO 기준 동일 / FAIL 재사용 깨진 지점을 SPEC 개정 / COMPLEXITY 낮음 / MODEL 기본·medium.

**PHASE 8 — 최종 단면 스타일**
PURPOSE §14 style_final / TASKS AIRWAY 갱신 스크립트, 재질·조명·팔레트·그레인·접촉 링, 정지 스타일 프레임 1장 승인 후 전체 렌더 / HUMAN GATE 스타일 프레임 승인 → 영상 승인 / PASS 사용자 "숏폼에 쓰겠다" / FAIL 팔레트·선 규칙 1회 교체 / COMPLEXITY 중 / MODEL 이미지 입력·medium.

**PHASE 9 — 일본어 L1 확장**
PURPOSE §13 / INPUT AI-Hub 일본어 L1 갈비 토큰 1개(ACTUAL 전사 사용자) / TASKS ALIGN_PHONES 매핑, JA_r_tap(+필요 viseme) PHASE 2 절차, run_word, 비교 렌더(라벨 명시) / PASS 새 viseme ≤2, 세 전사열 분리 유지, GO 기준 / COMPLEXITY 중 / MODEL 이미지 입력·medium.

**PHASE 10 — 실사·편집(단면 확정 후)**
§15. 촬영은 사람. Codex는 실루엣 트레이스·팔레트 추출·재렌더·Resolve 프로젝트 준비.

━━━━━━━━━━━━━━━━━━━━
## [19. FIRST CODEX JOB — FULL PROMPT] (PHASE 0 + PHASE 1 전반부까지, 그대로 복사)
━━━━━━━━━━━━━━━━━━━━
```
당신은 KOREAN_PRONUNCIATION_SHORTS 프로젝트의 IMPLEMENTER(Codex)다. LEAD ARCHITECT의 MASTER_IMPLEMENTATION_SPEC v1.0(이 파일과 같은 폴더의 사본)을 Source of Truth로 삼고, 아래 PHASE 0을 실행한 뒤 STOP한다.

PROJECT_ROOT=/Users/yiseungmin/Documents/Codex/KOREAN_PRONUNCIATION_SHORTS

[절대 규칙]
1. 삭제·이동·덮어쓰기 금지. 기존 파일은 읽기만 한다. 재사용 자산은 복사한다(cp -p). 복사 뒤 원본 해시·mtime이 변하지 않았음을 확인한다.
2. 유료 API·생성 크레딧·클라우드·새 계정 사용 금지. 네트워크는 pip/conda 설치와 공개 자료 열람만.
3. 생성형으로 조음·이미지를 만들지 않는다. 커스텀 혀 물리 금지.
4. GUI 자동 조작 금지. Blender는 `-b`로만.
5. 메모리: 단계 시작·끝에 `memory_pressure | tail -1` 기록. 여유 <30%면 STOP.
6. 복구 가능한 오류(모듈 없음, 경로 오타, API 이름 차이, 인코딩)는 사용자에게 묻지 말고 진단→수정→재시도→계속, LOG에 남긴다. 진짜 blocker(원본 수정 필요, 유료 필요, 디스크 <10GB, 사람 판단 필요)에서만 STOP.
7. 하지 않은 일을 했다고 쓰지 않는다. 모든 산출물은 `ls -l`로 존재 확인 후 보고.

[PHASE 0 — 읽기 전용 감사 → 트리 생성 → 매니페스트 → 복사]
A. 읽기 전용 감사: `find "$PROJECT_ROOT" -maxdepth 3 -type d` 와 각 1단계 폴더의 크기(du -sh)·최신 수정일을 `SYSTEM/DOCS/AUDIT_<날짜>.md`에 기록(아직 SYSTEM이 없으면 먼저 B를 수행하되 감사 내용은 그대로 적는다). 다음 자산의 실제 경로를 찾아 적는다: 정적 기하(구개·치조·앞니·턱축 좌표가 든 json/py/blend), F1 neutral tongue contour, 기존 QA 코드(contact/palate/self-intersection/topology/timing), GALBI_PILOT/15_GP_KEYSHAPE_TONGUE의 K1~K5 데이터(파일 형식·점 수·점 순서·좌표계·단위), 사용자 제공 한국어 원어민 녹음, AI-Hub 일본어 L1 갈비 토큰. 못 찾은 것은 "NOT FOUND"로 적는다(추측 금지).
B. 트리 생성: SPEC §1의 SYSTEM/, DATA/, PILOTS/GALBI/ 폴더 전부(mkdir -p). 기존 폴더는 건드리지 않는다.
C. ASSET_MANIFEST.md: SPEC §1 표를 그대로 옮기고, A에서 찾은 실제 경로·파일 수·해시(sha256, 큰 blend는 크기+mtime)를 각 행에 추가. 지위(KEEP/MIGRATE/REFERENCE ONLY/ARCHIVE)는 SPEC 값을 따른다. 판단이 필요한 항목은 "제안: … / 사용자 확인 필요"로 표시.
D. 복사(MIGRATE·KEEP만): 정적 기하 → SYSTEM/PUPPET/_migrated/ ; QA 코드 → SYSTEM/QA/_migrated/ ; 15_GP K1~K5 → SYSTEM/PHONEME_LIBRARY/_seed_from_15GP/ ; 녹음 → DATA/AUDIO/raw/ (파일명에 출처 접두어). 각 복사본 옆에 SOURCE.txt(원본 경로·해시·복사 시각).
E. anatomy_reference.json 초안: 정적 기하에서 치조점·윗앞니 끝·경구개 곡선(가능하면 64점)·턱관절 축·단위를 읽어 SPEC §2 좌표계(원점=윗앞니 끝, +x 앞, +y 위, mm)로 변환해 저장. 변환식과 원본 단위를 "conversion" 필드에 기록. 값을 못 찾으면 null + "NOT FOUND".
F. 15_GP 규격 검사: K1~K5 각각 점 수·닫힘 여부·점 순서(시계/반시계)·혀끝 인덱스·좌표 범위를 `SYSTEM/PHONEME_LIBRARY/_seed_from_15GP/SEED_CHECK.json`에. 128점·혀끝 0번이 아니면 "재표본 필요"로 표시(재표본은 PHASE 2에서).
G. PHASE_LOG.md 생성: PHASE 0 시작·끝 시각, 수행 항목, 메모리 기록, 발견한 문제.
H. 결정론·무결성 확인: 복사본 해시 = 원본 해시 전부 일치, 원본 mtime 불변 → 결과를 보고서에 표로.

[보고 형식 — 파티에 그대로]
[PHASE] 0
[결과] PASS / FAIL / 질문
[생성] 트리 목록(tree -L 2 SYSTEM DATA PILOTS), 생성 파일 수
[복사] 항목별 원본 경로 → 복사 경로, 해시 일치 여부
[NOT FOUND] 못 찾은 자산 목록
[anatomy_reference] 채워진 필드 / null 필드
[15_GP 규격] 점 수·순서·혀끝 인덱스 결과
[메모리] 시작/끝 여유 %
[복구한 오류] 무엇을 어떻게 (없으면 "없음")
[사용자 확인 필요] 매니페스트에서 "사용자 확인 필요"로 표시한 항목
[다음] "PHASE 1 승인 대기"

PHASE 0 보고 후 STOP. PHASE 1(01_build_puppet.py)은 사용자가 "PHASE 1 진행"이라고 말한 뒤에만 시작한다.
```

━━━━━━━━━━━━━━━━━━━━
## [20. MASTER ROADMAP]
━━━━━━━━━━━━━━━━━━━━
현재(설계 확정) → **PHASE 0** 재구성·매니페스트 [끝나면: 새 폴더 트리와 자산 지위표 1장] → **PHASE 1** 퍼펫 마스터 [단면 인형 정지 그림 1장] → **PHASE 2** viseme 6개 승인 [참고 프레임 옆에 놓인 승인 시트 6장] → **PHASE 3** 녹음 정렬 [갈비 음소 경계 표 + TextGrid] → **PHASE 4** 자동 키·QA [수치 QA 전부 PASS 보고서] → **PHASE 5 갈비 시스템 성공** [실제 녹음에 자동으로 움직이는 갈비 플랫 영상 3초 + 두 번째 토큰 영상] → **PHASE 6** 동시조음 [스위칭 vs 혼합 나란히 영상] → **PHASE 7** 두 번째 단어 [코드 무수정으로 나온 두 번째 단어 영상] → **PHASE 8 단면 최종 스타일** [최종 룩 갈비 단면 영상] → **PHASE 9** 일본어 L1 [한·일 비교 단면 영상(ACTUAL 라벨)] → **PHASE 10** 실사·편집 [식당→옆모습→wipe→단면→복귀 15초 완성본] → 시리즈 production [단어당: 녹음·전사·승인 3회 클릭 → 영상].

여기서 STOP. 구현 없음. 첫 실행은 §19 프롬프트를 Codex에 그대로 붙여 넣는 것.
