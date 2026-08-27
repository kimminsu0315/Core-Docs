# 물류 AMMR 설비 ID · Slot ID 목록

> 이 문서는 `Core_설비SlotID_AMMR_v1_0_d20.md` 기준으로 작성되었습니다.
> 최종 업데이트: 2026-08-28 00:10

---

## 1. 문서 개요

### 1.1 목적

이 문서는 물류 AMMR이 Core와 주고받는 설비 ID와 Slot ID의 전체 목록을 정의한다. Core ↔ 물류 AMMR
Interface Control Document가 설치 시점에 Core 측이 제공한다고 정한 그 목록이 이 문서다. AMMR 업체는
이 문서의 값으로 자기 맵의 목적지와 Slot을 대응시킨다.

이 문서의 모든 값은 Core가 확정한 것이며, AMMR 측 구현이 이 값에 맞춘다.

### 1.2 범위

이 문서는 물류 AMMR이 이동하고 적재·하역하는 AMMR 외부 설비의 식별자와, 물류 AMMR 자체의 Slot
식별자를 함께 다룬다. 자체 Slot은 Core가 인식하는 식별자 체계와 Slot 구성까지 이 문서가 정하며,
그 값을 주고받는 메시지 계약은 Core ↔ 물류 AMMR Interface Control Document가 갖는다.

다음은 이 문서 범위 밖이다.

- 설비의 물리 좌표와 배치. 이동 목적지는 좌표가 아니라 설비 ID로 지정하며, 물리 위치 해석은 AMMR이 자기 맵으로 수행한다.
- 충전 스테이션. 태블릿 설정값이 단일 출처이며 Charge Job에는 위치 필드를 포함하지 않는다.
- Slot의 점유 상태와 그 판정. 상태 보고 계약은 Core ↔ 물류 AMMR Interface Control Document가 갖는다.

### 1.3 설비 구성

대상 설비는 통합 Slot WIP과 CNC 작업대 두 종류다. CNC 작업대는 외부 시스템이 전 설비를
내려주지만 그 가운데 물류 AMMR 자동화 대상만 이 목록에 오른다. 물류 AMMR 자체 Slot은 설비가
아니므로 설비 합계와 별도로 적는다.

| 구분          | 대수 | 1대의 Slot 수 | Slot 합계 |
|---------------|------|---------------|-----------|
| 통합 Slot WIP | 10   | 18            | 180       |
| CNC 작업대    | 30   | 2             | 60        |
| 설비 합계     | 40   |               | 240       |
| 물류 AMMR     | 2    | 6             | 12        |

---

## 2. 식별자 체계

설비 ID와 Slot ID 두 층위다. 이동 목적지는 설비 ID로 지정하고, Unit을 적재하거나 하역하는 대상은
Slot ID로 지정한다. 설비와 물류 AMMR 모두 앞에 자기 식별자를 두고 Slot 구분을 뒤에 붙이는 같은 구조를 따른다.

| 대상                 | 형식                       | 예                   |
|----------------------|----------------------------|----------------------|
| 통합 Slot WIP        | `WIP-{공정코드}{일련번호}` | `WIP-CLN001`         |
| 통합 Slot WIP의 Slot | `{설비 ID}-{열}{행}`       | `WIP-CLN001-A1`      |
| CNC 작업대           | `CNC-{장비명}`             | `CNC-RAC-A01`        |
| CNC 작업대의 Slot    | `{설비 ID}-{구분}`         | `CNC-RAC-A01-BEFORE` |
| 물류 AMMR            | `AMMR-{타입}{일련번호}`    | `AMMR-LOGI001`       |
| 물류 AMMR의 Slot     | `{AMMR 식별자}-{열}{행}`   | `AMMR-LOGI001-A1`    |

- 일련번호는 3자리 0채움으로 표기한다 (`001`~`999`).
- Slot의 열은 `A`부터 `ZZ`까지, 행은 `1`부터 `999`까지 부여하며 행에는 0채움을 적용하지 않는다.
- CNC 장비명은 외부 시스템이 제공하는 장비명 값을 그대로 쓴다.
- CNC 작업대 Slot의 구분값은 `BEFORE`와 `AFTER` 두 가지다.
- CNC 작업대는 WIP이 아니므로 `WIP-` 접두를 쓰지 않는다.
- 물류 AMMR의 Slot은 열을 `A`로 고정하고 행을 `1`부터 `6`까지 부여한다.

