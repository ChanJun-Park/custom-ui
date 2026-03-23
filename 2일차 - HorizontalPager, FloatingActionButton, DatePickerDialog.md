# 2일차 - HorizontalPager, FloatingActionButton 애니메이션, DatePickerDialog

> 📅 2026-03-23 | Android UI 매일 3개 챌린지 Day 2

---

## 📌 오늘 배울 것

| # | UI 컴포넌트 | 난이도 | 핵심 기술 |
|---|---|---|---|
| 1 | HorizontalPager (월/주 스와이프) | ⭐⭐⭐ | Pager, PagerState, LaunchedEffect |
| 2 | FloatingActionButton 애니메이션 | ⭐⭐ | FAB, ExtendedFAB, AnimatedVisibility, LazyListState |
| 3 | DatePickerDialog (Material 3) | ⭐⭐ | DatePicker, DatePickerState, DatePickerDialog |

> 💡 **오늘의 주제는 캘린더 앱에 직접 활용 가능한 핵심 UI 패턴들입니다.**

---

## 1. 📅 HorizontalPager (월/주 단위 스와이프)

### 개념 설명

`HorizontalPager`는 Jetpack Compose에서 제공하는 스와이프 가능한 페이지 컨테이너입니다.
캘린더 앱에서 **월(Month) 또는 주(Week) 단위 좌우 스와이프**를 구현할 때 핵심이 되는 컴포넌트입니다.

기존 View 시스템의 `ViewPager2`를 대체하며, `foundation-pager` 라이브러리를 통해 사용합니다.

### 구현 방법

#### Step 1. 의존성 추가

`build.gradle.kts (app)`:

```kotlin
dependencies {
    // Compose Foundation (HorizontalPager 포함)
    implementation("androidx.compose.foundation:foundation:1.6.4")
}
```

> 📌 **참고:** `androidx.compose.foundation`은 대부분의 Compose BOM에 포함되어 있으므로, BOM을 사용한다면 버전 명시 없이 추가해도 됩니다.

---

#### Step 2. PagerState 이해하기

```kotlin
// pageCount: 총 페이지 수 (무한 스크롤을 위해 큰 수 사용)
// initialPage: 오늘 날짜를 중간 페이지로 배치 (앞뒤로 이동 가능하도록)
val pageCount = 1200 // 약 100년치 (±50년)
val initialPage = pageCount / 2 // 600번 페이지가 "오늘"

val pagerState = rememberPagerState(
    initialPage = initialPage,
    pageCount = { pageCount }
)
```

> 📌 **무한 스크롤 패턴**: 실제로 무한한 페이지는 불가능하므로, 충분히 큰 숫자(예: 1200)를 총 페이지 수로 설정하고 중간값을 현재 날짜로 매핑하는 방식을 사용합니다.

---

#### Step 3. 페이지 인덱스 → 실제 날짜 변환

```kotlin
/**
 * 페이지 인덱스를 실제 YearMonth로 변환하는 함수
 * @param page 현재 페이지 인덱스
 * @param initialPage 기준 페이지 (오늘 날짜와 매핑된 인덱스)
 * @return 해당 페이지에 해당하는 YearMonth
 */
fun pageToYearMonth(page: Int, initialPage: Int): YearMonth {
    val today = YearMonth.now()
    val offset = page - initialPage // 오늘로부터 몇 달 차이인지 계산
    return today.plusMonths(offset.toLong())
}
```

---

#### Step 4. HorizontalPager 구현

