# 3일차 - Android Window란?, Edge-to-Edge & WindowInsets, Compose Custom Layout

> 📅 2026-03-24 | Android UI 매일 3개 챌린지 Day 3

---

## 📌 오늘 배울 것

| # | 주제 | 난이도 | 핵심 기술 |
|---|---|---|---|
| 1 | Android Window란? | ⭐⭐⭐ | Window, PhoneWindow, WindowManager, Surface |
| 2 | Edge-to-Edge & WindowInsets 처리 | ⭐⭐⭐ | enableEdgeToEdge(), WindowInsetsCompat, padding |
| 3 | Compose Custom Layout | ⭐⭐⭐ | Layout, MeasurePolicy, SubcomposeLayout |

> 💡 **오늘의 주제는 Android UI의 '뼈대'에 해당합니다. Window 개념을 이해하면 UI 관련 버그의 절반은 원인을 바로 파악할 수 있습니다.**

---

## 1. 🪟 Android Window란?

### 개념 설명

많은 개발자들이 "화면 = Activity"라고 생각하지만, 정확히는 **화면 = Window**입니다.
`Window`는 Android UI 시스템의 가장 기본 단위로, 화면에 그려지는 모든 것은 반드시 Window 위에 올라갑니다.

---

### Window의 계층 구조

```
Android UI 전체 구조
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 WindowManager (시스템 서비스)
 │
 ├── Window (상태바, 내비게이션 바 - 시스템 소유)
 │
 ├── Window ← Activity의 PhoneWindow
 │    └── DecorView (Window의 최상위 View)
 │         ├── StatusBar 영역
 │         ├── ContentView (setContentView()로 우리가 올리는 곳)
 │         └── NavigationBar 영역
 │
 ├── Window ← Dialog의 Window
 │    └── DecorView
 │
 └── Window ← 다른 앱의 오버레이 (SYSTEM_ALERT_WINDOW)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### 핵심 구성요소 4가지

#### 1) Window (추상 클래스)

```kotlin
// Window는 추상 클래스 - 직접 사용하지 않음
// Activity에서 접근하는 방법:
val window: Window = activity.window

// 자주 쓰는 Window 조작
window.statusBarColor = Color.TRANSPARENT       // 상태바 색상 변경
window.navigationBarColor = Color.TRANSPARENT   // 내비게이션 바 색상 변경
window.addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON) // 화면 켜짐 유지
```

#### 2) PhoneWindow (Window의 실제 구현체)

```
우리가 쓰는 Window는 사실 PhoneWindow입니다.
Activity.attach() 내부에서 자동으로 생성되며,
setContentView()를 호출하면 PhoneWindow 내부의 ContentView에 레이아웃이 추가됩니다.

Activity.setContentView(R.layout.main)
    └─→ PhoneWindow.setContentView()
            └─→ DecorView에 ContentView로 inflate
```

#### 3) WindowManager

```kotlin
// WindowManager: Window를 추가/제거/업데이트하는 시스템 서비스
val windowManager = getSystemService(Context.WINDOW_SERVICE) as WindowManager

// WindowManager.LayoutParams: Window의 크기, 위치, 타입, 플래그 설정
val params = WindowManager.LayoutParams(
    WindowManager.LayoutParams.WRAP_CONTENT,
    WindowManager.LayoutParams.WRAP_CONTENT,
    WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY, // 다른 앱 위에 표시 (권한 필요)
    WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE,       // 포커스 받지 않음
    PixelFormat.TRANSLUCENT
)

// 커스텀 View를 Window에 직접 추가 (Activity 없이도 가능)
val overlayView = LayoutInflater.from(context).inflate(R.layout.overlay, null)
windowManager.addView(overlayView, params)
```

> 📌 **언제 WindowManager를 직접 쓰나?**
> 일반 앱에서는 거의 쓸 일이 없습니다. 주로 플로팅 버튼 앱, 화면 녹화 앱, 접근성 서비스처럼 **앱 바깥에 UI를 그려야 할 때** 사용합니다.

#### 4) Surface & SurfaceFlinger

```
Surface: Window가 실제로 픽셀을 그리는 버퍼 (Canvas의 목적지)
SurfaceFlinger: 여러 Window의 Surface를 합성해서 실제 디스플레이에 출력하는 시스템 프로세스