물류 AMMR의 Slot 구성과 실물 목록은 물류 AMMR 절이 정한다.

---

## 3. 통합 Slot WIP

### 3.1 공정 코드

| 공정          | 코드  | 설비 ID 예   |
|---------------|-------|--------------|
| CNC WIP       | `CNC` | `WIP-CNC001` |
| DP            | `DP`  | `WIP-DP001`  |
| 세척·블라스팅 | `CLN` | `WIP-CLN001` |

세척과 블라스팅은 같은 물리 설비를 공유하므로 하나의 공정 코드를 쓴다.

### 3.2 설비 ID

| 설비 ID      | 공정          | Slot 수 |
|--------------|---------------|---------|
| `WIP-CNC001` | CNC WIP       | 18      |
| `WIP-CNC002` | CNC WIP       | 18      |
| `WIP-CNC003` | CNC WIP       | 18      |
| `WIP-CNC004` | CNC WIP       | 18      |
| `WIP-DP001`  | DP            | 18      |
| `WIP-DP002`  | DP            | 18      |
| `WIP-CLN001` | 세척·블라스팅 | 18      |
| `WIP-CLN002` | 세척·블라스팅 | 18      |
| `WIP-CLN003` | 세척·블라스팅 | 18      |
| `WIP-CLN004` | 세척·블라스팅 | 18      |

### 3.3 Slot ID 구성

설비 1대는 3열 6행, 모두 18 Slot이다. Slot ID는 설비 ID 뒤에 열 문자와 행 번호를 붙여 만든다.

```text
            열 A      열 B      열 C
          ┌────────┬────────┬────────┐
   행 1   │   A1   │   B1   │   C1   │
          ├────────┼────────┼────────┤
   행 2   │   A2   │   B2   │   C2   │
          ├────────┼────────┼────────┤
   행 3   │   A3   │   B3   │   C3   │
          ├────────┼────────┼────────┤
   행 4   │   A4   │   B4   │   C4   │
          ├────────┼────────┼────────┤
   행 5   │   A5   │   B5   │   C5   │
          ├────────┼────────┼────────┤
   행 6   │   A6   │   B6   │   C6   │
          └────────┴────────┴────────┘
```

위 도식은 열과 행이 조합되는 방식만 나타낸다. 각 열·행이 실물의 어느 칸인지는 설치 시점에 확정하며,
식별자 형식은 물리 배치와 무관하다.

통합 Slot WIP의 Slot에는 입고·출고 구분을 두지 않는다.

### 3.4 Slot ID 목록

