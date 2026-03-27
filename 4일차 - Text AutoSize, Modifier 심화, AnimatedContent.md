# 4일차 - Text AutoSize, Modifier 심화(graphicsLayer/drawBehind), AnimatedContent

> 📅 2026-03-25 | Android UI 매일 3개 챌린지 Day 4

---

## 📌 오늘 배울 것

| # | 주제 | 난이도 | 핵심 기술 |
|---|---|---|---|
| 1 | Text AutoSize | ⭐⭐⭐ | onTextLayout, BasicText autoSize, BoxWithConstraints |
| 2 | Modifier 심화 - graphicsLayer & drawBehind | ⭐⭐⭐ | graphicsLayer, drawBehind, drawWithContent |
| 3 | AnimatedContent & ContentTransform | ⭐⭐⭐ | AnimatedContent, ContentTransform, SizeTransform |

> 💡 **캘린더 앱 직결 포인트: 날짜 숫자 자동 크기 조절, 셀 커스텀 그래픽, 월 전환 애니메이션**

---

## 1. 🔠 Text AutoSize

### 개념 설명

**AutoSize**는 텍스트가 주어진 공간에 맞게 폰트 크기를 자동으로 줄이거나 늘리는 기능입니다.
캘린더 앱에서는 날짜 셀 안에 일정 제목을 표시할 때, 긴 제목도 셀을 벗어나지 않고 보여줘야 하는 상황에서 유용합니다.

Compose에서 AutoSize를 구현하는 방법은 두 가지입니다:
- **방법 A**: `onTextLayout` 콜백으로 직접 구현 (모든 Compose 버전 호환)
- **방법 B**: `BasicText`의 `autoSize` 파라미터 사용 (Compose Foundation 1.7.0+)

---

### 방법 A. onTextLayout 콜백으로 직접 구현

```kotlin
/**
 * 텍스트가 주어진 공간에 맞게 폰트 크기를 자동으로 줄이는 컴포저블
 *
 * 동작 원리:
 * 1. maxFontSize로 텍스트를 그려봄
 * 2. onTextLayout에서 overflow 여부 확인
 * 3. overflow 발생 시 fontSize를 줄임 → 재구성(recomposition) → 2번 반복
 * 4. overflow가 없거나 minFontSize에 도달하면 종료
 */
@Composable
fun AutoSizeText(
    text: String,
    modifier: Modifier = Modifier,
    minFontSize: TextUnit = 10.sp,
    maxFontSize: TextUnit = 32.sp,
    maxLines: Int = 1,
    style: TextStyle = LocalTextStyle.current,
    color: Color = Color.Unspecified
) {
    // 현재 적용 중인 폰트 크기 상태. 항상 maxFontSize에서 시작해서 줄여나감
    var fontSize by remember(text) { mutableStateOf(maxFontSize) }
    // overflow 발생 여부 추적 (onTextLayout 콜백에서 업데이트)
    var hasOverflow by remember(text) { mutableStateOf(false) }

    Text(
        text = text,
        modifier = modifier,
        style = style.copy(fontSize = fontSize),
        color = color,
        maxLines = maxLines,
        overflow = TextOverflow.Visible, // ← Clip하지 않아야 overflow 감지 가능
        softWrap = maxLines > 1,
        onTextLayout = { textLayoutResult ->
            // hasVisualOverflow: 텍스트가 주어진 공간을 벗어났는지 여부
            if (textLayoutResult.hasVisualOverflow) {
                // 폰트를 10% 줄임, 단 minFontSize 이하로는 내려가지 않음
                val reduced = fontSize * 0.9f
                fontSize = if (reduced > minFontSize) reduced else minFontSize
            }
        }
    )
}
```

```kotlin
// 사용 예 - 캘린더 일정 제목
@Composable
fun EventTitle(title: String) {
    AutoSizeText(
        text = title,
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 4.dp),
        minFontSize = 8.sp,
        maxFontSize = 14.sp,
        maxLines = 1,
        style = MaterialTheme.typography.labelSmall,
        color = MaterialTheme.colorScheme.onPrimaryContainer
    )
}
```

> ⚠️ **주의**: `onTextLayout` 방식은 overflow가 해소될 때까지 recomposition이 반복됩니다.
> `minFontSize`를 반드시 지정해 무한 루프를 방지하세요.

---

### 방법 B. BasicText autoSize 파라미터 (Compose Foundation 1.7.0+)