앱에서 View.draw(canvas) 호출
    └→ Canvas가 Surface 버퍼에 픽셀 기록
        └→ SurfaceFlinger가 모든 Surface를 합성
            └→ 디스플레이에 최종 출력 (60fps / 120fps)
```

---

### Window와 관련된 핵심 개념: WindowInsets

```
┌─────────────────────────────────────────┐  ← 물리적 화면 상단
│  상태바 (Status Bar)       ← insets 영역 │
├─────────────────────────────────────────┤
│                                         │
│         앱 콘텐츠 영역                   │
│                                         │
├─────────────────────────────────────────┤
│  내비게이션 바 (Nav Bar)   ← insets 영역 │
└─────────────────────────────────────────┘  ← 물리적 화면 하단
```

WindowInsets는 "시스템 UI가 차지하는 영역"에 대한 정보입니다.
앱은 이 정보를 받아서 콘텐츠가 시스템 UI에 가리지 않도록 padding을 조절해야 합니다.
이 내용은 다음 섹션(Edge-to-Edge)에서 자세히 다룹니다.

---

### Window 관련 자주 하는 실수

```kotlin
// ❌ 잘못된 예: Dialog에서 window가 null일 수 있음
dialog.window?.setBackgroundDrawable(ColorDrawable(Color.TRANSPARENT))

// ✅ 올바른 예: show() 이후에 window에 접근
dialog.show()
dialog.window?.setLayout(
    ViewGroup.LayoutParams.MATCH_PARENT,
    ViewGroup.LayoutParams.WRAP_CONTENT
)

// ❌ 잘못된 예: onCreate()에서 아직 Window가 완전히 준비되지 않은 상태에서 크기 측정
override fun onCreate(...) {
    val width = window.decorView.width // 항상 0 반환!
}

// ✅ 올바른 예: ViewTreeObserver 또는 doOnLayout 사용
window.decorView.doOnLayout {
    val width = it.width // 실제 크기가 측정된 이후
}
```

---

### 핵심 정리

- Android의 모든 UI는 **Window 위에 그려짐** (Activity, Dialog, PopupWindow 모두 각자의 Window를 가짐)
- 실제 구현체는 **PhoneWindow** - `setContentView()`로 우리가 View를 올리는 곳
- **WindowManager**: Window의 생성/삭제/업데이트 담당 시스템 서비스
- **Surface**: 실제 픽셀이 그려지는 버퍼, **SurfaceFlinger**가 합성해서 화면에 출력
- **WindowInsets**: 시스템 UI(상태바, 내비게이션 바, IME)가 차지하는 영역 정보

---

## 2. 📐 Edge-to-Edge & WindowInsets 처리

### 개념 설명

**Edge-to-Edge**란 앱 콘텐츠를 상태바·내비게이션 바 영역까지 확장해서 그리는 방식입니다.
Android 15(API 35)부터 **기본값으로 강제 적용**되므로, 지금 제대로 이해하지 않으면 레이아웃이 시스템 UI에 가리는 버그가 반드시 발생합니다.

```
기존 방식 (non Edge-to-Edge)     Edge-to-Edge
────────────────────────          ────────────────────────
[상태바]                          [상태바]  ← 앱 콘텐츠가 뒤에 깔림
────────────────────────          [앱 콘텐츠 시작]
[앱 콘텐츠 시작]                   │
│                                  │
│                                  │
[앱 콘텐츠 끝]                     [앱 콘텐츠 끝]
────────────────────────          [내비게이션 바] ← 앱 콘텐츠가 뒤에 깔림
[내비게이션 바]
```

---

### 구현 방법

#### Step 1. Edge-to-Edge 활성화

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 반드시 setContentView / setContent 이전에 호출
        // → 앱 콘텐츠가 상태바 + 내비게이션 바 영역까지 확장됨
        enableEdgeToEdge()

        setContent {
            MyAppTheme {
                MainScreen()
            }
        }
    }
}
```

> 📌 **`enableEdgeToEdge()`** 는 내부적으로 다음을 처리합니다:
> - 상태바 / 내비게이션 바 배경을 투명으로 설정
> - 시스템 UI 아이콘 색상 자동 조정 (라이트/다크 테마에 따라)
> - `WindowCompat.setDecorFitsSystemWindows(window, false)` 호출

---

#### Step 2. WindowInsets 타입 이해

