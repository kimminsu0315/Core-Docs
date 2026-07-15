# Core 시스템 문서 페이지 변경 이력

[d42] 2026-07-15 · 식별자 명명 규칙 전면 도입 + Core 공정 관리 제거 (개인용·push)
- 본체 — SRS d104 (§1.2 식별자 명명 규칙 신설: 통합WIP=WIP-{CLN|DP|CNC}{D3}·CNC작업대=CNC-{SM MACHINE_NAME}-{BEFORE|AFTER}·AMMR=AMMR-{LOGI|PROC}-{D3}·슬롯 ID화 · 되담기 WIP→CNC WIP · Core 공정 관리 제거: Recipe는 GM 소유·투입 단위 Unit 부속·Core는 조회만) · SAD d154 (개명·CNC 작업대 슬롯 스킴·공정 관리 파급) · RnR d34
- 외부전달 — ICD_AMMR d33 (ammr-001→AMMR-LOGI-001·slot_index→slot_id 풀 ID·node 라벨 새 스킴·§5.2 라벨 형식 재작성) · UI정의제안 d37 (표시 라벨 되담기 WIP→CNC WIP·목업 ASCII 폭 정합) · SRS_AMMR d25
- 학습자료 — 처리주체분기·Channels해설 개명 반영
- 커밋 (push 후 기재)

[d41] 2026-07-15 · ack 용어 정리 + ICD 생명순 문서 전체 반영 (개인용·push)
- 본체 — SAD d153 (앱계층 Ack 8곳 우리말화: Job Ack·도킹 Ack→'Job 결과 보고'·'도킹 완료 보고', 단순 Ack→'단순 확인 응답' · ack/ACK는 MQTT 패킷 이름[CONNACK·PINGRESP]에만 예약)
- 외부전달 — ICD_AMMR d31 (§3.3 Topic 표·§3.4 QoS 표를 메시지 카탈로그 생명순으로 재배치[conn=1번] · '도킹 Ack'→'도킹 완료 보고' 2곳 · CONNACK 유지)
- 학습자료 — 어댑터입력처리 d14 ('단순 ACK'→'단순 확인 응답' 3곳)
- 커밋 1aff54e (ede2024..1aff54e)·5파일 · github.com/kimminsu0315/Project_Core main

[d40] 2026-07-15 · 외부전달 ICD 설계 변경 + 메시지 생명순 재번호 동기화 (개인용·push)
- 본체 — SRS d101 (AMMR 주기 일괄 보고에 Slot별 적재 정보 포함·적재 정보 스냅샷 통합 반영 — 이전 push 누락 누적분) · SAD d152 (재동기화 경로를 Core 연결 상태 발신 push 기반으로 교체)
- 외부전달 — ICD_AMMR d30 (job/ack→job/report 개명 · Core 연결 상태 core/conn 신설 · C-5 예비화 · §3.7 수신확인 표 분리 · 메시지 A/C 생명순 재번호[conn=1번]) · UI정의제안 d36 (시스템 연결 판정에 Core 측 연결 상태 반영)
- 커밋 ede2024 (2305baf..ede2024)·6파일·github.com/kimminsu0315/Project_Core main

[d39] 2026-07-14 · UI 활성화 프리스크립션 제거 (개인용·push)
- 외부전달 — UI정의제안 d34 (일괄 보고 주기 필드 활성 시점은 업체 결정 영역 — 활성화 문구 제거)
- 커밋 2305baf (8ae1f2e..2305baf)·3파일

[d38] 2026-07-14 · UI 목업 정정 (개인용·push)
- 외부전달 — UI정의제안 d33 (§9.3 설정 화면 모형에 '일괄 보고 주기' 필드 반영·§6.2 표와 정합)
- 커밋 8ae1f2e (1b0a454..8ae1f2e)·3파일

[d37] 2026-07-14 · 순단 설계 후속 정정 (개인용·push)
- 외부전달 — SRS_AMMR d24 (BMS 주기 5→10초 정정) · ICD_AMMR d27 (CONNECT 설정 절 신설·미명시 옵션 미사용 명시)
- 커밋 1b0a454 (9eeceef..1b0a454)·4파일

[d36] 2026-07-14 · 누적 작업 전체 동기화 (개인용·push)
- 본체 — SRS d100 · SAD d148 · RnR d33 (직전 본문 push d32 이후 누적분 일괄)
- 외부전달 — SRS_AMMR d23 · ICD_AMMR d26 · UI정의제안 d32 (와이파이 음영 순단 대비 통신 설계 반영)
- 학습자료 10건 — 재빌드 (변경분만 시각 갱신)
- 커밋 9eeceef (8743470..9eeceef)·13파일

[d35] 2026-07-12 · 인덱스 접기 버튼 위치 오른쪽 정렬 (개인용·push)
- index.html — "모두 접기/펼치기" 버튼을 오른쪽 끝 정렬로(로컬 뷰어 인덱스와 위치 일치)·배경 흰색 통일
- 커밋 8743470 (0d132ff..8743470)·2파일