```text
WIP-CNC001
   WIP-CNC001-A1   WIP-CNC001-B1   WIP-CNC001-C1
   WIP-CNC001-A2   WIP-CNC001-B2   WIP-CNC001-C2
   WIP-CNC001-A3   WIP-CNC001-B3   WIP-CNC001-C3
   WIP-CNC001-A4   WIP-CNC001-B4   WIP-CNC001-C4
   WIP-CNC001-A5   WIP-CNC001-B5   WIP-CNC001-C5
   WIP-CNC001-A6   WIP-CNC001-B6   WIP-CNC001-C6

WIP-CNC002
   WIP-CNC002-A1   WIP-CNC002-B1   WIP-CNC002-C1
   WIP-CNC002-A2   WIP-CNC002-B2   WIP-CNC002-C2
   WIP-CNC002-A3   WIP-CNC002-B3   WIP-CNC002-C3
   WIP-CNC002-A4   WIP-CNC002-B4   WIP-CNC002-C4
   WIP-CNC002-A5   WIP-CNC002-B5   WIP-CNC002-C5
   WIP-CNC002-A6   WIP-CNC002-B6   WIP-CNC002-C6

WIP-CNC003
   WIP-CNC003-A1   WIP-CNC003-B1   WIP-CNC003-C1
   WIP-CNC003-A2   WIP-CNC003-B2   WIP-CNC003-C2
   WIP-CNC003-A3   WIP-CNC003-B3   WIP-CNC003-C3
   WIP-CNC003-A4   WIP-CNC003-B4   WIP-CNC003-C4
   WIP-CNC003-A5   WIP-CNC003-B5   WIP-CNC003-C5
   WIP-CNC003-A6   WIP-CNC003-B6   WIP-CNC003-C6

WIP-CNC004
   WIP-CNC004-A1   WIP-CNC004-B1   WIP-CNC004-C1
   WIP-CNC004-A2   WIP-CNC004-B2   WIP-CNC004-C2
   WIP-CNC004-A3   WIP-CNC004-B3   WIP-CNC004-C3
   WIP-CNC004-A4   WIP-CNC004-B4   WIP-CNC004-C4
   WIP-CNC004-A5   WIP-CNC004-B5   WIP-CNC004-C5
   WIP-CNC004-A6   WIP-CNC004-B6   WIP-CNC004-C6

WIP-DP001
   WIP-DP001-A1   WIP-DP001-B1   WIP-DP001-C1
   WIP-DP001-A2   WIP-DP001-B2   WIP-DP001-C2
   WIP-DP001-A3   WIP-DP001-B3   WIP-DP001-C3
   WIP-DP001-A4   WIP-DP001-B4   WIP-DP001-C4
   WIP-DP001-A5   WIP-DP001-B5   WIP-DP001-C5
   WIP-DP001-A6   WIP-DP001-B6   WIP-DP001-C6

WIP-DP002
   WIP-DP002-A1   WIP-DP002-B1   WIP-DP002-C1
   WIP-DP002-A2   WIP-DP002-B2   WIP-DP002-C2
   WIP-DP002-A3   WIP-DP002-B3   WIP-DP002-C3
   WIP-DP002-A4   WIP-DP002-B4   WIP-DP002-C4
   WIP-DP002-A5   WIP-DP002-B5   WIP-DP002-C5
   WIP-DP002-A6   WIP-DP002-B6   WIP-DP002-C6

WIP-CLN001
   WIP-CLN001-A1   WIP-CLN001-B1   WIP-CLN001-C1
   WIP-CLN001-A2   WIP-CLN001-B2   WIP-CLN001-C2
   WIP-CLN001-A3   WIP-CLN001-B3   WIP-CLN001-C3
   WIP-CLN001-A4   WIP-CLN001-B4   WIP-CLN001-C4
   WIP-CLN001-A5   WIP-CLN001-B5   WIP-CLN001-C5
   WIP-CLN001-A6   WIP-CLN001-B6   WIP-CLN001-C6

WIP-CLN002
   WIP-CLN002-A1   WIP-CLN002-B1   WIP-CLN002-C1
   WIP-CLN002-A2   WIP-CLN002-B2   WIP-CLN002-C2
   WIP-CLN002-A3   WIP-CLN002-B3   WIP-CLN002-C3
   WIP-CLN002-A4   WIP-CLN002-B4   WIP-CLN002-C4
   WIP-CLN002-A5   WIP-CLN002-B5   WIP-CLN002-C5
   WIP-CLN002-A6   WIP-CLN002-B6   WIP-CLN002-C6

WIP-CLN003
   WIP-CLN003-A1   WIP-CLN003-B1   WIP-CLN003-C1
   WIP-CLN003-A2   WIP-CLN003-B2   WIP-CLN003-C2
   WIP-CLN003-A3   WIP-CLN003-B3   WIP-CLN003-C3
   WIP-CLN003-A4   WIP-CLN003-B4   WIP-CLN003-C4
   WIP-CLN003-A5   WIP-CLN003-B5   WIP-CLN003-C5
   WIP-CLN003-A6   WIP-CLN003-B6   WIP-CLN003-C6

WIP-CLN004
   WIP-CLN004-A1   WIP-CLN004-B1   WIP-CLN004-C1
   WIP-CLN004-A2   WIP-CLN004-B2   WIP-CLN004-C2
   WIP-CLN004-A3   WIP-CLN004-B3   WIP-CLN004-C3
   WIP-CLN004-A4   WIP-CLN004-B4   WIP-CLN004-C4
   WIP-CLN004-A5   WIP-CLN004-B5   WIP-CLN004-C5
   WIP-CLN004-A6   WIP-CLN004-B6   WIP-CLN004-C6
```

---

## 4. CNC 작업대

### 4.1 Slot ID 구성