```kotlin
// WindowInsetsCompat이 제공하는 주요 insets 타입
WindowInsetsCompat.Type.statusBars()         // 상태바 높이
WindowInsetsCompat.Type.navigationBars()     // 내비게이션 바 높이 (제스처/버튼 모두)
WindowInsetsCompat.Type.systemBars()         // 상태바 + 내비게이션 바 (가장 자주 씀)
WindowInsetsCompat.Type.ime()                // 소프트 키보드(IME) 높이
WindowInsetsCompat.Type.displayCutout()      // 노치/펀치홀 영역
WindowInsetsCompat.Type.systemGestures()     // 시스템 제스처 영역
```

---

#### Step 3. Compose에서 WindowInsets 처리

Compose는 `windowInsetsPadding` 계열 Modifier를 제공합니다:

```kotlin
@Composable
fun MainScreen() {
    Scaffold(
        // Scaffold가 알아서 systemBars insets를 처리해줌 (권장)
        // TopAppBar → 상단 insets 자동 처리
        // BottomBar → 하단 insets 자동 처리
        topBar = {
            TopAppBar(title = { Text("캘린더") })
        },
        bottomBar = {
            BottomNavigationBar()
        }
    ) { innerPadding ->
        // innerPadding: Scaffold가 계산한 콘텐츠 영역의 padding
        // 반드시 적용해야 TopBar/BottomBar에 가리지 않음
        CalendarContent(modifier = Modifier.padding(innerPadding))
    }
}
```

---

#### Step 4. Scaffold 없이 직접 Insets 처리

Scaffold를 쓰지 않는 커스텀 화면에서는 직접 처리합니다:

```kotlin
@Composable
fun FullScreenCalendar() {
    Box(
        modifier = Modifier
            .fillMaxSize()
            // systemBars: 상태바 + 내비게이션 바 모두 padding으로 확보
            .windowInsetsPadding(WindowInsets.systemBars)
    ) {
        CalendarContent()
    }
}

// 더 세밀한 제어가 필요할 때 - 각 방향별로 분리
@Composable
fun CalendarWithCustomInsets() {
    Column(
        modifier = Modifier
            .fillMaxSize()
            // 상단만: 상태바 높이만큼 padding
            .padding(top = WindowInsets.statusBars.asPaddingValues().calculateTopPadding())
    ) {
        CalendarHeader()

        LazyColumn(
            // 하단만: 내비게이션 바 높이만큼 contentPadding (스크롤 끝부분 여백)
            contentPadding = WindowInsets.navigationBars.asPaddingValues()
        ) {
            items(events) { event ->
                EventItem(event)
            }
        }
    }
}
```

---

#### Step 5. 키보드(IME) Insets 처리 - 일정 입력 화면에서 필수

```kotlin
@Composable
fun AddEventScreen() {
    // imePadding(): 키보드가 올라올 때 자동으로 하단 padding 추가
    // → 입력 필드가 키보드에 가리지 않음
    Column(
        modifier = Modifier
            .fillMaxSize()
            .windowInsetsPadding(WindowInsets.systemBars)
            .imePadding() // ← 키보드 높이만큼 자동으로 bottom padding
            .verticalScroll(rememberScrollState())
    ) {
        Text("일정 제목", style = MaterialTheme.typography.titleMedium)

        OutlinedTextField(
            value = title,
            onValueChange = { title = it },
            label = { Text("제목") },
            modifier = Modifier.fillMaxWidth()
        )

        OutlinedTextField(
            value = memo,
            onValueChange = { memo = it },
            label = { Text("메모") },
            modifier = Modifier.fillMaxWidth()
        )
    }
}
```

---

#### Step 6. 캘린더 앱 실전 예시 - 상단은 투명, 하단만 padding

```kotlin
@Composable
fun CalendarMainScreen() {
    Box(modifier = Modifier.fillMaxSize()) {
        // 배경 이미지나 색상이 상태바까지 꽉 채워지길 원할 때
        CalendarBackground(modifier = Modifier.fillMaxSize())

        Column(modifier = Modifier.fillMaxSize()) {
            // 상태바 높이만큼 Spacer → 콘텐츠가 상태바 아래부터 시작
            Spacer(
                modifier = Modifier.windowInsetsTopHeight(WindowInsets.statusBars)
            )

            // 캘린더 헤더 (월 표시, 이전/다음 버튼)
            CalendarHeader()

            // 캘린더 본문
            HorizontalPager(/* ... */) { page ->
                MonthCalendarGrid()
            }

            // 이벤트 목록
            LazyColumn(
                // 내비게이션 바 높이만큼 하단 여백 확보
                contentPadding = WindowInsets.navigationBars.asPaddingValues()
            ) {
                items(todayEvents) { event ->
                    EventItem(event)
                }
            }
        }
    }
}
```