```kotlin
@Composable
fun CalendarMonthPager() {
    val pageCount = 1200
    val initialPage = pageCount / 2

    val pagerState = rememberPagerState(
        initialPage = initialPage,
        pageCount = { pageCount }
    )

    // 현재 페이지에 해당하는 YearMonth를 State로 파생
    val currentYearMonth by remember {
        derivedStateOf {
            pageToYearMonth(pagerState.currentPage, initialPage)
        }
    }

    Column {
        // 상단 월 표시 헤더
        MonthHeader(
            yearMonth = currentYearMonth,
            onPrevClick = {
                // 이전 달로 이동 (animate 사용 시 부드러운 전환)
                // coroutineScope 필요
            },
            onNextClick = {
                // 다음 달로 이동
            }
        )

        HorizontalPager(
            state = pagerState,
            modifier = Modifier.fillMaxWidth(),
            // beyondBoundsPageCount: 현재 페이지 기준 미리 렌더링할 페이지 수
            // 1로 설정하면 좌우 한 페이지씩 미리 렌더링 → 스와이프 시 끊김 방지
            beyondBoundsPageCount = 1
        ) { page ->
            val yearMonth = pageToYearMonth(page, initialPage)
            MonthCalendarGrid(yearMonth = yearMonth)
        }
    }
}
```

---

#### Step 5. 버튼 클릭으로 프로그래밍 방식 페이지 이동

```kotlin
@Composable
fun CalendarMonthPager() {
    val pagerState = rememberPagerState(/* ... */)
    // 코루틴 스코프: animateScrollToPage는 suspend 함수이므로 필요
    val coroutineScope = rememberCoroutineScope()

    Column {
        Row(
            horizontalArrangement = Arrangement.SpaceBetween,
            modifier = Modifier.fillMaxWidth().padding(horizontal = 16.dp)
        ) {
            // 이전 달 버튼
            IconButton(onClick = {
                coroutineScope.launch {
                    // animateScrollToPage: 부드러운 애니메이션으로 페이지 이동
                    pagerState.animateScrollToPage(pagerState.currentPage - 1)
                }
            }) {
                Icon(Icons.Default.ChevronLeft, contentDescription = "이전 달")
            }

            // 현재 월 텍스트 클릭 시 오늘로 이동
            TextButton(onClick = {
                coroutineScope.launch {
                    // scrollToPage: 애니메이션 없이 즉시 이동 (멀리 있을 때 유용)
                    pagerState.scrollToPage(initialPage)
                }
            }) {
                Text(
                    text = currentYearMonth.format(DateTimeFormatter.ofPattern("yyyy년 M월")),
                    style = MaterialTheme.typography.titleMedium
                )
            }

            // 다음 달 버튼
            IconButton(onClick = {
                coroutineScope.launch {
                    pagerState.animateScrollToPage(pagerState.currentPage + 1)
                }
            }) {
                Icon(Icons.Default.ChevronRight, contentDescription = "다음 달")
            }
        }

        HorizontalPager(state = pagerState) { page ->
            val yearMonth = pageToYearMonth(page, initialPage)
            MonthCalendarGrid(yearMonth = yearMonth)
        }
    }
}
```

---

#### Step 6. 페이지 전환 진행도 활용 (고급)

```kotlin
// pagerState.currentPageOffsetFraction: 현재 페이지에서 다음/이전으로 얼마나 스와이프됐는지 (-1f ~ 1f)
// 이 값을 활용해 페이지 전환 중 커스텀 시각 효과를 줄 수 있음
HorizontalPager(state = pagerState) { page ->
    val pageOffset = (pagerState.currentPage - page) +
        pagerState.currentPageOffsetFraction

    MonthCalendarGrid(
        yearMonth = pageToYearMonth(page, initialPage),
        modifier = Modifier
            .graphicsLayer {
                // 현재 페이지가 아닌 페이지는 약간 축소 + 투명하게 처리
                val scale = lerp(0.85f, 1f, 1f - pageOffset.absoluteValue.coerceIn(0f, 1f))
                scaleX = scale
                scaleY = scale
                alpha = lerp(0.5f, 1f, 1f - pageOffset.absoluteValue.coerceIn(0f, 1f))
            }
    )
}
```

---

### 핵심 정리