설비 1대는 입고 1 Slot과 출고 1 Slot, 모두 2 Slot이다. Slot ID는 설비 ID 뒤에 구분값을 붙여 만든다.

| 구분값   | 명칭      | 정의                           |
|----------|-----------|--------------------------------|
| `BEFORE` | 입고 Slot | 가공 전 Unit을 받는 Slot       |
| `AFTER`  | 출고 Slot | 가공을 마친 Unit이 나오는 Slot |

### 4.2 설비 ID · Slot ID

물류 AMMR 자동화 대상은 CNC 작업대 30대다. 설비 ID는 외부 시스템이 내려주는
장비명에서 파생하며, 아래 30개가 Core가 확정해 제공하는 값이다.

대상 설비가 늘거나 바뀌면 Core가 갱신본을 제공한다.

| 설비 ID       | 장비명    | 입고 Slot ID         | 출고 Slot ID        |
|---------------|-----------|----------------------|---------------------|
| `CNC-RAC-A01` | `RAC-A01` | `CNC-RAC-A01-BEFORE` | `CNC-RAC-A01-AFTER` |
| `CNC-RAC-A02` | `RAC-A02` | `CNC-RAC-A02-BEFORE` | `CNC-RAC-A02-AFTER` |
| `CNC-RAC-A03` | `RAC-A03` | `CNC-RAC-A03-BEFORE` | `CNC-RAC-A03-AFTER` |
| `CNC-RAC-A04` | `RAC-A04` | `CNC-RAC-A04-BEFORE` | `CNC-RAC-A04-AFTER` |
| `CNC-RAC-A05` | `RAC-A05` | `CNC-RAC-A05-BEFORE` | `CNC-RAC-A05-AFTER` |
| `CNC-RAC-A06` | `RAC-A06` | `CNC-RAC-A06-BEFORE` | `CNC-RAC-A06-AFTER` |
| `CNC-RAC-A07` | `RAC-A07` | `CNC-RAC-A07-BEFORE` | `CNC-RAC-A07-AFTER` |
| `CNC-RAC-A08` | `RAC-A08` | `CNC-RAC-A08-BEFORE` | `CNC-RAC-A08-AFTER` |
| `CNC-RAC-A09` | `RAC-A09` | `CNC-RAC-A09-BEFORE` | `CNC-RAC-A09-AFTER` |
| `CNC-RAC-A10` | `RAC-A10` | `CNC-RAC-A10-BEFORE` | `CNC-RAC-A10-AFTER` |
| `CNC-RAC-A11` | `RAC-A11` | `CNC-RAC-A11-BEFORE` | `CNC-RAC-A11-AFTER` |
| `CNC-RAC-A12` | `RAC-A12` | `CNC-RAC-A12-BEFORE` | `CNC-RAC-A12-AFTER` |
| `CNC-RAC-A13` | `RAC-A13` | `CNC-RAC-A13-BEFORE` | `CNC-RAC-A13-AFTER` |
| `CNC-RAC-A14` | `RAC-A14` | `CNC-RAC-A14-BEFORE` | `CNC-RAC-A14-AFTER` |
| `CNC-RAC-A15` | `RAC-A15` | `CNC-RAC-A15-BEFORE` | `CNC-RAC-A15-AFTER` |
| `CNC-RAC-A16` | `RAC-A16` | `CNC-RAC-A16-BEFORE` | `CNC-RAC-A16-AFTER` |
| `CNC-RAC-A17` | `RAC-A17` | `CNC-RAC-A17-BEFORE` | `CNC-RAC-A17-AFTER` |
| `CNC-RAC-A18` | `RAC-A18` | `CNC-RAC-A18-BEFORE` | `CNC-RAC-A18-AFTER` |
| `CNC-RAC-A19` | `RAC-A19` | `CNC-RAC-A19-BEFORE` | `CNC-RAC-A19-AFTER` |
| `CNC-RAC-A20` | `RAC-A20` | `CNC-RAC-A20-BEFORE` | `CNC-RAC-A20-AFTER` |
| `CNC-RAC-A21` | `RAC-A21` | `CNC-RAC-A21-BEFORE` | `CNC-RAC-A21-AFTER` |
| `CNC-RAC-A22` | `RAC-A22` | `CNC-RAC-A22-BEFORE` | `CNC-RAC-A22-AFTER` |
| `CNC-RAC-A23` | `RAC-A23` | `CNC-RAC-A23-BEFORE` | `CNC-RAC-A23-AFTER` |
| `CNC-RAC-A24` | `RAC-A24` | `CNC-RAC-A24-BEFORE` | `CNC-RAC-A24-AFTER` |
| `CNC-RAC-A25` | `RAC-A25` | `CNC-RAC-A25-BEFORE` | `CNC-RAC-A25-AFTER` |
| `CNC-RAC-A26` | `RAC-A26` | `CNC-RAC-A26-BEFORE` | `CNC-RAC-A26-AFTER` |
| `CNC-RAC-A27` | `RAC-A27` | `CNC-RAC-A27-BEFORE` | `CNC-RAC-A27-AFTER` |
| `CNC-RAC-A28` | `RAC-A28` | `CNC-RAC-A28-BEFORE` | `CNC-RAC-A28-AFTER` |
| `CNC-RAC-A29` | `RAC-A29` | `CNC-RAC-A29-BEFORE` | `CNC-RAC-A29-AFTER` |
| `CNC-RAC-A30` | `RAC-A30` | `CNC-RAC-A30-BEFORE` | `CNC-RAC-A30-AFTER` |