---

### 자주 발생하는 버그와 해결법

```kotlin
// 🐛 버그: Edge-to-Edge 적용 후 내용이 내비게이션 바에 가림
// 원인: innerPadding을 적용하지 않음
Scaffold { _ ->  // innerPadding 무시!
    Content()    // ← 하단이 내비게이션 바에 가려짐
}

// ✅ 해결:
Scaffold { innerPadding ->
    Content(modifier = Modifier.padding(innerPadding))
}

// 🐛 버그: windowInsetsPadding과 padding을 이중으로 적용해 여백이 2배로 생김
Scaffold { innerPadding ->
    Content(
        modifier = Modifier
            .padding(innerPadding)          // Scaffold가 이미 insets 처리함
            .windowInsetsPadding(WindowInsets.systemBars) // ← 이중 적용! 삭제해야 함
    )
}
```

---

### 핵심 정리

- Android 15부터 Edge-to-Edge **강제 적용** → 지금 미리 대응해야 함
- `enableEdgeToEdge()`를 `setContent` **이전**에 호출
- `Scaffold` 사용 시 `innerPadding`을 **반드시** 콘텐츠에 적용
- `imePadding()`으로 키보드가 입력 필드를 가리는 문제 원클릭 해결
- `windowInsetsTopHeight` / `windowInsetsBottomHeight` Spacer로 세밀한 레이아웃 제어

---

## 3. 🔧 Compose Custom Layout

### 개념 설명

`Row`, `Column`, `Box` 같은 기본 레이아웃으로 해결할 수 없는 커스텀 배치가 필요할 때 사용합니다.
캘린더 앱에서는 **7열 그리드(요일)**, **날짜 셀의 이벤트 점 배치** 등에서 자주 필요합니다.

Compose의 레이아웃 시스템은 세 단계로 동작합니다:

```
1. Measure (측정): 각 자식이 얼마나 큰지 측정
2. Place (배치): 측정된 크기를 바탕으로 위치 결정
3. Draw (그리기): 배치된 위치에 실제로 그리기
```

---

### 구현 방법

#### Step 1. Layout 컴포저블 기본 구조

```kotlin
@Composable
fun CustomLayout(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Layout(
        content = content,
        modifier = modifier
    ) { measurables, constraints ->
        // measurables: content 안의 각 자식 컴포저블 목록
        // constraints: 이 Layout에게 부모가 허용한 크기 범위 (minWidth~maxWidth, minHeight~maxHeight)

        // Step 1: 각 자식 측정
        val placeables = measurables.map { measurable ->
            // measure(): 자식에게 constraints를 전달해서 크기를 측정하도록 요청
            measurable.measure(constraints)
        }

        // Step 2: Layout 전체 크기 결정
        val totalWidth = constraints.maxWidth
        val totalHeight = placeables.sumOf { it.height }

        // layout(): 최종 크기를 결정하고 배치 블록 진입
        layout(width = totalWidth, height = totalHeight) {
            // Step 3: 각 자식 위치 결정
            var yOffset = 0
            placeables.forEach { placeable ->
                // placeRelative(): LTR/RTL을 자동으로 고려한 배치 (권장)
                placeable.placeRelative(x = 0, y = yOffset)
                yOffset += placeable.height
            }
        }
    }
}
```

---

#### Step 2. 실전 예제 - 캘린더 7열 날짜 그리드

```kotlin
/**
 * 캘린더의 날짜를 7열로 배치하는 커스텀 레이아웃
 * Row + 7개 weight(1f) 와 유사하지만, 날짜 셀마다 정확히 동일한 너비를 보장합니다.
 */
@Composable
fun CalendarWeekRow(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit // 7개의 날짜 셀 컴포저블
) {
    Layout(
        content = content,
        modifier = modifier
    ) { measurables, constraints ->
        val columnCount = 7
        // 각 셀의 너비: 전체 너비를 7등분 (소수점 오차 없이 정확하게)
        val cellWidth = constraints.maxWidth / columnCount
        val cellConstraints = constraints.copy(
            minWidth = cellWidth,
            maxWidth = cellWidth
        )

        // 모든 셀을 동일한 너비 제약으로 측정
        val placeables = measurables.map { it.measure(cellConstraints) }

        // 행 높이: 가장 큰 셀 높이를 기준으로 (날짜 셀마다 이벤트 수가 달라 높이가 다를 수 있음)
        val rowHeight = placeables.maxOfOrNull { it.height } ?: 0

        layout(width = constraints.maxWidth, height = rowHeight) {
            placeables.forEachIndexed { index, placeable ->
                placeable.placeRelative(
                    x = index * cellWidth, // 7열 위치
                    y = 0
                )
            }
        }
    }
}

// 사용 예
@Composable
fun CalendarRow(week: List<LocalDate?>) {
    CalendarWeekRow(modifier = Modifier.fillMaxWidth()) {
        week.forEach { date ->
            DayCell(date = date)
        }
    }
}
```