[d34] 2026-07-12 · 인덱스 접기 버튼 + 문서 햄버거 재표시 (개인용·push)
- index.html — 폴더 "모두 접기/펼치기" 마스터 버튼 추가 (로컬 뷰어 인덱스와 일치·생성기 build_index)
- head-custom.html — 햄버거 스크롤 숨김에 "멈추면 0.6초 뒤 재표시" 추가 (아래로 스크롤 중엔 숨김 유지·소섹션 도달 시 복귀). 세 렌더면(로컬·번들·깃) 공통 적용
- 커밋 0d132ff (3b93fe3..0d132ff)·3파일

[d33] 2026-07-12 · head-custom v1.6 이식 — 문서 페이지 목차 동작을 로컬 뷰어·번들과 일치 (개인용·push)
- head-custom.html — 로컬 뷰어 v1.6 목차 UX 이식: D1 사이드바 상시(목차없음 표시)·D2 개요접힘+toc-open 진입·D3 단일 개요토글·D4 키보드 T/Esc·D5 스크롤스파이 상위버블·D6 홈경로 VIEWER_HOME 주입·D7 앵커여백. mermaid CDN·셀렉터 유지
- 본체·외부전달·학습자료 — 재빌드 (본문 불변·시각 유지, 자산만 갱신)
- 커밋 3b93fe3 (2b16214..3b93fe3)·3파일

[d32] 2026-07-12 · 누적 작업 전체 동기화 (개인용·push)
- 본체 — SRS d97 · SAD d147 · RnR d32 (직전 push d31 이후 누적분 일괄)
- 외부전달 — SRS_AMMR d21 · ICD_AMMR d21 · UI정의제안 d31
- 학습자료 10건 — 재빌드 (변경분만 시각 갱신)
- 커밋 2b16214 (ea10c00..2b16214)·8파일

[d31] 2026-07-04 · 누적 작업 전체 동기화 (개인용·push)
- 본체 — SRS d81 · SAD d133 · RnR d31 (직전 push 이후 누적분 일괄)
- 외부전달 — SRS_AMMR d14 · ICD_AMMR d15 · UI정의제안 d27
- 학습자료 10건 — 재빌드 (변경분만 시각 갱신)
- 커밋 c8a525c (fa8eb6c..c8a525c)·18파일

[d30] 2026-06-16 · out/ 시각 자동 통일 + index 설명글 괄호 제거 (개인용·push)
- 외부전달 out/ — 시각을 본체와 동일 시각 자동으로 통일 (내용 바뀌면 시각 갱신·NEW) · 업체엔 별도 산출물로 전달하므로 Pages 시각·NEW 는 내부 표시용 · 생성기 out/ 고정 분기 제거 · 매뉴얼 §6.2 · 산출물목록 주석
- index 설명글 — 본체·외부전달 설명글 괄호 제거 (SAD (C4 L1~L3) · UI/ICD (MQTT))
- ICD_AMMR — 시각 2026-06-16 21:31 (담은 변경이 시각 자동 통일로 NEW 표면화) · SAD 2026-06-16 20:30 · SRS_AMMR/UI 2026-05-25 18:01 유지
- 커밋 fa8eb6c (0033208..fa8eb6c)·3파일

[d29] 2026-06-16 · 색채어 sweep 산출물 반영 (개인용·push)
- 본체 — SAD d126 (본문·다이어그램·코드블록 색채어 일상어화: 결/결로→방식·것·축·원칙 / 박힘→정해진다 / 흡수→처리 / 본질→핵심 / 본체(일반명사)→주·기준 · 약 95 occurrence · 식별자·enum·영어 컴포넌트명 보존)
- 외부전달 — ICD_AMMR (박은→담은)
- 나머지(SRS·RnR·out/SRS_AMMR·out/UI·learn) = 색채어 무변경 (빌드 자동B 시각 유지)
- 검사 스코프 — Manifest §5.2 색채어스캔 신설·본체 편입 (d42 본체 제외 폐기), check_vocab·sweep-scan 코드블록 본문 포함
- 커밋 0033208 (363ad6c..0033208)·4파일

[d28] 2026-06-16 · SAD 머리 인용구 마침표 정정 (개인용·push)
- 본체 — SAD d125 (머리 인용구 셋째 줄 "Core 시스템을 C4 Model L1~L3 수준에서 서술한다" → "… 서술한다." · 산출물 마침표 양식 정합·d27 구조 정렬 후속)
- 그 외 무변경 (index 빌드 시각만 갱신)
- 커밋 363ad6c (f7cb9c0..363ad6c)·3파일 17+/8−·github.com/kimminsu0315/Project_Core main

[d27] 2026-06-16 · 산출물 본문 구조 표준 정렬 반영 동기화 (개인용·push)
- 본체 — SRS d74 (섹션 heading H1→H2·하위 H3 재레벨·제목/버전안내/목차 H2 유지·앵커 불변) · SAD d124 (머리 인용구 3줄 신설·§1 개요 → 아키텍처 개요)
- 외부전달 — UI정의제안 d20 (머리 기준 본체 추가·§1 → 태블릿 UI 개요·현재문서 버전줄 제거) · SRS_AMMR d9 (heading 재레벨) · ICD_AMMR d8 (현재문서 버전줄 제거)
- 학습자료 10건 — 재빌드
- 표준 단일 진실 = 작업운영 §8.11 산출물 본문 구조 표준 (heading H2·개요 [무엇]개요·머리 인용구·상단 알림 인라인·버전/업데이트 빌드 자동 주입)
- 커밋 f7cb9c0 (2702b9b..f7cb9c0)·17파일 833+/799−·github.com/kimminsu0315/Project_Core main