Compose Foundation 1.7.0부터 `BasicText`에 `autoSize` 파라미터가 공식 추가됐습니다.
내부적으로 이진 탐색(binary search)으로 최적 크기를 한 번에 찾으므로 **방법 A보다 효율적**입니다.

```kotlin
dependencies {
    // 1.7.0 이상 필요
    implementation("androidx.compose.foundation:foundation:1.7.0")
}
```

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun AutoSizeEventTitle(title: String, modifier: Modifier = Modifier) {
    BasicText(
        text = title,
        modifier = modifier,
        style = MaterialTheme.typography.bodyMedium.copy(
            color = MaterialTheme.colorScheme.onSurface
        ),
        maxLines = 1,
        // TextAutoSize.StepBased: minFontSize ~ maxFontSize 사이에서 stepSize 단위로 탐색
        autoSize = TextAutoSize.StepBased(
            minFontSize = 8.sp,
            maxFontSize = 14.sp,
            stepSize = 1.sp   // 1sp 단위로 탐색 (작을수록 정밀하지만 탐색 횟수 증가)
        )
    )
}
```

> 📌 **방법 A vs 방법 B 비교**

| | 방법 A (onTextLayout) | 방법 B (BasicText autoSize) |
|---|---|---|
| 최소 버전 | 모든 Compose 버전 | Foundation 1.7.0+ |
| 탐색 방식 | 점진적 축소 (recomposition 반복) | 이진 탐색 (1회 탐색) |
| 성능 | 상대적으로 낮음 | 상대적으로 높음 |
| Material3 Text 지원 | ✅ | ❌ (BasicText만 지원) |
| 커스터마이징 | 자유로움 | stepSize 단위 제한 |

---

### BoxWithConstraints 방식 - 다국어 대응이 필요할 때

텍스트 길이가 언어마다 크게 다른 경우, 컨테이너 너비 기준으로 폰트 크기를 비율로 계산합니다:

```kotlin
@Composable
fun ResponsiveDateText(date: Int, modifier: Modifier = Modifier) {
    BoxWithConstraints(modifier = modifier) {
        // maxWidth를 기준으로 폰트 크기를 비율로 계산
        // 예: 셀 너비의 40%를 폰트 크기로 사용
        val fontSize = (maxWidth.value * 0.4f).sp

        Text(
            text = date.toString(),
            fontSize = fontSize.coerceIn(10.sp, 28.sp), // 최소/최대 범위 제한
            textAlign = TextAlign.Center,
            modifier = Modifier.align(Alignment.Center)
        )
    }
}
```

---

### 핵심 정리

- **방법 A** (`onTextLayout`): 모든 버전 호환, Material3 `Text`에서 사용 가능. overflow 시 recomposition 반복이 단점
- **방법 B** (`BasicText autoSize`): Foundation 1.7.0+, 이진 탐색으로 효율적. `BasicText`에서만 사용 가능
- **BoxWithConstraints**: 컨테이너 크기를 비율로 계산할 때 유용. 다국어 대응에 적합
- `minFontSize`는 반드시 지정 → 무한 recomposition 방지

---

## 2. 🎨 Modifier 심화 - graphicsLayer & drawBehind

### 개념 설명

`Modifier.graphicsLayer`와 `Modifier.drawBehind`는 Compose에서 **렌더링 레벨의 시각 효과**를 줄 때 사용합니다.
`alpha`, `scale`, `clip`처럼 단순한 Modifier로는 표현할 수 없는 고급 효과를 구현할 수 있습니다.

---

### graphicsLayer - 하드웨어 가속 시각 변환

`graphicsLayer`는 해당 Composable을 **별도의 레이어(RenderNode)** 로 분리해 렌더링합니다.
이 레이어에서 변환(이동/회전/확대), 투명도, 블러, 클리핑 등을 GPU에서 직접 처리합니다.

```kotlin
@Composable
fun CalendarDayCell(
    date: LocalDate,
    isToday: Boolean,
    isSelected: Boolean,
    modifier: Modifier = Modifier
) {
    Box(
        modifier = modifier
            .graphicsLayer {
                // 선택된 날짜는 살짝 확대 + 그림자 효과
                if (isSelected) {
                    scaleX = 1.1f
                    scaleY = 1.1f
                    shadowElevation = 8f     // 그림자 깊이 (px)
                    shape = CircleShape      // 그림자 모양
                    clip = true              // shape으로 클리핑
                }

                // 오늘이 아닌 날짜는 살짝 투명하게
                alpha = if (isToday || isSelected) 1f else 0.7f

                // 다른 달 날짜는 더 투명하게 + 살짝 축소
                // (필요에 따라 조건 추가)
            }
            .size(40.dp),
        contentAlignment = Alignment.Center
    ) {
        Text(text = date.dayOfMonth.toString())
    }
}
```

---

**graphicsLayer vs Modifier.alpha / Modifier.scale 비교:**

```kotlin
// ❌ 이렇게 하면 매 recomposition마다 전체 UI 트리 재렌더링
Box(modifier = Modifier.alpha(animatedAlpha))