- `rememberPagerState(initialPage, pageCount)` 로 페이지 상태 초기화
- **무한 스크롤**: 큰 `pageCount` + 중간 인덱스를 오늘 날짜로 매핑
- `animateScrollToPage()`: 부드러운 이동 / `scrollToPage()`: 즉시 이동 (둘 다 suspend 함수)
- `beyondBoundsPageCount = 1` 로 인접 페이지 미리 렌더링 → 스와이프 끊김 방지
- `currentPageOffsetFraction` 으로 전환 중 커스텀 애니메이션 구현 가능

---

## 2. ➕ FloatingActionButton 애니메이션

### 개념 설명

`FloatingActionButton(FAB)`은 화면의 주요 액션을 담당하는 버튼입니다.
캘린더 앱에서는 "일정 추가" 버튼으로 활용되며, 두 가지 UX 패턴이 자주 쓰입니다:

- **스크롤 시 숨기기/보이기**: 스크롤 다운 → FAB 숨김, 스크롤 업 → FAB 표시
- **Extended FAB ↔ FAB 축소/확장**: 스크롤 시 라벨 텍스트가 사라지며 축소

### 구현 방법

#### Step 1. 기본 FAB + AnimatedVisibility

```kotlin
@Composable
fun CalendarScreenWithFAB() {
    val listState = rememberLazyListState()
    // 스크롤 방향 감지: 이전 첫 번째 visible item 인덱스 저장
    var lastFirstVisibleItemIndex by remember { mutableIntStateOf(0) }
    // FAB 표시 여부 상태
    var isFabVisible by remember { mutableStateOf(true) }

    // 스크롤 상태 변화를 감지해 FAB visibility 업데이트
    LaunchedEffect(listState.firstVisibleItemIndex) {
        val currentIndex = listState.firstVisibleItemIndex
        // 스크롤 다운 (인덱스 증가) → FAB 숨기기
        // 스크롤 업 (인덱스 감소) → FAB 보이기
        isFabVisible = currentIndex <= lastFirstVisibleItemIndex
        lastFirstVisibleItemIndex = currentIndex
    }

    Scaffold(
        floatingActionButton = {
            AnimatedVisibility(
                visible = isFabVisible,
                // 위에서 등장, 아래로 사라지는 애니메이션
                enter = slideInVertically(initialOffsetY = { it }) + fadeIn(),
                exit = slideOutVertically(targetOffsetY = { it }) + fadeOut()
            ) {
                FloatingActionButton(
                    onClick = { /* 일정 추가 다이얼로그 열기 */ },
                    containerColor = MaterialTheme.colorScheme.primaryContainer
                ) {
                    Icon(Icons.Default.Add, contentDescription = "일정 추가")
                }
            }
        }
    ) { paddingValues ->
        EventList(listState = listState, modifier = Modifier.padding(paddingValues))
    }
}
```

---

#### Step 2. Extended FAB ↔ 일반 FAB 전환

```kotlin
@Composable
fun CalendarFAB(listState: LazyListState) {
    // isScrollingUp: 스크롤 방향을 감지하는 확장 함수 (아래 Step 3 참고)
    val isScrollingUp by listState.isScrollingUp()

    // isScrollingUp이 true면 라벨 포함 ExtendedFAB, false면 일반 FAB
    // animate*AsState 계열이 아닌, ExtendedFloatingActionButton 자체의 expanded 파라미터를 활용
    ExtendedFloatingActionButton(
        // expanded: true → 아이콘 + 텍스트, false → 아이콘만 (부드러운 애니메이션 내장)
        expanded = isScrollingUp,
        onClick = { /* 일정 추가 */ },
        icon = {
            Icon(Icons.Default.Add, contentDescription = "일정 추가")
        },
        text = {
            Text(text = "일정 추가")
        },
        containerColor = MaterialTheme.colorScheme.primary,
        contentColor = MaterialTheme.colorScheme.onPrimary
    )
}
```

---

#### Step 3. 스크롤 방향 감지 확장 함수

