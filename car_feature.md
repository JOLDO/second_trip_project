# 🚗 Car (차량 렌트) 기능 문서

## 📱 화면 흐름

```mermaid
flowchart LR
    A[CarRentHomeScreen\n지역·날짜 선택] -->|달력 아이콘 탭| B[CalendarScreen\n날짜·시간 선택]
    B -->|확인| A
    A -->|확인 버튼| C[CarListScreen\n차량 목록]
    C -->|차량 선택 후 예약하기| D[CarReservationScreen\n예약 확인·결제]
    D -->|예약 완료| E[홈으로 이동]
```

---

## 🏗️ 레이어 구조

```mermaid
flowchart TD
    subgraph Screen["📱 Screen (UI)"]
        S1[CarRentHomeScreen]
        S2[CalendarScreen]
        S3[CarListScreen]
        S4[CarReservationScreen]
    end

    subgraph Controller["🧠 Controller (Provider/ChangeNotifier)"]
        C1[CarRentHomeController\n지역 목록 상태]
        C2[CalendarController\n날짜·시간 범위 상태]
        C3[CarRentListController\n차량 목록 + 커서 페이징]
        C4[CarReservationController\n예약 생성·조회·취소]
    end

    subgraph Service["🔌 Service (API 호출)"]
        SV1[CarRentHomeService]
        SV2[CarRentListService]
        SV3[CarReservationService]
    end

    subgraph API["🌐 API"]
        A1["publicDio\nGET /car/regions\nGET /car/search/all"]
        A2["dio (인증)\nPOST /api/car/reservation\nGET /api/car/reservation/my\nDELETE /api/car/reservation/:id"]
    end

    S1 --> C1 & C2
    S2 --> C2
    S3 --> C3
    S4 --> C4

    C1 --> SV1 --> A1
    C3 --> SV2 --> A1
    C4 --> SV3 --> A2
```

---

## 📄 커서 페이징 흐름 (차량 목록)

```mermaid
sequenceDiagram
    participant Screen as CarListScreen
    participant Controller as CarRentListController
    participant Service as CarRentListService
    participant Server

    Screen->>Controller: fetchAvailableCars(region, startDate, endDate)
    Controller->>Service: searchCars(request, cursor=null)
    Service->>Server: GET /car/search/all
    Server-->>Service: content[], hasNext, nextCursorPrice, nextCursorName
    Service-->>Controller: CarSearchCursorResponseDTO
    Controller-->>Screen: cars 업데이트, notifyListeners()

    Note over Screen: 스크롤이 하단 200px 이내 도달
    Screen->>Controller: loadMoreCars()
    Controller->>Service: searchCars(request, cursor=nextCursor)
    Service->>Server: GET /car/search/all?cursorPrice=...
    Server-->>Service: 다음 페이지 데이터
    Controller-->>Screen: cars에 추가, notifyListeners()
```

---

## 📝 예약 생성 흐름

```mermaid
sequenceDiagram
    participant Screen as CarReservationScreen
    participant Controller as CarReservationController
    participant Service as CarReservationService
    participant Server

    Screen->>Controller: createRental(carId, startDate, endDate)
    Controller->>Service: createRental()
    Service->>Server: POST /api/car/reservation

    alt 성공
        Server-->>Service: CarRentalReservationDTO
        Controller-->>Screen: 예약 완료 → 홈으로 이동
    else 401/403 (미로그인)
        Server-->>Service: 401
        Controller-->>Screen: errorMessage = "로그인이 필요합니다."
        Screen->>Screen: 로그인 화면 이동 후 재시도
    else 기타 실패
        Server-->>Service: 에러
        Controller-->>Screen: errorMessage 표시 (SnackBar)
    end
```

---

## 📅 CalendarController 날짜 선택 로직

```mermaid
flowchart TD
    A[날짜 탭] --> B{rangeStart == null\n또는 rangeEnd != null?}
    B -->|Yes| C[rangeStart = 선택날짜\nrangeEnd = null]
    B -->|No\n시작일만 선택된 상태| D{선택날짜가\nrangeStart 이전?}
    D -->|Yes| E[rangeStart 재설정]
    D -->|No| F[rangeEnd = 선택날짜]
    C & E & F --> G[notifyListeners]
```

---

## 🧩 컨트롤러 역할 요약

| 컨트롤러 | 상태 | 주요 메서드 |
|---|---|---|
| `CarRentHomeController` | `regions`, `isLoading` | `fetchRegions()` |
| `CalendarController` | `rangeStart`, `rangeEnd`, `startTime`, `endTime` | `onDaySelected()`, `saveState()`, `restoreState()` |
| `CarRentListController` | `cars[]`, `hasNext`, 커서값 | `fetchAvailableCars()`, `loadMoreCars()` |
| `CarReservationController` | `myRentals[]`, `hasNext`, 커서값 | `createRental()`, `fetchMyRentals()`, `cancelRental()` |

---

## ⚠️ 특이사항

- **CalendarScreen 뒤로가기 취소**
  - `CarRentHomeScreen`에서 `saveState()`로 상태 백업 후 달력 이동
  - 취소 시 `restoreState()`로 이전 상태 복원

- **Shimmer 로딩**
  - 차량 목록 최초 로딩 중: shimmer 카드 10개 표시
  - 추가 로딩(페이징) 중: 리스트 마지막에 shimmer 1개 추가

- **AutomaticKeepAliveClientMixin**
  - 차량 카드의 "더보기" 펼침 상태가 스크롤 시 초기화되는 것을 방지

- **차량 타입별 이미지**
  - `SUV` / `대형` / `중형` / `소형` / `경형` / `승합` 각각 다른 assets 이미지 사용