// ✅ graphicsLayer는 레이어 속성만 변경 → 자식 UI 재측정 없이 GPU에서 직접 처리
// 애니메이션처럼 값이 빠르게 변할 때 훨씬 효율적
Box(modifier = Modifier.graphicsLayer { alpha = animatedAlpha })
```

> 📌 **언제 graphicsLayer를 써야 하나?**
> 애니메이션처럼 **값이 프레임마다 바뀌는 상황**에서 `alpha`, `scale`, `rotation`을 적용할 때 사용하세요.
> 정적인 값이라면 `Modifier.alpha()`, `Modifier.scale()` 등 개별 Modifier로 충분합니다.

---

**renderEffect로 블러 효과 (Android 12+):**

```kotlin
@Composable
fun BlurredEventBackground(modifier: Modifier = Modifier) {
    Box(
        modifier = modifier
            .graphicsLayer {
                // BlurMaskFilter를 GPU에서 직접 적용 (Android 12+)
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
                    renderEffect = BlurEffect(
                        radiusX = 10f,
                        radiusY = 10f,
                        edgeTreatment = TileMode.Decal
                    ).asComposeRenderEffect()
                }
            }
            .background(MaterialTheme.colorScheme.primaryContainer)
    )
}
```

---

### drawBehind - Composable 뒤에 직접 그리기

`drawBehind`는 해당 Composable의 **뒤쪽(배경 레이어)** 에 Canvas로 직접 그림을 그립니다.
`background(color)`보다 훨씬 자유롭게 배경을 커스터마이징할 수 있습니다.

```kotlin
// 캘린더 날짜 셀 - 오늘 날짜 표시를 원형으로 강조
@Composable
fun TodayHighlightCell(
    date: Int,
    isToday: Boolean,
    modifier: Modifier = Modifier
) {
    val primaryColor = MaterialTheme.colorScheme.primary

    Text(
        text = date.toString(),
        textAlign = TextAlign.Center,
        modifier = modifier
            .padding(4.dp)
            .drawBehind {
                if (isToday) {
                    // Composable 뒤에 원형 배경을 직접 그림
                    // size: 현재 Composable의 크기 (DrawScope가 제공)
                    val radius = minOf(size.width, size.height) / 2f
                    drawCircle(
                        color = primaryColor,
                        radius = radius,
                        center = Offset(size.width / 2f, size.height / 2f)
                    )
                }
            }
    )
}
```

---

### drawWithContent - 기존 내용 위/아래에 추가로 그리기

`drawBehind`는 Composable 뒤에만 그릴 수 있지만, `drawWithContent`는 **그리는 순서를 직접 제어**할 수 있습니다.

```kotlin
// 이벤트 있는 날짜: 날짜 숫자 아래에 색상 점을 그림
@Composable
fun DateCellWithDot(
    date: Int,
    eventColor: Color?,
    modifier: Modifier = Modifier
) {
    Text(
        text = date.toString(),
        textAlign = TextAlign.Center,
        modifier = modifier
            .padding(4.dp)
            .drawWithContent {
                // 1. 기존 Text 내용을 먼저 그림
                drawContent()

                // 2. 그 위에 (실제로는 아래쪽 좌표에) 이벤트 점을 추가로 그림
                if (eventColor != null) {
                    drawCircle(
                        color = eventColor,
                        radius = 3.dp.toPx(),
                        center = Offset(
                            x = size.width / 2f,
                            y = size.height - 4.dp.toPx() // 텍스트 아래
                        )
                    )
                }
            }
    )
}
```

---

### 핵심 정리

- `graphicsLayer`: 별도 RenderNode 생성 → GPU 직접 처리. 애니메이션처럼 값이 빠르게 변할 때 효율적
- `renderEffect`: 블러 등 고급 그래픽 효과 (Android 12+)
- `drawBehind`: Composable 뒤쪽에 Canvas로 자유롭게 배경 그리기
- `drawWithContent`: `drawContent()` 호출 위치로 기존 콘텐츠와 커스텀 그래픽의 Z순서 제어
- 정적 값: 개별 Modifier(`alpha()`, `scale()`) / 애니메이션 값: `graphicsLayer` 블록

---

## 3. 🔄 AnimatedContent & ContentTransform

### 개념 설명

`AnimatedContent`는 **상태 변화에 따라 콘텐츠 자체가 바뀔 때** 전환 애니메이션을 적용합니다.
단순한 show/hide 애니메이션인 `AnimatedVisibility`와 달리, **"이전 콘텐츠가 나가고 새 콘텐츠가 들어오는"** 전환을 처리합니다.

캘린더 앱에서는 월이 바뀔 때 이전 달이 슬라이드 아웃되고 새 달이 슬라이드 인되는 효과에 활용할 수 있습니다.

---

### 구현 방법

#### Step 1. AnimatedContent 기본 구조

```kotlin
@Composable
fun MonthTitle(currentYearMonth: YearMonth) {
    // targetState가 바뀔 때마다 전환 애니메이션 실행
    AnimatedContent(
        targetState = currentYearMonth,
        label = "month_title_animation"
    ) { yearMonth ->
        // 이 블록은 targetState가 바뀔 때마다 새로 실행됨
        // yearMonth: 현재 애니메이션 중인 YearMonth (이전 값 또는 새 값)
        Text(
            text = yearMonth.format(DateTimeFormatter.ofPattern("yyyy년 M월")),
            style = MaterialTheme.typography.titleLarge
        )
    }
}
```

---

#### Step 2. ContentTransform으로 전환 방향 제어

`ContentTransform`은 **들어오는 애니메이션(enter)** + **나가는 애니메이션(exit)** 의 조합입니다.
캘린더에서 다음 달로 넘어가면 오른쪽으로, 이전 달로 가면 왼쪽으로 슬라이드하는 효과를 만들 수 있습니다.

```kotlin
@Composable
fun CalendarMonthTitle(
    currentYearMonth: YearMonth,
    previousYearMonth: YearMonth
) {
    // 이전 달 대비 현재 달이 앞인지 뒤인지에 따라 방향 결정
    val isForward = currentYearMonth > previousYearMonth

    AnimatedContent(
        targetState = currentYearMonth,
        transitionSpec = {
            // ContentTransform = EnterTransition + ExitTransition (+ 선택적 SizeTransform)
            if (isForward) {
                // 다음 달: 오른쪽에서 들어오고, 왼쪽으로 나감
                ContentTransform(
                    targetContentEnter = slideInHorizontally { it } + fadeIn(),
                    initialContentExit = slideOutHorizontally { -it } + fadeOut()
                )
            } else {
                // 이전 달: 왼쪽에서 들어오고, 오른쪽으로 나감
                ContentTransform(
                    targetContentEnter = slideInHorizontally { -it } + fadeIn(),
                    initialContentExit = slideOutHorizontally { it } + fadeOut()
                )
            }
        },
        label = "calendar_month_title"
    ) { yearMonth ->
        Text(
            text = yearMonth.format(DateTimeFormatter.ofPattern("yyyy년 M월")),
            style = MaterialTheme.typography.titleLarge,
            modifier = Modifier.fillMaxWidth(),
            textAlign = TextAlign.Center
        )
    }
}
```

---

#### Step 3. transitionSpec 내부 축약 문법

```kotlin
AnimatedContent(
    targetState = currentMonth,
    transitionSpec = {
        // initialState: 이전 상태, targetState: 새 상태
        // using() 중위 함수로 ContentTransform을 더 간결하게 표현 가능
        val forward = targetState > initialState

        (slideInHorizontally { if (forward) it else -it } + fadeIn())
            .togetherWith(slideOutHorizontally { if (forward) -it else it } + fadeOut())
            // SizeTransform: 콘텐츠 크기가 달라질 때 크기 변화도 애니메이션
            .using(SizeTransform(clip = false))
    },
    label = "month_transition"
) { yearMonth ->
    MonthCalendarGrid(yearMonth = yearMonth)
}
```

> 📌 **`SizeTransform(clip = false)`**
> 전환 중에 이전/새 콘텐츠가 크기가 다를 때, 크기 변화 자체도 애니메이션으로 처리합니다.
> `clip = false`로 설정하면 전환 중 콘텐츠가 경계 밖으로 나가도 잘리지 않습니다.

---

#### Step 4. AnimatedContent vs AnimatedVisibility 선택 기준

```
AnimatedVisibility
    → 하나의 콘텐츠를 보이게/숨기게 할 때
    → 예: FAB 표시/숨김, 로딩 인디케이터 등장/퇴장