```kotlin
/**
 * LazyListState에서 스크롤 방향을 감지하는 확장 함수
 * @return true: 위로 스크롤 중 (또는 정지) / false: 아래로 스크롤 중
 */
@Composable
fun LazyListState.isScrollingUp(): State<Boolean> {
    // 이전 firstVisibleItemIndex 저장 (초기값은 현재 값)
    var previousIndex by remember(this) {
        mutableIntStateOf(firstVisibleItemIndex)
    }
    // 이전 firstVisibleItemScrollOffset 저장
    var previousScrollOffset by remember(this) {
        mutableIntStateOf(firstVisibleItemScrollOffset)
    }

    return remember(this) {
        derivedStateOf {
            // 첫 번째 visible item 인덱스가 이전보다 줄어들면 → 위로 스크롤
            if (previousIndex != firstVisibleItemIndex) {
                (previousIndex > firstVisibleItemIndex).also {
                    previousIndex = firstVisibleItemIndex
                }
            } else {
                // 같은 아이템 내에서 오프셋이 감소하면 → 위로 스크롤
                (previousScrollOffset >= firstVisibleItemScrollOffset).also {
                    previousScrollOffset = firstVisibleItemScrollOffset
                }
            }
        }
    }
}
```

---

#### Step 4. Multi-FAB (여러 액션 옵션 펼치기)

```kotlin
@Composable
fun MultiActionFAB() {
    var isExpanded by remember { mutableStateOf(false) }

    Column(
        horizontalAlignment = Alignment.End,
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        // 하위 FAB들: isExpanded 상태에 따라 보이고 숨기기
        AnimatedVisibility(
            visible = isExpanded,
            enter = fadeIn() + expandVertically(),
            exit = fadeOut() + shrinkVertically()
        ) {
            Column(
                horizontalAlignment = Alignment.End,
                verticalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                // 하위 액션 1: 메모 추가
                SmallFloatingActionButton(
                    onClick = { isExpanded = false /* + 메모 추가 액션 */ },
                    containerColor = MaterialTheme.colorScheme.secondaryContainer
                ) {
                    Icon(Icons.Default.Edit, contentDescription = "메모 추가")
                }

                // 하위 액션 2: 일정 추가
                SmallFloatingActionButton(
                    onClick = { isExpanded = false /* + 일정 추가 액션 */ },
                    containerColor = MaterialTheme.colorScheme.tertiaryContainer
                ) {
                    Icon(Icons.Default.Event, contentDescription = "일정 추가")
                }
            }
        }

        // 메인 FAB: 클릭 시 isExpanded 토글, 아이콘 회전 애니메이션 추가
        val rotation by animateFloatAsState(
            targetValue = if (isExpanded) 45f else 0f,
            label = "fab_rotation"
        )

        FloatingActionButton(
            onClick = { isExpanded = !isExpanded }
        ) {
            Icon(
                imageVector = Icons.Default.Add,
                contentDescription = "메뉴 열기",
                modifier = Modifier.rotate(rotation) // + 아이콘을 45도 회전 → X 아이콘처럼 보임
            )
        }
    }
}
```

---

### 핵심 정리

- `AnimatedVisibility` + `slideInVertically/fadeIn` 조합으로 FAB 등장/퇴장 연출
- `ExtendedFloatingActionButton`의 `expanded` 파라미터로 텍스트 라벨 표시 전환 (내장 애니메이션)
- `LazyListState.isScrollingUp()` 확장 함수로 스크롤 방향을 `derivedStateOf`로 반응형 감지
- `animateFloatAsState`로 FAB 아이콘 회전 애니메이션 구현 가능

---

## 3. 🗓️ DatePickerDialog (Material 3)

### 개념 설명

Material 3의 `DatePickerDialog`는 날짜 선택 UI를 다이얼로그 형태로 제공합니다.
캘린더 앱에서 **일정의 시작/종료 날짜 선택** 또는 **특정 날짜로 빠른 이동** 기능에 활용됩니다.

Compose에서는 `material3` 라이브러리에 포함되어 있으며, 별도 라이브러리 없이 사용 가능합니다.

### 구현 방법

#### Step 1. 의존성 확인