---

#### Step 3. SubcomposeLayout - 측정 결과가 Composition 자체에 영향을 줄 때

일반 `Layout`과의 핵심 차이를 먼저 이해해야 합니다.

```
일반 Layout
  Composition (모든 자식 구성 완료) → Measure → Place
  ↑ 자식이 몇 개인지, 무슨 내용인지는 이미 Composition에서 확정됨

SubcomposeLayout
  Measure 단계 진입 → subcompose() 호출 → Composition → Measure → Place
  ↑ 측정 중간에 Composition을 실행할 수 있으므로,
    "측정 결과를 보고 나서 무엇을 Composition할지" 결정 가능
```

> 📌 일반 `Layout`도 자식마다 다른 constraints를 넘겨서 측정할 수 있습니다.
> `SubcomposeLayout`이 필요한 순간은 **측정값이 constraints가 아닌 Composition 자체(무엇을 그릴지)를 결정해야 할 때**입니다.

---

**실전 예제: 캘린더 날짜 셀에 이벤트 칩을 너비에 맞게 동적으로 배치**

셀 너비를 먼저 측정한 뒤, 그 너비에 실제로 들어갈 수 있는 개수만큼만 이벤트 칩을 Composition합니다. 남은 이벤트는 "+N" 오버플로우 칩으로 대체합니다.

```kotlin
/**
 * 사용 가능한 너비를 측정한 후, 그 너비에 맞는 개수만큼만 이벤트 칩을 compose함
 * → 측정 결과가 "몇 개의 칩을 compose할지"라는 Composition 결정에 영향을 줌
 * → 일반 Layout으로는 불가능: Composition이 먼저 끝나야 measurables를 받을 수 있기 때문
 */
@Composable
fun EventChipRow(
    events: List<Event>,
    modifier: Modifier = Modifier
) {
    SubcomposeLayout(modifier = modifier) { constraints ->
        val chipSpacing = 4.dp.roundToPx()

        // Phase 1: "+N" 오버플로우 칩 너비를 미리 측정
        // (공간이 부족할 때 마지막에 표시할 칩)
        val overflowPlaceable = subcompose("overflow") {
            OverflowChip(count = events.size) // 예: "+3"
        }.first().measure(constraints)

        // Phase 2: 이벤트 칩을 하나씩 compose & 측정하며 너비 초과 여부 확인
        // → 몇 개를 compose할지가 측정값(availableWidth)에 의해 결정됨
        var availableWidth = constraints.maxWidth
        val visibleChipPlaceables = mutableListOf<Placeable>()

        for (i in events.indices) {
            val chipPlaceable = subcompose("chip_$i") {
                EventChip(event = events[i])
            }.first().measure(constraints)

            // 다음 칩을 추가했을 때 오버플로우 칩이 들어갈 공간까지 고려
            val spaceNeeded = chipPlaceable.width + chipSpacing +
                if (i < events.lastIndex) overflowPlaceable.width else 0

            if (availableWidth >= spaceNeeded) {
                visibleChipPlaceables.add(chipPlaceable)
                availableWidth -= chipPlaceable.width + chipSpacing
            } else {
                break // 더 이상 들어갈 공간 없음 → 나머지는 오버플로우로 처리
            }
        }

        val showOverflow = visibleChipPlaceables.size < events.size
        val rowHeight = visibleChipPlaceables.maxOfOrNull { it.height }
            ?: overflowPlaceable.height

        layout(constraints.maxWidth, rowHeight) {
            var xOffset = 0

            visibleChipPlaceables.forEach { placeable ->
                placeable.placeRelative(xOffset, 0)
                xOffset += placeable.width + chipSpacing
            }

            // 공간이 부족해 잘린 이벤트가 있으면 "+N" 칩 표시
            if (showOverflow) {
                overflowPlaceable.placeRelative(xOffset, 0)
            }
        }
    }
}
```