AnimatedContent
    → 상태에 따라 완전히 다른 콘텐츠로 교체될 때
    → 예: 로딩 중 → 실제 데이터, 이전 달 → 다음 달, 탭 전환
```

```kotlin
// 로딩 → 데이터 전환 예시
@Composable
fun CalendarContent(uiState: CalendarUiState) {
    AnimatedContent(
        targetState = uiState,
        transitionSpec = { fadeIn() togetherWith fadeOut() },
        label = "calendar_content"
    ) { state ->
        when (state) {
            is CalendarUiState.Loading -> {
                MonthShimmerSkeleton()
            }
            is CalendarUiState.Success -> {
                MonthCalendarGrid(events = state.events)
            }
            is CalendarUiState.Error -> {
                ErrorView(message = state.message)
            }
        }
    }
}
```

---

#### Step 5. animateContentSize - 크기 변화만 애니메이션

콘텐츠 교체 없이 **같은 콘텐츠의 크기만 부드럽게 바뀌어야 할 때**는 더 가벼운 `animateContentSize`를 사용합니다:

```kotlin
// 일정 카드 - 클릭 시 상세 내용 펼치기/접기
@Composable
fun ExpandableEventCard(event: Event) {
    var isExpanded by remember { mutableStateOf(false) }

    Card(
        modifier = Modifier
            .fillMaxWidth()
            .clickable { isExpanded = !isExpanded }
            // 높이 변화를 부드럽게 애니메이션 (AnimatedContent보다 가벼움)
            .animateContentSize(
                animationSpec = spring(
                    dampingRatio = Spring.DampingRatioMediumBouncy,
                    stiffness = Spring.StiffnessMedium
                )
            )
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(text = event.title, style = MaterialTheme.typography.titleMedium)

            // isExpanded가 바뀌면 이 Column의 높이가 변하고,
            // animateContentSize가 그 변화를 부드럽게 처리
            if (isExpanded) {
                Spacer(modifier = Modifier.height(8.dp))
                Text(text = event.description, style = MaterialTheme.typography.bodyMedium)
                Text(text = event.location, style = MaterialTheme.typography.bodySmall)
            }
        }
    }
}
```

---

### 핵심 정리

- `AnimatedContent(targetState)`: 상태가 바뀔 때 이전 → 새 콘텐츠 전환 애니메이션
- `ContentTransform` = `EnterTransition + ExitTransition`: `togetherWith` 중위 함수로 조합
- `initialState / targetState`: transitionSpec 블록 안에서 이전/새 상태를 참조해 **방향 제어**
- `SizeTransform(clip = false)`: 전환 중 크기 변화 애니메이션 + 경계 클리핑 방지
- `animateContentSize`: 콘텐츠 교체 없이 크기 변화만 애니메이션할 때 더 가벼운 선택

---

## 📝 오늘의 학습 요약

| 주제 | 핵심 API | 기억할 포인트 |
|---|---|---|
| Text AutoSize | `onTextLayout`, `BasicText(autoSize=)` | 방법 A: 모든 버전 호환 / 방법 B: Foundation 1.7+, 이진탐색으로 효율적 |
| graphicsLayer & drawBehind | `graphicsLayer`, `drawBehind`, `drawWithContent` | 애니메이션 값은 `graphicsLayer`로. `drawBehind`로 원형 오늘 날짜 강조 |
| AnimatedContent | `ContentTransform`, `togetherWith`, `SizeTransform` | `initialState > targetState`로 방향 판별 → 슬라이드 방향 제어 |

---

## 🔗 참고 자료

- [Compose Text 공식 문서](https://developer.android.com/develop/ui/compose/text/style-text)
- [BasicText autoSize - Foundation 1.7 릴리즈 노트](https://developer.android.com/jetpack/androidx/releases/compose-foundation)
- [graphicsLayer 공식 문서](https://developer.android.com/develop/ui/compose/graphics/draw/modifiers)
- [AnimatedContent 공식 문서](https://developer.android.com/develop/ui/compose/animation/composables-modifiers#animatedcontent)
- [Compose Animation Codelab](https://developer.android.com/codelabs/jetpack-compose-animation)