[d26] 2026-06-15 · 외부전달(out/) Pages 빌드 보정 (개인용·push)
- out/ 3건(SRS_AMMR·ICD_AMMR·AMMR_Tablet_UI) 헤더 상단 `> **dN 갱신**` 노트 = 빌드서 제거 (소스 유지·빌드만)
- out/ 헤더 `최종 업데이트` = 직전 페이지 싱크 시각 `2026-05-25 18:01` 고정 — 생성기 자동B(바이트 diff→현재 시각) 우회, notation·rebuild 로는 안 바뀜, 실제 본체 내용 sync 때만 산출물목록서 갱신
- index.html 하단 시각 = 빌드/푸시 시각 유지 (out/ 링크 data-updated·link-updated = 18:01 따라감)
- 본체(main)·학습(learn) = 현행 자동 시각·노트 유지
- 생성기 make_doc_body(out 노트 제거)·메인 루프(out 시각 고정) + 산출물목록 out/ updated 18:01 수정

[d25] 2026-06-15 · 표기·어휘 전수 sweep(본체·외부전달 그룹) 반영 전체 동기화 (개인용·push 완료)

- 본체 3건 — SRS d69·SAD d120·RnR d27(담당자 마스킹) → docs/Core_*.md
- 외부전달 3건 — SRS_물류AMMR d8·ICD_AMMR d7·UI정의제안 d18 → docs/out/*.md
- 학습자료 10건 — 재빌드 (본문은 아직 sweep 전·다음 단계 큐)
- .gitignore 신설 + 변경기록 슬롯 rename (옛 변경로그 제거)
- 커밋 516fc7e (3c3f94a..516fc7e)·22파일 822+/792−·github.com/kimminsu0315/Project_Core main

[d24] 2026-05-25 · 학습자료 영역 신설 — docs/learn/ 폴더트리 + 학습자료 → docs 동기화 구성 ((B) 직접 편집)

**(B) 학습자료 영역 신설** (학습 허브 index.md 제외, 읽는 순서 번호·난이도 한글 파일명):
- 학습자료/기술참고/기술개념모음 d5       → docs/learn/01_기술개념모음_기초.md
- 학습자료/기술참고/DI와동시성 d8         → docs/learn/02_DI와동시성_기초중급.md
- 학습자료/구현참고/ASPNET_PoC d3         → docs/learn/03_ASPNET_PoC_기초중급.md
- 학습자료/구현참고/Channels해설 d5       → docs/learn/04_Channels해설_중급.md
- 학습자료/구현참고/권위일원화 d3         → docs/learn/05_권위일원화_중급.md
- 학습자료/구현참고/어댑터입력처리 d5     → docs/learn/06_어댑터입력처리_심화.md
- 학습자료/구현참고/처리주체분기 d6       → docs/learn/07_처리주체분기_심화.md
- 학습자료/구현참고/어댑터헬스모니터 d3   → docs/learn/08_어댑터헬스모니터_심화.md
- 학습자료/구현참고/애플리케이션로그 d4   → docs/learn/09_애플리케이션로그_중급운영.md
- 학습자료/운영참고/DB부하측정 d3         → docs/learn/10_DB부하측정_중급운영.md

동반:
- 각 docs 도입부에 기준 본체·최종 업데이트(2026-05-25 23:00) 헤더 추가 (AMMR 양식)
- index.html 학습자료 폴더트리 섹션 신설 (out처럼 <details>, 10링크 + .link-updated·data-updated 2026-05-25T23:00) + 페이지 하단 시간 23:00
- 학습자료 영역 = 개인깃 전용 (회사깃 제외, 사내 비공개) — GitHubPages 매뉴얼 d15 동반
- push 미진행 — 구성만, 사용자 트리거 시 개인용 push

[d23] 2026-05-25 · 전체 파일 동기화 (개인용) + §3.1 마스킹 규칙 변경 반영 ((A) 자동 동기화 + (B) 직접 편집 동반)

**(A) 본체 → docs 동기화 6건** — 본체 본문 복사 + 도입부 시간 갱신:
- 본체/1_SRS/Core_SRS_v2_0_d68.md                → docs/Core_SRS.md
- 본체/2_SAD/Core_SAD_v1_0_d118.md               → docs/Core_SAD.md
- 운영/RnR/Core_RnR_v0_1_d25.md                  → docs/Core_RnR.md
- 외부전달/Core_SRS_물류AMMR업체용_v1_0_d7.md     → docs/out/Core_SRS_AMMR.md
- 외부전달/Core_ICD_AMMR_v0_1_d6.md              → docs/out/Core_ICD_AMMR.md
- 외부전달/Core_UI정의제안_AMMRTablet_v0_1_d17.md → docs/out/AMMR_Tablet_UI.md

**RnR 마스킹** — 새 §3.1 규칙 (GitHubPages 매뉴얼 d13):
- 담당자 칸에 내용 있으면 칸 전체 → "***" (특정 이름 매칭 폐기)
- 대상 = 담당자 4인 + TBD, 데이터 18자리 전부
- 회사용 동기화 = 마스킹 부재 (원본 그대로)

**(B) index.html 직접 편집** — 시간 세 자리 (§5):
- 각 docs 도입부 시간 6건 → 2026-05-25 14:10
- 페이지 하단 시간 → 2026-05-25 14:10
- 각 링크 .link-updated + data-updated 6건 → 2026-05-25 14:10 / 2026-05-25T14:10
- repo 주소 = 이미 개인용 무변경 (https://github.com/kimminsu0315/Project_Core)

개인용 repo push 동반 (GitHub_ProjectDocs)

[d22] 2026-05-24 · index.html 자세 보강 ((B) 직접 편집 자취, 사용자 결정 자세)

**1. L256 폴더명 슬래시 제거**:
- `<span class="folder-name">out/</span>` → `<span class="folder-name">out</span>`
- 옛 자취 = `out/` (자세 슬래시 들어 있어 명자 자세 어색) → 새 자취 = `out` (자세 정합)

**2. out 영역 3 자취에 `data-updated` + `.link-updated` 자세 추가**:

| 자취 | data-updated | link-updated |
|---|---|---|
| L260 (Core_SRS_AMMR) | `2026-05-24T22:55` | `최종 업데이트: 2026-05-24 22:55` |
| L271 (AMMR_Tablet_UI) | `2026-05-24T22:55` | `최종 업데이트: 2026-05-24 22:55` |
| L282 (Core_ICD_AMMR) | `2026-05-24T22:55` | `최종 업데이트: 2026-05-24 22:55` |

**3. 자세 변경 자세 자취**:
- 옛 자세 자세 자세: 외부전달 = `.link-updated` 부재 → NEW 대상 외 (옛 §6 자세)
- 새 자세 자세 자세: 외부전달 = `.link-updated` 자세 자체 → NEW 대상 (새 §6 자세, 본체와 동일)

**4. 동반 매뉴얼 보강**:
- GitHubPages 매뉴얼 §6 본문 보강 — 대상 적용 자리 자세 자세 (d9 → d10)
- 옛: "본체 영역 3건에만 자리 있음" → 새: "본체 + 외부전달 모두 NEW 대상"

[d21] 2026-05-24 · 전체 파일 동기화 — 개인용 + 외부전달 영역 포함 ((A) 자동 동기화 + (B) 직접 편집 동반)

**대상 파일 6건 — 본체 본문 그대로 복사 + 도입부 양식 + 시간 갱신**:

- **docs/Core_SRS.md** — 본체 SRS v2.0_d68 본문 + 도입부 기준 파일명 `Core_SRS_v2_0_d68.md` + 도입부 시간 `2026-05-24 22:55`
- **docs/Core_SAD.md** — 본체 SAD v1.0_d118 본문 + 도입부 기준 파일명 `Core_SAD_v1_0_d118.md` + 도입부 시간 `2026-05-24 22:55`
- **docs/Core_RnR.md** — 본체 RnR v0.1_d25 본문 + **마스킹 적용 (개인용)** — 담당자 실명 6자리 → "*** 프로님" (GitHubPages 매뉴얼 §3.1) + 도입부 기준 파일명 `Core_RnR_v0_1_d25.md` + 도입부 시간 `2026-05-24 22:55`
- **docs/out/Core_SRS_AMMR.md** — 외부전달 SRS_AMMR v1.0_d7 본문 + 도입부 기준 파일명 `Core_SRS_물류AMMR업체용_v1_0_d7.md` + 도입부 시간 `2026-05-24 22:55`
- **docs/out/Core_ICD_AMMR.md** — 외부전달 ICD_AMMR v0.1_d6 본문 + 도입부 기준 파일명 `Core_ICD_AMMR_v0_1_d6.md` + 도입부 시간 `2026-05-24 22:55`
- **docs/out/AMMR_Tablet_UI.md** — 외부전달 UI정의제안 v0.1_d17 본문 + 도입부 기준 파일명 `Core_UI정의제안_AMMRTablet_v0_1_d17.md` + 도입부 시간 `2026-05-24 22:55`

**index.html 갱신**:

- **L210 레포 주소** = `[회사용 repo]` (회사용) → `https://github.com/kimminsu0315/Project_Core` (개인용)
- **L225 `.link-updated` (Core SRS 자리)** = `2026-05-24 22:55`
- **L236 `.link-updated` (Core SAD 자리)** = `2026-05-24 22:55`
- **L247 `.link-updated` (Core RnR 자리)** = `2026-05-24 22:55`
- **L297 페이지 전체 시간** = `2026-05-24 22:55`

**마스킹 워크플로우 첫 적용 자취** — GitHubPages 매뉴얼 d9 §3.1 RnR 개인용 마스킹 신설 직후 첫 동기화에서 자세 박음. 본 사이클 검증 통과.

[d20] 2026-05-22 · 본체 SAD d117 → d118 결식 결로 박힌 결의 docs 동기화 ((A) 자동 동기화 결식, 부수 자리 점검 §6.3 결로 박힌 본체 SAD 보강 4자리 결과 후속)

- **docs/Core_SAD.md** = 본체 SAD d118 본문 결로 갈음 + 도입부 기준 파일명 자리 갱신 (`Core_SAD_v1_0_d117.md` → `Core_SAD_v1_0_d118.md`) + 도입부 시간 결 갱신 (`2026-05-22 00:06` → `2026-05-22 16:57`).
- **index.html L241 `data-updated`** = `2026-05-22T00:06` → `2026-05-22T16:57`.
- **index.html L247 `.link-updated`** (Core SAD 자리) = `2026-05-22 00:06` → `2026-05-22 16:57`.
- **index.html L297 페이지 전체 시간 결** = `2026-05-22 12:43` → `2026-05-22 16:57` (인프라 결 변경 결로 무조건 갱신, 매뉴얼 §11.5 결식).
- Core SRS·Core RnR 결식 결로 박힌 결의 `data-updated`·`.link-updated` 무변경 (본체 SRS·RnR 무변경 결로).
- 외부전달 영역 docs/out/* 결식 결로 박힌 결 미박음 (사용자 결식 = "외부 폴더 제외").

[d19] 2026-05-22 · R&R 파일명 `Core_R&R_*` → `Core_RnR_*` 결로 갈음 ((B) 직접 결식, 부수 자리 점검 §6.2 결, `&` 다운로드 실패 자리 정정)

- index.html L219 결식 결로 `href="./Core_R&R.html"` → `href="./Core_RnR.html"` 결로 갈음 + `data-updated="2026-05-21T18:30"` → `data-updated="2026-05-22T12:43"` 결로 갱신.
- index.html L223 결식 결로 `<span class="link-title">Core_R&R</span>` → `<span class="link-title">Core_RnR</span>` 결로 갈음 (표시 텍스트 결식 결로 파일명 결과 통일).
- index.html L225 결식 결로 `.link-updated` 시간 결 갱신 (`2026-05-21 18:30` → `2026-05-22 12:43`) — 본 R&R 결식 결로 박힘 결.
- index.html L297 페이지 전체 시간 결 갱신 (`2026-05-22 11:32` → `2026-05-22 12:43`) — 인프라 결 변경 결로 매뉴얼 §5 결식 결로 무조건 갱신.
- docs/Core_R&R.md → docs/Core_RnR.md 결로 파일명 갈음 + 도입부 기준 파일명 자리 갈음 (`Core_R&R_v0_1_d23.md` → `Core_RnR_v0_1_d24.md`) + 도입부 시간 결 갱신 (`2026-05-21 18:30` → `2026-05-22 12:43`).
- L224 결식 결로 박힌 `<span class="link-desc">R&R시스템 영역별 개발</span>` 결식 결로 박힌 결은 본문 어휘 결로 책임 분담 의미 결로 자연 유지 — 파일명 결만 갈음.
- 별도 파일 GitHubPages d4 → d5 결식 결로 박음 — 본문 안 R&R 박힌 4자리 (§2 매핑 표 L33, §3 박지 않을 결 L94, §6 NEW 배지 식별자 예시 L155, §7 자취 결식 예시 L188) 결로 RnR 결로 갈음.

[d18] 2026-05-22 · NEW 배지 시각·자리·동작 결식 갈음 ((B) 직접 결식)

- index.html CSS 결식 결로 `.tag-new` 클래스 제거, `.badge-new` 클래스 신설 — 파랑 (`#3B82F6`) + 흰 글자, pill shape 결식 (`border-radius: 10px`, `font-size: 9px`, `padding: 2px 6px`, `letter-spacing: 0.5px`, `text-transform: uppercase`). 일반적 NEW 배지 결식 (작고 둥근 파란 결로 박힘).
- index.html body 끝 JS 결식 갈음 — localStorage 키 결식 `core_docs_last_visit` (단일 ISO 결) → `core_docs_visits` (객체 결식, `{"<href>": "<ISO 시각>", ...}`) 결로 갈음. 식별자 결식 결로 각 `.link-item`의 `href` 속성 결로 박음 (예: `./Core_R&R.html`). 파일별 분리 결식 결로 박음.
- index.html 방문 시각 갱신 시점 결식 갈음 — 페이지 로드 직후 일괄 갱신 → 해당 `.link-item` 클릭 이벤트 결식 결로 박힌 결식 결로 박힌 결식 결로 박음. 클릭 결식 결로 박힐 시 해당 파일의 방문 시각 결만 갱신 결로 박음. 한 파일 결식 결로 방문 결식 결로 박힐 시 다른 파일의 NEW 결식 결로는 갱신 부재 — 사용자 결식 결로 박힌 결식 결로 박힌 결식 ("파일별 설정 되도록").
- index.html NEW 배지 박힘 자리 결식 갈음 — 기존 태그 결식 옆 (Internal/Draft 옆) → `.link-desc` 직후 결로 갈음 (설명 끝, `.link-updated` 앞). 사용자 결식 결로 박힌 결식 결로 박힌 결식 ("줄 가장 마지막에 나왔으면 좋겠어").
- 동반 시간 결 갱신 (매뉴얼 §5 결, 인프라 결 변경 결로 페이지 전체 시간 결 무조건 갱신):
  - index.html 페이지 전체 시간 결 갱신 (`2026-05-22 11:17` → `2026-05-22 11:32`).
  - 본체 결 변경 부재 결로 docs/*.md 도입부·각 링크 `.link-updated` 결식 결로는 시간 결 그대로.
- 별도 파일 GitHubPages d3 → d4 결식 결로 박음 — §6 NEW 배지 결식 결로 본 갈음 자취 결식 박음 (시각·자리·동작 갱신). Pages 변경기록 결식 결로는 매뉴얼 자체 변경 자취 박힘 부재 (§1 결식 결로 docs/ 하위 범위 결로 박힘).

[d17] 2026-05-22 · NEW 배지 결식 박음 (별도 파일 GitHubPages §6 결) + 회사용 레포 결로 갈음 ((B) 직접 결식)

- index.html L196 레포 주소 결식 결로 회사용 결로 갈음 — `https://github.com/kimminsu0315/Project_Core` → `[회사용 repo]`. 직전 d16 결식 결로 박혀 있던 개인용 결식 결로는 본 사이클 결로 회사용 결로 갈음 (별도 파일 GitHubPages §3 (2) 결식 결로 박혀 있는 두 결식 중 회사용 결로 박힘 — "회사깃에 올릴 수 있도록" 사용자 결식 결로 박힘).
- index.html CSS 결식 결로 `.tag-new` 클래스 신설 — 진한 빨강 (`#B83227`) + 흰 글자 결로 박음 (기존 태그 결식 `.tag-repo`/`.tag-int`/`.tag-ext`/`.tag-draft` 결과 차별 결로 박힘).
- index.html 본체 영역 3건 (R&R·SRS·SAD) `.link-item` 결식 결로 `data-updated` 속성 박음 — `.link-updated` 결로 박힌 결과 동일 시각 결로 박음 (ISO 8601 결식 `YYYY-MM-DDTHH:MM`, KST 부재). Core_R&R = `2026-05-21T18:30`, Core_SRS·Core_SAD = `2026-05-22T00:06`. 외부전달 3건은 `.link-updated` 결식 박힘 부재 결로 자연 NEW 대상 외.
- index.html body 끝 결식 결로 NEW 배지 동작 JS 박음 — localStorage 키 `core_docs_last_visit` 결로 박힌 마지막 방문 시각 결식과 `data-updated` 결식 결로 박힌 갱신 시각 결 비교 결로 NEW 배지 동적 박힘. 처음 방문자 (lastVisit 부재) = 대상 전부 NEW (A안 결식). 페이지 로드 직후 현재 시각 결로 lastVisit 갱신 (`new Date().toISOString()`).
- 동반 시간 결 갱신 (매뉴얼 §5 결, 인프라 결 변경 결로 페이지 전체 시간 결 무조건 갱신):
  - index.html 페이지 전체 시간 결 갱신 (`2026-05-22 00:06` → `2026-05-22 11:17`).
  - 본체 결 변경 부재 결로 docs/*.md 도입부·각 링크 `.link-updated` 결식 결로는 시간 결 그대로.
- 별도 파일 GitHubPages d2 → d3 결식 결로 박음 — §6 NEW 배지 결식 신설, 기존 §6 자취 결식 → §7 결로 밀림. Pages 변경기록 결식 결로는 매뉴얼 자체 변경 자취 박힘 부재 (§1 결식 결로 docs/ 하위 범위 결로 박힘).

[d16] 2026-05-22 · index.html L196 레포 주소 결식 결로 개인용 결로 갈음 — `[회사용 repo]` → `https://github.com/kimminsu0315/Project_Core`. 직전 d14 결로 회사용 결로 갈음 박혀 있던 결을 본 사이클 결로 개인용 결로 되돌림. 매뉴얼 §11.2 (2) 버전 결식 결로 박힘 (회사용·개인용 두 결식 결로 박음, 동기화 결식 결로 박힐 시 선택 결로 박을 결).
- 동반 시간 결 세 자리 결식 결로 갱신 (매뉴얼 §11.4 결, 별도 파일 GitHubPages §5 결):
  - docs/Core_SRS.md·docs/Core_SAD.md 도입부 시간 결 갱신 (`2026-05-21 18:30` → `2026-05-22 00:06`) — 직전 별도 작업 결로 본체 파일명 자취 정정 박은 docs 두 결식만 갱신.
  - index.html L283 페이지 전체 시간 결 갱신 (`2026-05-21 18:30` → `2026-05-22 00:06`).
  - index.html Core SRS (L222)·Core SAD (L233) `.link-updated` 시간 결 갱신 (`2026-05-21 18:30` → `2026-05-22 00:06`). Core_R&R (L211) 결식 결로는 본 사이클 결 변경 부재 결로 시간 결 그대로.
- 별도 파일 GitHubPages d1 → d2 결식 결로 박음 — §4·§5 결식에 박힘 시점 결식 명시 추가 (클로드가 응답 박는 시점의 시간 결로 박음, 컨테이너 `TZ=Asia/Seoul date` 결식 결로 박힐 시각 결로 박음, 사용자 결식 결로 명시 결로 박힐 시 그 결로 갈음).

[d15] 2026-05-21 · 본체 영역 동기화 (매뉴얼 §11.2 (A) 자동 트리거 결)
- 본체/1_SRS/Core_SRS_v2_0_d68.md  → docs/Core_SRS.md
- 본체/2_SAD/Core_SAD_v1_0_d117.md → docs/Core_SAD.md
- 운영/Core_R&R_v0_1_d23.md        → docs/Core_R&R.md

동반 시간 결 박음 (§11.4 결, 세 자리):
- 각 docs/*.md 도입부 시간 결 추가 박음 (`최종 업데이트: 2026-05-21 18:30`) — §11.3 결식 결로 두 줄 결로 박음.
- index.html L274 페이지 전체 시간 결 갱신 (`2026-05-21 15:41 KST` → `2026-05-21 18:30`, KST 결 부재 결식 결로 자연 갈음).
- index.html 본체 영역 3건 (R&R·SRS·SAD) `.link-updated` 결식 결로 시간 결 박음 — CSS `.link-updated` 클래스 신설 (timestamp 결과 동일 폰트·색 결, `margin-left: auto` 결로 자연 오른쪽 정렬).

매뉴얼 §11.5 (A) 자동 동기화 자취 결식 결로 세 자리 결 명시 결식 결로 박힘 — 직전 결로는 .md 본문 동기화 자취 박지 않음 결식이었으나, 본 사이클 결로 (A) 자동 트리거 결·(B) 직접 결식 두 결식 구분 결로 박음 (매뉴얼 d107 → d108). 본 결에 시간 결식 자리 결로 세 자리 결식 결로 박힘 명시 갱신 박음 (매뉴얼 d108 → d109).

매뉴얼 §11.3 결식 결로 도입부 시간 결 두 줄 결식 결로 박힘 결 추가 (d109). §11.4 결식 결로 세 자리 결식 결로 박힘 결 갱신 (d109).

매뉴얼 §7.13 결 개선 자취 (d109) — 헤더 "사이클 공통 결" → "전 세션 공통 결" + 진행 표시 단위 결식 결로 별도 작업·단순 정정 결 추가 + 종결 응답 결 강제 명시 + 박힘 누락 자가 검수 결식 신설.

[d14] 2026-05-21 · index.html L190 레포 주소 결식 결로 회사용 결로 갈음 — `https://github.com/kimminsu0315/Project_Core` → `[회사용 repo]`. 매뉴얼 §11.2 (2) 버전 결식 결로 박힘 (회사용·개인용 두 결식 결로 박음, 동기화 결식 결로 박힐 시 선택 결로 박을 결).
- index.html L274 최종 업데이트 시간 결식 결로 현재 시간 KST 결로 갱신 — `2026-05-18 13:43 KST` → `2026-05-21 15:41 KST`. 매뉴얼 §11.4 결식 결로 박힘 (docs 결식 결로 박힐 시 결식 결로 항상 갱신 결).
- 매뉴얼 §11 GitHub Pages 동기화 결식 결로 신설 자취 (큐 10 결) — 매뉴얼 결식 결로 박힘, Pages 변경기록 결식 결로는 인프라 결만 결식 결로 박을 결.

[d13] 2026-05-18 · index.html `out/` 폴더 접기·펼치기 결 박음 — HTML5 `<details open>`/`<summary>` 결로 폴더 헤더 + 외부 전달 3건 묶음. 기본 펼침 상태(`open` 속성), 폴더 헤더 클릭 시 접힘 ↔ 펼침 토글. JavaScript 부재.
- 화살표 ▼(펼침)/▶(접힘) 결 — `.summary-arrow` 클래스, 접힘 시 `transform: rotate(-90deg)` 0.2s 트랜지션.
- 기존 marker 결 제거 — `summary::-webkit-details-marker { display: none }` + `summary::marker { display: none }` + `summary { list-style: none; cursor: pointer }`.
- 트리 라인(├─/├─/└─)·태그·링크 결 무변경.

[d12] 2026-05-18 · index.html 인덱스 외부 전달 3건을 `📁 out/` 폴더 헤더 + 트리 라인 결로 묶음 — `.folder-header`·`.folder-icon`·`.folder-name`·`.folder-desc`·`.tree-prefix` CSS 클래스 신규. 내부 결 3건(R&R·SRS·SAD)·Repo 결은 기존 형식 그대로 유지.

[d11] 2026-05-16 · head-custom.html 모바일(≤1100px) 표 강제 폭 맞춤 — `display: table; width: 100%; table-layout: fixed; overflow: visible` (Primer 기본 `display: block; width: max-content; overflow: auto`를 덮어, 가로 스크롤 대신 셀 wrap으로 세로 늘어남)
- index.html `<head>`에 viewport meta 박음 — `<meta name="viewport" content="width=device-width, initial-scale=1">` (Jekyll layout을 거치지 않는 정적 페이지라 viewport meta가 빠져 모바일에서 데스크탑 폭 980px로 축소 렌더되던 문제 해결)

[d10] 2026-05-16 · 모바일 햄버거 스크롤 방향 토글 — `body.scroll-down` 클래스 기반, 다운 시 `transform: translateY(-120%)`로 숨김, 업 시 다시 표시. 페이지 최상단 80px 이내에서는 항상 표시, ±5px 데드존으로 미세 흔들림 무시.
- 데스크탑 사이드바 접기 버튼 박음 — 사이드바 상단 우측 `toc-top-bar` 안 `‹` 버튼, 클릭 시 `body.toc-collapsed-desktop` 추가 → 사이드바 좌측 슬라이드아웃 + 본문 padding 해제 + 햄버거 버튼 표시. 햄버거 클릭 시 펼침 복귀(모바일은 기존 `toc-open` 토글 동작 유지, 데스크탑에서는 `toc-collapsed-desktop` 제거).
- 햄버거 버튼 transition에 `transform 0.25s ease` 추가 (스크롤 토글 슬라이드 연출).

[d9] 2026-05-16 · 모바일(≤1100px) 표 안 인라인 코드(`<code>`) 강제 wrap — `word-break: break-all` + `overflow-wrap: anywhere` (ICD 토픽 경로·타임스탬프 등 끊을 수 없는 토큰이 컬럼 폭을 부풀려 가로 스크롤·헤더 깨짐 유발하던 문제 해결)
- 모바일 한정 표 안 코드 폰트 축소 — `font-size: 0.82em` (컬럼 압박 추가 완화)
- 모바일 TOC 햄버거 토글 박음 — 좌상단 fixed 햄버거 버튼(44×44, 흰 배경 박스), 클릭 시 사이드바 좌측 슬라이드인 + dim 오버레이, dim 클릭/TOC 항목 클릭 시 자동 닫기
- 데스크탑(≥1101px)에서는 햄버거 버튼·dim 자동 숨김 — 기존 좌측 고정 사이드바 동작 무변경
- 기존 모바일 분기 `.toc-sidebar { display: none; }` 제거, `transform: translateX(-100%)` 슬라이드 방식으로 교체

[d7] 2026-05-15 14:24 · Mermaid CDN 버전 specific 핀 — `mermaid@11` → `mermaid@11.12.0` (민수님 VSCode 확장 동일 버전, 매뉴얼 §8.6/§8.7 명시 기준)
- ELK 레이아웃 제거 (`flowchart.defaultRenderer: 'elk'` 삭제) → dagre 기본 복귀 (vscode와 동일 엔진, 매뉴얼 §8.3 가이드와 일관성)
- VSCode 렌더와 동일 layout 검증 후 정식 적용 예정

[d6] 2026-05-15 13:15 · 변경기록 파일명: `CHANGELOG.md` → `Core_GitHub_ProjectDocs_Pages_변경기록.md` (repo 명명 컨벤션 정렬)
- 사이드바 헤더(링크) 클릭 시 자식 접기/펼치기 토글 동작 추가 (점프는 기본 동작 유지)
- Mermaid 엔진 변경 시도 — v10 → v11, dagre → ELK layout, useMaxWidth false → true (크기 정상화)

[d5] 2026-05-15 13:00 · 사이드바 트리 구조 DOM화 — h1 > h2 > h3 부모-자식 관계 (기존 flat list + CSS 들여쓰기 → 진짜 트리)
- 항목 개별 접기/펼치기 토글(▾/▸) + 사이드바 상단 [모두 펼치기/접기] 버튼
- 본문 상단 `## 목차` 섹션 Jekyll 페이지에서 JS로 자동 제거 (사이드바와 중복 해소, .md 파일 자체는 무수정)
- 변경기록 파일 신규 추가

[d4] 2026-05-15 11:58 · index에 SAD 항목 추가 (Draft 태그, ICD 위)
- 본문 마크다운 클로드 스타일 전체 적용 (Primer 위 덮어쓰기 — 배경/링크/표/코드블록/인용 모두 통일)
- TOC 트리 들여쓰기 강화 (h2/h3/h4 3단계, 색·크기 차등)
- 사이드바 홈 버튼 카드 형태로 강조

[d3] 2026-05-15 11:32 · 좌측 TOC 사이드바 신규 추가 (h2/h3 자동 추출, scrollspy)
- 화면 폭 ≤ 1100px 자동 숨김
- 사이드바 상단에 홈 백링크 표시

[d2] 2026-05-15 11:26 · Mermaid 옵션 보강 — maxTextSize 100k, maxEdges 1k, securityLevel loose, useMaxWidth false
- `.mermaid` 컨테이너 가로 스크롤 허용 (대형 다이어그램 대응)

[d1] 2026-05-15 11:01 · 신규: index.html (메인 페이지), _config.yml (Jekyll 설정), _includes/head-custom.html (Mermaid 처리)
- Primer 테마 + kramdown(GFM) + Rouge syntax highlight
- .md → 자동 .html 변환 (front matter 불필요, `defaults`로 layout 자동 적용)