---

## 5. 물류 AMMR

### 5.1 Slot ID 구성

물류 AMMR 1대는 1열 6행, 모두 6 Slot이다. Slot ID는 AMMR 식별자 뒤에 열 문자와 행 번호를 붙여 만든다.

```text
            열 A
          ┌────────┐
   행 1   │   A1   │
          ├────────┤
   행 2   │   A2   │
          ├────────┤
   행 3   │   A3   │
          ├────────┤
   행 4   │   A4   │
          ├────────┤
   행 5   │   A5   │
          ├────────┤
   행 6   │   A6   │
          └────────┘
```

위 도식의 위아래는 적재 칸의 위아래와 같다. 행 1이 맨 위 칸이고 행 6이 맨 아래 칸이며, 행 번호가
커질수록 아래로 내려간다. 통합 Slot WIP과 달리 이 대응은 설치 시점에 확정하지 않는다. Core가
확정한 값이므로 AMMR 측 구현이 이 배치에 맞춘다.

행 번호는 태블릿 화면이 표시하는 Slot 번호와 같다.

### 5.2 설비 ID · Slot ID

물류 AMMR은 현재 2대를 운영한다. AMMR 식별자는 설치 시점에 Core가 할당하며, 아래가 Core가 확정해
제공하는 값이다.

운영 대수가 늘거나 바뀌면 Core가 갱신본을 제공한다.

```text
AMMR-LOGI001
   AMMR-LOGI001-A1
   AMMR-LOGI001-A2
   AMMR-LOGI001-A3
   AMMR-LOGI001-A4
   AMMR-LOGI001-A5
   AMMR-LOGI001-A6

AMMR-LOGI002
   AMMR-LOGI002-A1
   AMMR-LOGI002-A2
   AMMR-LOGI002-A3
   AMMR-LOGI002-A4
   AMMR-LOGI002-A5
   AMMR-LOGI002-A6
```

---

## 6. 계약 필드 대응

Core ↔ 물류 AMMR Interface Control Document의 어느 필드에 어느 층위가 실리는지 정리한다.

| 필드                                       | 값 층위                                  | 예                                 |
|--------------------------------------------|------------------------------------------|------------------------------------|
| `work_location_id`                         | 설비 ID                                  | `WIP-CLN001`, `CNC-RAC-A02`        |
| `from_location_id`, `to_location_id`       | 설비 ID                                  | `WIP-CLN001`, `CNC-RAC-A02`        |
| `slot_info`의 `from_slot_id`, `to_slot_id` | Slot ID                                  | `WIP-CLN001-A1`, `AMMR-LOGI001-A1` |
| `ammr_id`                                  | 물류 AMMR 식별자                         | `AMMR-LOGI001`                     |
| `slot_id`                                  | 물류 AMMR 자체 Slot ID                   | `AMMR-LOGI001-A3`                  |
| `node_id`                                  | AMMR 맵의 Node ID. 설비 ID 체계와 별개다 | `WIP-CLN001`                       |
