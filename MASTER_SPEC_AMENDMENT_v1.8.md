# MASTER_IMPLEMENTATION_SPEC — AMENDMENT v1.8 (2026-10-06)
기준: v1.0 @ 74b5a011(불변) + v1.1~v1.7 + ERRATUM 1·2 + PHASE 1.5 RELEASE(동결). **v1.8은 PHASE 2 이후의 동적 viseme QA를 정의하고, 표시 규칙 하나(BODY_TISSUE 위 경계)를 바꾼다. PHASE 1.5 규칙은 건드리지 않는다.** 근거: review/PHASE2_DYNAMIC_QA_DECISION_2026-10-06.md.

## H1. §10 — PHASE 2+ QA 3층 구조
- S STRUCTURAL(hard, 공통): 128/24/16점, 시계 방향, v0=TIP, v64=ROOT_TURN, 호별 재표본 검사, 자기교차 0, 같은 화자·배율·원점, 뒤집힘 없음(각 x에서 upper y ≥ lower y).
- A ARTICULATORY(hard, 음소별): H3 표의 정의 관계만.
- D DISPLAY SAFETY(렌더 검사): 렌더 마스크에서 구강 영역 안 혀 LOWER 호 아래 배경 픽셀 0.
- P PLAUSIBILITY(보고, 상한만 FAIL): ROOT 54–75 REST 대비 같은 인덱스 변위 중앙값 ≤25mm; 면적·p90 등은 기록만.
- PHASE 1.5의 ROOT STABILITY·FLOOR·BASE COVERAGE 게이트는 **REST_PROXY 전용**이며 PHASE 2+에 적용하지 않는다.

## H2. §10 — 접촉·관통 규칙(접촉 음소)
- 접촉 집합: 치조 음소(ㄹ·ㄴ·ㄷ·ㅌ·ㅅ 등) S_l = {0} ∪ 104–127 ∪ 1–10; 연구개 음소(ㄱ·ㅋ·ㅇ) S_k = 76–103; 경구개 음소(ㅈ·ㅊ·ㅣ 접근) S_j = 90–117.
- 접촉 PASS: `min dist(접촉 집합, 대상 구간) ≤ 1.0mm + τ` 이고 최근접 대상점이 그 구간 안. 대상 구간: 치조 = PALATE_LINE에서 ALVEOLAR_POINT ±8mm 호 길이; 연구개 = VELUM_LINE ∪ PALATE_LINE 뒤 16점; 경구개 = PALATE_LINE 앞쪽 절반.
- 겹침 ≤1.5mm + τ(접촉 집합 안) = 접촉으로 해석, 렌더 시 선에 스냅. 관통 FAIL = 겹침 >1.5mm + τ 또는 접촉 집합 밖 정점의 구개 초과.
- vertex 0 단독 접촉 조건 폐기.

## H3. §4 — FIRST SET 정의 관계(A)
| viseme | A 조건 |
|---|---|
| REST | PHASE 1.5 동결 규칙 |
| KO_k_velar | S_k 접촉 PASS; S_l→치조 ≥3mm |
| KO_a_open | DORSUM(76–103)→PALATE_LINE ≥8mm; 턱 ≥ REST+6°; 입술 간격 ≥8mm |
| KO_l_lateral | S_l 접촉 PASS; DORSUM→PALATE ≥4mm; notes에 "lateral release not measurable midsagittally" |
| KO_p_bilabial | 입술 간격 ≤0.1mm(스냅); 혀 조건 없음 |
| KO_i_front | DORSUM_front(90–103)→PALATE 앞쪽 절반 1.5~5mm; 입술 간격 ≤6mm; 턱 ≤ REST+4° |
- 새 음소 추가 시 같은 형식으로 정의 관계 1~2개를 먼저 적고 viseme를 만든다(관계 없는 viseme 금지).

## H4. §3 — BODY_TISSUE_DISPLAY 위 경계 동적화, BASE_FILL 폐기
- BODY_TISSUE_DISPLAY 위 경계 = **현재 프레임 TONGUE LOWER 호(1–63)를 0.5mm 아래로 오프셋한 곡선**(DISPLAY, 애니메이션에서 유도) → F_end→P 이하 체인은 ERRATUM 1·2 그대로. 매 프레임 갱신(07_apply_animation.py에서 정점 키프레임 또는 드라이버).
- TONGUE_BASE_FILL: deprecated. 오브젝트 보존, 렌더·QA 모두 미사용. BASE COVERAGE 검사 삭제(D 검사로 대체).
- 렌더 순서·카메라 프레임·occluder 규칙(ERRATUM 2)은 유지.

## H5. §18 — PHASE 2 PASS
- viseme VALIDATED = S + A + D + P(상한) + HUMAN(조음 위치·방식 ✓, 혀로 보임 ✓, 소스 MRI 오버레이와 같은 관계 ✓). FIRST SET 6개 VALIDATED = PHASE 2 PASS. 재트레이스·기하 변형 없이 기존 best candidate로 재판정.
