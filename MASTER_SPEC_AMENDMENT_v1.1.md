# MASTER_IMPLEMENTATION_SPEC — AMENDMENT v1.1 (2026-10-05)
기준 문서: MASTER_IMPLEMENTATION_SPEC.md @ 74b5a011 (변경하지 않음). 이 파일이 우선 적용되는 항목만 적는다. 근거: review/PHASE2_CLAUDE_VERDICT_2026-10-05.md.

## A1. §2 CORE DATA MODEL — 추가
- viseme JSON에 `"tract_reference":{"speaker":"USC_pk_2015","frame_id":"<캡처 파일명>","scale_mm_per_px":<float>,"note":"SHAPE_REFERENCE_ONLY"}` 필드 추가.
- 혀 인덱스 분절은 **REST에서 호 길이 비율로 고정**: tip 0–15, blade 16–31, dorsum 32–79, root 80–127(혀끝 = 0, 시계 방향). 모든 viseme는 같은 인덱스 의미를 가진다(재표본 시 혀끝·뿌리 앵커로 정렬).
- anatomy_reference.json에 `"jaw_opening_sign":"+Z"`, `"legacy_conversion":{"legacy_sign":"-Z","rule":"deg_canonical = -deg_legacy"}` 기록.

## A2. §3 SAGITTAL PUPPET MASTER — 교체
- SKULL_PALATE·UPPER_TEETH·LOWER_TEETH·VELUM·PHARYNX_WALL·HEAD_SILHOUETTE는 **참조 화자 1명(USC rtMRI IPA chart, pk_2015)의 휴지 프레임 1장을 트레이스**해 만든다. 플레이스홀더 금지.
- 추가 오브젝트: `PALATE_LINE`(경구개 하단 곡선 64점, 치조 융기 포함, static), `VELUM_LINE`(연구개 하단 16점, V_open 셰이프키), `FLOOR_LINE`(구강저/아래턱 내면 32점, JAW_PIVOT 자식), `HYOID_ANCHOR`(Empty, 혀뿌리 부착점, ROOT 자식).
- 턱 부호: **+Z = 열림** 확정. JAW_PIVOT = 참조 프레임의 하악 과두 위치.
- 배율: 윗앞니 끝–연구개 끝 거리를 anatomy_reference 값(없으면 50mm)으로 정규화. 원점·축은 §2.

## A3. §4 PHONEME VISEME LIBRARY — 구성 규칙 교체
- viseme 혀 윤곽은 **참조 성도 안에서** 만든다: 같은 화자·같은 배율의 해당 음소 MRI 프레임에서 혀 윤곽을 트레이스 → 128점 재표본(혀끝 0) → 아랫면은 FLOOR_LINE에, 뿌리는 HYOID_ANCHOR에 **부착**(거리 ≤1mm). 빈 공간에 떠 있는 자유 덩어리 금지.
- 한국어 특이점(ㄹ 설측: 혀 몸 낮춤, ㅂ: REST 혀)은 텍스트 참고로 소폭 조정만 하고 조정 내용을 `validation.notes`에 기록.
- 체크리스트 추가: 10) **혀가 구강을 채운다** — PALATE_LINE과 혀 윗면 사이 기도 단면적이 REST에서 참조 프레임 측정치의 ±30% 안 11) 아랫면·뿌리 부착 거리 ≤1mm 12) 승인 시트는 "참조 MRI 프레임 + 반투명 트레이스 오버레이 | 퍼펫 렌더" 2열.
- 면적 ±15%는 참고치 유지, **자연스러움·관계(구개·바닥·뿌리) 우선**.

## A4. §5 CURRENT GP 5 KEYS — 결정 변경
- B → **ARCHIVE(전부)**. 15_GP K1~K5는 위상 규격(128점) 참고로만 쓰고 형태는 쓰지 않는다. `_seed_from_15GP/`는 보존, 매니페스트 지위 ARCHIVE.

## A5. §10 QA — 추가
- AUTO: 기도 단면적 범위(REST 대비 음소별 허용 범위는 참조 프레임 측정치로 설정), 아랫면·뿌리 부착 거리 ≤1mm, 턱 부호 +Z.
- "scientifically exact" 금지 목록에 "참조 화자의 성도 형태 = 한국어 화자" 추가.

## A6. §18 — PHASE 1.5 삽입(PHASE 1 뒤, PHASE 2 앞)
**PHASE 1.5 — REFERENCE TRACT MASTER**
PURPOSE 참조 화자 1명의 휴지 프레임에서 성도 전체를 트레이스해 anatomy 교체 + REST 재건 / INPUT USC rtMRI IPA chart(pk_2015) 휴지 프레임 캡처(사용자가 저장해 `SYSTEM/PHONEME_LIBRARY/references/pk_2015/`), anatomy_reference.json / TASKS 윤곽 추출 스크립트(이미지 → 폴리라인 초안, Codex), 배율·원점 정규화, PALATE_LINE·VELUM_LINE·FLOOR_LINE·HYOID_ANCHOR·치아·입술·인두·실루엣 생성, REST 혀 128점(부착 조건), 01_build_puppet.py 갱신, 턱 +Z 변환 / CREATE anatomy_tract.json, puppet_master.blend(갱신), REST.json/png, 오버레이 승인 시트 2장(성도·REST) / REUSE PHASE 1 오브젝트 구조·셰이프키 메커니즘 / NOT TO TOUCH 15_GP, 기존 6 viseme(ARCHIVE로 이동 표시만) / TOOLS python(Pillow, numpy), blender -b / OUTPUT 승인 시트 2장 / AUTO QA 점 수·자기교차 0·부착 거리·기도 면적 참조 대비 ±30%·배율 기록 / HUMAN GATE "성도가 MRI와 같은 구조로 보인다", "REST 혀가 구강을 채우고 붙어 있다" / PASS 둘 다 ✓ / FAIL 트레이스 초안 1회 재작성(윤곽 추출 파라미터), 두 번째는 사용자가 오버레이에서 수정 지시 / STOP 성도+REST 승인 전 viseme 5개 착수 금지 / COMPLEXITY 중 / MODEL 이미지 입력·medium.
PHASE 2는 1.5 PASS 후 A3 규칙으로 **재시작**(기존 6 draft는 ARCHIVE).

## A7. §19 — 다음 Codex 작업 프롬프트는 PHASE 1.5 (요약; 전문은 LEAD가 승인 후 작성)
읽기 전용 → references/pk_2015 캡처 확인(없으면 사용자에게 캡처 요청 후 STOP) → 윤곽 추출 초안 → 정규화 → 오브젝트 생성 → REST → 오버레이 시트 2장 → §11 형식 보고 → STOP.