---

**왜 일반 `Layout`으로는 이걸 못 하는가?**

```kotlin
// ❌ 일반 Layout 시도 - 불가능한 이유
Layout(
    content = {
        // Composition이 여기서 먼저 끝남
        // 이 시점에서 availableWidth를 모르므로 몇 개를 그려야 할지 결정 불가
        events.forEach { EventChip(it) } // 무조건 전부 compose해버림
    }
) { measurables, constraints ->
    // 측정은 여기서 하지만, 위에서 이미 모든 칩이 compose된 상태
    // → "너비에 맞는 개수만 compose" 자체가 불가능
}

// ✅ SubcomposeLayout - 가능한 이유
SubcomposeLayout { constraints ->
    // 이 시점에서 constraints(너비)를 알고 있음
    // → 루프 돌면서 너비 초과 시 중단 → 필요한 만큼만 subcompose() 호출
}
```

> 📌 **`BoxWithConstraints`** 도 내부적으로 `SubcomposeLayout`을 사용합니다.
> constraints가 확정된 이후 content를 compose하므로, content 안에서 `maxWidth`/`maxHeight`를 바로 참조할 수 있습니다:
> ```kotlin
> BoxWithConstraints {
>     // maxWidth는 이미 실제 측정된 값
>     if (maxWidth > 600.dp) TwoColumnCalendar() else SingleColumnCalendar()
> }
> ```

---

#### Step 4. 커스텀 레이아웃 선택 기준

```
기본 레이아웃 (Row, Column, Box)
    → 일반적인 경우 항상 먼저 시도
    → weight(), Arrangement 등으로 대부분 해결 가능

Layout 컴포저블
    → 기본 레이아웃으로 표현할 수 없는 독창적인 배치가 필요할 때
    → 예: 7열 날짜 그리드, 방사형 배치, 겹침 효과

SubcomposeLayout
    → 측정 결과가 "무엇을 Composition할지" 자체를 결정해야 할 때
    → 예: 너비에 따라 표시할 이벤트 칩 개수를 동적으로 결정
    → 예: BoxWithConstraints처럼 constraints를 Composition 안에서 참조해야 할 때
    → ⚠️ Composition을 Measure 단계 안에서 실행하므로 성능 비용이 있음. 꼭 필요할 때만 사용
```

---

### 핵심 정리

- `Layout`: `measurables`(측정할 자식들) → `measure()` → `layout()` → `placeRelative()` 순서
- `cellWidth = constraints.maxWidth / columnCount` 패턴으로 정확한 등분 레이아웃 구현
- `SubcomposeLayout`: 측정 결과가 **Composition 자체(무엇을 그릴지)** 에 영향을 줄 때 사용. constraints 변경은 일반 `Layout`으로도 가능
- `placeRelative()` 사용 권장 → RTL 언어 자동 대응

---

## 📝 오늘의 학습 요약

| 주제 | 핵심 개념 | 기억할 포인트 |
|---|---|---|
| Android Window | PhoneWindow, WindowManager, Surface | 모든 UI는 Window 위에 그려짐. DecorView → ContentView 구조 |
| Edge-to-Edge & Insets | `enableEdgeToEdge()`, `WindowInsets`, `imePadding()` | Android 15 강제 적용. `innerPadding` 반드시 콘텐츠에 전달 |
| Compose Custom Layout | `Layout`, `SubcomposeLayout`, `placeRelative()` | Measure → Layout → Place 3단계. constraints 변경은 `Layout`으로 충분. `SubcomposeLayout`은 측정값이 Composition 결정에 영향줄 때만 |

---

## 🔗 참고 자료

- [Android Window 공식 문서](https://developer.android.com/reference/android/view/Window)
- [Edge-to-Edge 공식 가이드](https://developer.android.com/develop/ui/views/layout/edge-to-edge)
- [WindowInsets in Compose](https://developer.android.com/develop/ui/compose/layouts/insets)
- [Compose Custom Layout 공식 문서](https://developer.android.com/develop/ui/compose/layouts/custom)
- [Android 15 Edge-to-Edge 강제 적용 변경사항](https://developer.android.com/about/versions/15/behavior-changes-15#edge-to-edge)