```kotlin
dependencies {
    // Material 3 (DatePickerDialog 포함)
    implementation("androidx.compose.material3:material3:1.2.1")
}
```

---

#### Step 2. 기본 DatePickerDialog 구현

```kotlin
@Composable
fun EventDatePicker(
    onDateSelected: (Long) -> Unit, // 선택된 날짜의 epoch milliseconds 반환
    onDismiss: () -> Unit
) {
    // DatePickerState: 날짜 선택 UI의 상태를 관리
    // initialSelectedDateMillis: 초기 선택 날짜 (null이면 선택 없음)
    // yearRange: 선택 가능한 연도 범위
    val datePickerState = rememberDatePickerState(
        initialSelectedDateMillis = System.currentTimeMillis(),
        yearRange = IntRange(2020, 2030)
    )

    DatePickerDialog(
        onDismissRequest = onDismiss,
        // confirmButton: 확인 버튼 슬롯
        confirmButton = {
            TextButton(
                onClick = {
                    // selectedDateMillis: 사용자가 선택한 날짜 (UTC milliseconds)
                    datePickerState.selectedDateMillis?.let { onDateSelected(it) }
                    onDismiss()
                },
                enabled = datePickerState.selectedDateMillis != null // 날짜 선택 안 하면 비활성화
            ) {
                Text("확인")
            }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) {
                Text("취소")
            }
        }
    ) {
        // DatePicker: 다이얼로그 내부에 배치되는 실제 날짜 선택 UI
        DatePicker(state = datePickerState)
    }
}
```

---

#### Step 3. 날짜 선택 결과를 LocalDate로 변환

```kotlin
/**
 * epoch milliseconds (UTC)를 LocalDate로 변환하는 확장 함수
 * DatePickerDialog가 반환하는 값은 UTC 기준이므로, 로컬 타임존 변환이 필요합니다
 */
fun Long.toLocalDate(): LocalDate {
    return Instant.ofEpochMilli(this)
        .atZone(ZoneId.of("UTC")) // DatePicker는 UTC 자정(00:00)으로 반환
        .toLocalDate()
}

// 사용 예시
val selectedDate = dateMillis.toLocalDate()
// → 2026-03-23 형태의 LocalDate
```

> ⚠️ **주의**: `DatePickerDialog`가 반환하는 epoch ms는 **UTC 기준 자정(00:00)**입니다.
> `ZoneId.systemDefault()`를 사용하면 타임존에 따라 날짜가 하루 밀릴 수 있으므로 반드시 UTC로 변환하세요.

---

#### Step 4. 날짜 범위 선택 (DateRangePicker)

일정의 시작일/종료일을 한 번에 선택하는 UI입니다:

```kotlin
@Composable
fun EventDateRangePicker(
    onRangeSelected: (startMillis: Long, endMillis: Long) -> Unit,
    onDismiss: () -> Unit
) {
    // DateRangePickerState: 시작일 + 종료일 상태 관리
    val dateRangePickerState = rememberDateRangePickerState(
        initialSelectedStartDateMillis = System.currentTimeMillis()
    )

    DatePickerDialog(
        onDismissRequest = onDismiss,
        confirmButton = {
            TextButton(
                onClick = {
                    val start = dateRangePickerState.selectedStartDateMillis
                    val end = dateRangePickerState.selectedEndDateMillis
                    if (start != null && end != null) {
                        onRangeSelected(start, end)
                    }
                    onDismiss()
                },
                // 시작일과 종료일 모두 선택됐을 때만 활성화
                enabled = dateRangePickerState.selectedStartDateMillis != null &&
                          dateRangePickerState.selectedEndDateMillis != null
            ) {
                Text("확인")
            }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) { Text("취소") }
        }
    ) {
        // DateRangePicker: 두 날짜 범위를 선택하는 UI (하이라이트 표시)
        DateRangePicker(
            state = dateRangePickerState,
            modifier = Modifier.weight(1f) // 다이얼로그 내에서 가용 공간 차지
        )
    }
}
```

---

#### Step 5. 화면에서 DatePickerDialog 사용하기

```kotlin
@Composable
fun AddEventScreen() {
    var showDatePicker by remember { mutableStateOf(false) }
    var selectedDate by remember { mutableStateOf<LocalDate?>(null) }

    Column(modifier = Modifier.padding(16.dp)) {
        // 날짜 표시 필드 (클릭 시 DatePicker 열기)
        OutlinedTextField(
            value = selectedDate?.format(DateTimeFormatter.ofPattern("yyyy년 M월 d일")) ?: "날짜를 선택하세요",
            onValueChange = {}, // 직접 입력 불가, 클릭으로만 선택
            readOnly = true,
            label = { Text("일정 날짜") },
            trailingIcon = {
                IconButton(onClick = { showDatePicker = true }) {
                    Icon(Icons.Default.CalendarToday, contentDescription = "날짜 선택")
                }
            },
            modifier = Modifier
                .fillMaxWidth()
                // 텍스트 필드 전체를 클릭 가능하게
                .clickable { showDatePicker = true }
        )

        Spacer(modifier = Modifier.height(16.dp))

        Button(
            onClick = { /* 일정 저장 */ },
            enabled = selectedDate != null,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("일정 저장")
        }
    }

    // DatePickerDialog: showDatePicker가 true일 때만 표시
    if (showDatePicker) {
        EventDatePicker(
            onDateSelected = { millis ->
                selectedDate = millis.toLocalDate()
                showDatePicker = false
            },
            onDismiss = { showDatePicker = false }
        )
    }
}
```

---

#### Step 6. 날짜 선택 가능 범위 제한

과거 날짜 선택 금지, 특정 날짜만 활성화 등 제약 조건을 설정할 수 있습니다:

```kotlin
DatePicker(
    state = datePickerState,
    // DateValidator: 날짜마다 선택 가능 여부를 판단하는 함수
    dateValidator = { dateMillis ->
        // 오늘 이후의 날짜만 선택 가능 (미래 날짜 선택 UI)
        val today = LocalDate.now()
        val selectedDate = Instant.ofEpochMilli(dateMillis)
            .atZone(ZoneId.of("UTC"))
            .toLocalDate()
        !selectedDate.isBefore(today) // 오늘 포함, 이후 날짜만 허용
    }
)
```

---

### 핵심 정리

- `rememberDatePickerState(initialSelectedDateMillis, yearRange)` 로 초기 상태 설정
- 반환값은 **UTC 기준 epoch ms** → `ZoneId.of("UTC")`로 변환 (타임존 이슈 주의!)
- `DateRangePicker`: 시작일/종료일을 한 번에 선택하는 범위 선택 UI
- `dateValidator`로 선택 가능 날짜 범위 제한 (예: 오늘 이후만 선택 가능)
- `showDialog` 상태 변수로 다이얼로그 표시/숨김 제어

---

## 📝 오늘의 학습 요약

| UI | 핵심 API | 기억할 포인트 |
|---|---|---|
| HorizontalPager | `rememberPagerState`, `HorizontalPager` | 큰 pageCount + 중간 인덱스 = 오늘 날짜로 무한 스크롤 구현 |
| FAB 애니메이션 | `ExtendedFAB`, `AnimatedVisibility`, `animateFloatAsState` | `isScrollingUp()` 확장 함수로 스크롤 방향 반응형 감지 |
| DatePickerDialog | `rememberDatePickerState`, `DatePickerDialog` | 반환값은 UTC epoch ms → `ZoneId.of("UTC")`로 변환 필수 |

---

## 🔗 참고 자료

- [Compose Pager 공식 문서](https://developer.android.com/develop/ui/compose/layouts/pager)
- [FloatingActionButton - Material 3](https://m3.material.io/components/floating-action-button/overview)
- [DatePicker - Material 3](https://m3.material.io/components/date-pickers/overview)
- [Compose Foundation BOM](https://developer.android.com/jetpack/compose/bom/bom-mapping)
