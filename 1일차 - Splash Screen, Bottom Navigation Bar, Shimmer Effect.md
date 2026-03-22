# 1일차 - Animated Splash Screen, Bottom Navigation Bar, Shimmer Loading Effect

> 📅 2026-03-22 | Android UI 매일 3개 챌린지 Day 1

---

## 📌 오늘 배울 것

| # | UI 컴포넌트 | 난이도 | 핵심 기술 |
|---|---|---|---|
| 1 | Animated Splash Screen | ⭐⭐ | SplashScreen API, AnimatedVisibility |
| 2 | Bottom Navigation Bar | ⭐⭐ | Navigation Compose, BottomNavigation |
| 3 | Shimmer Loading Effect | ⭐⭐⭐ | Canvas, InfiniteTransition, Brush |

---

## 1. 🎬 Animated Splash Screen

### 개념 설명

Splash Screen은 앱이 초기화되는 동안 보여주는 시작 화면입니다.
Android 12(API 31)부터 **SplashScreen API**가 공식 도입되어, 이전처럼 별도 Activity를 만들 필요 없이 시스템 레벨에서 처리할 수 있습니다.

### 구현 방법

#### Step 1. 의존성 추가

`build.gradle.kts (app)` 에 다음을 추가합니다:

```kotlin
dependencies {
    implementation("androidx.core:core-splashscreen:1.0.1")
}
```

> 📌 **왜 필요한가?**
> `core-splashscreen` 라이브러리는 Android 12 미만 기기에서도 동일한 Splash 경험을 제공하기 위한 **하위 호환성 라이브러리**입니다.

---

#### Step 2. 테마 설정

`res/values/themes.xml`:

```xml
<!-- Splash 전용 테마: 앱이 실행되자마자 이 테마가 적용됩니다 -->
<style name="Theme.App.SplashScreen" parent="Theme.SplashScreen">
    <!-- 가운데 보여질 아이콘 -->
    <item name="windowSplashScreenAnimatedIcon">@drawable/ic_launcher_foreground</item>
    <!-- 아이콘 배경 색상 (선택사항) -->
    <item name="windowSplashScreenIconBackgroundColor">@color/splash_icon_bg</item>
    <!-- Splash 배경 색상 -->
    <item name="windowSplashScreenBackground">@color/splash_background</item>
    <!-- Splash가 끝난 후 적용될 앱 기본 테마 -->
    <item name="postSplashScreenTheme">@style/Theme.MyApp</item>
</style>
```

`AndroidManifest.xml`:

```xml
<activity
    android:name=".MainActivity"
    android:theme="@style/Theme.App.SplashScreen"> <!-- Splash 테마 적용 -->
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

---

#### Step 3. MainActivity에서 Splash 설치

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        // ⚠️ super.onCreate() 보다 먼저 호출해야 합니다!
        val splashScreen = installSplashScreen()

        super.onCreate(savedInstanceState)

        // 초기 데이터 로딩이 완료될 때까지 Splash를 유지하는 조건 설정
        // keepOnScreen이 true인 동안 Splash가 화면에 유지됩니다
        splashScreen.setKeepOnScreenCondition {
            // 예: ViewModel의 로딩 상태가 완료되면 false 반환 → Splash 종료
            false // 여기서는 즉시 종료
        }

        // Splash가 사라질 때 커스텀 애니메이션 적용
        splashScreen.setOnExitAnimationListener { splashScreenView ->
            // 아래로 슬라이드 아웃하는 애니메이션
            val slideUp = ObjectAnimator.ofFloat(
                splashScreenView.view,
                View.TRANSLATION_Y,
                0f,
                -splashScreenView.view.height.toFloat()
            )
            slideUp.duration = 500L
            slideUp.doOnEnd { splashScreenView.remove() } // 애니메이션 완료 후 Splash 제거
            slideUp.start()
        }

        setContent {
            MyAppTheme {
                MainScreen()
            }
        }
    }
}
```

> 📌 **setKeepOnScreenCondition 활용 팁**
> ViewModel과 함께 사용하면 강력합니다:
> ```kotlin
> val viewModel: MainViewModel by viewModels()
> splashScreen.setKeepOnScreenCondition { !viewModel.isReady.value }
> ```
> 데이터 로딩이 끝나야 Splash가 사라지므로, 빈 화면이 잠깐 보이는 문제를 방지할 수 있습니다.

---

#### Step 4. Compose로 Splash 후 진입 화면 페이드인 효과 추가

```kotlin
@Composable
fun MainScreen() {
    // 앱 진입 시 페이드인 효과를 위한 알파 애니메이션 상태
    var visible by remember { mutableStateOf(false) }

    // 컴포저블이 화면에 나타나자마자 visible을 true로 변경
    LaunchedEffect(Unit) {
        visible = true
    }

    // AnimatedVisibility: visible 상태에 따라 부드럽게 등장/퇴장 처리
    AnimatedVisibility(
        visible = visible,
        enter = fadeIn(animationSpec = tween(durationMillis = 800)) // 0.8초 페이드인
    ) {
        // 실제 메인 화면 콘텐츠
        HomeContent()
    }
}
```

---

### 핵심 정리

- `installSplashScreen()`은 반드시 `super.onCreate()` **이전**에 호출
- `setKeepOnScreenCondition`으로 데이터 로딩 완료 시점까지 Splash 유지 가능
- `setOnExitAnimationListener`로 종료 애니메이션 커스터마이징
- Android 12 이상에서는 아이콘 애니메이션(Animated Vector Drawable)도 지원

---

## 2. 🧭 Bottom Navigation Bar

### 개념 설명

앱 하단에 위치한 주요 탭 네비게이션입니다.
Material 3의 `NavigationBar`와 Jetpack Compose Navigation을 함께 사용하는 것이 현재 권장 방식입니다.

### 구현 방법

#### Step 1. 의존성 추가

```kotlin
dependencies {
    implementation("androidx.navigation:navigation-compose:2.7.7")
}
```

---

#### Step 2. 탭 항목 데이터 정의

```kotlin
// 탭 항목을 sealed class로 정의: 타입 안전성 확보
sealed class BottomNavItem(
    val route: String,        // 네비게이션 라우트 (화면 식별자)
    val label: String,        // 탭 라벨 텍스트
    val selectedIcon: ImageVector,   // 선택됐을 때 아이콘
    val unselectedIcon: ImageVector  // 선택 안 됐을 때 아이콘
) {
    data object Home : BottomNavItem(
        route = "home",
        label = "홈",
        selectedIcon = Icons.Filled.Home,
        unselectedIcon = Icons.Outlined.Home
    )
    data object Calendar : BottomNavItem(
        route = "calendar",
        label = "캘린더",
        selectedIcon = Icons.Filled.CalendarMonth,
        unselectedIcon = Icons.Outlined.CalendarMonth
    )
    data object Profile : BottomNavItem(
        route = "profile",
        label = "프로필",
        selectedIcon = Icons.Filled.Person,
        unselectedIcon = Icons.Outlined.Person
    )
}
```

---

#### Step 3. BottomNavigationBar 컴포저블 구현

```kotlin
@Composable
fun BottomNavigationBar(navController: NavController) {
    // 탭 항목 목록
    val items = listOf(
        BottomNavItem.Home,
        BottomNavItem.Calendar,
        BottomNavItem.Profile
    )

    // 현재 백스택의 엔트리를 State로 관찰
    // navController.currentBackStackEntryAsState(): 현재 화면이 바뀔 때마다 recomposition 트리거
    val navBackStackEntry by navController.currentBackStackEntryAsState()
    val currentRoute = navBackStackEntry?.destination?.route

    NavigationBar(
        containerColor = MaterialTheme.colorScheme.surface,
        tonalElevation = 8.dp
    ) {
        items.forEach { item ->
            val isSelected = currentRoute == item.route

            NavigationBarItem(
                selected = isSelected,
                onClick = {
                    navController.navigate(item.route) {
                        // 백스택에 동일한 화면이 중복으로 쌓이지 않도록
                        popUpTo(navController.graph.findStartDestination().id) {
                            saveState = true // 이전 탭 상태 저장
                        }
                        launchSingleTop = true  // 동일 탭 재클릭 시 새 인스턴스 생성 방지
                        restoreState = true     // 저장된 탭 상태 복원
                    }
                },
                icon = {
                    // 선택 여부에 따라 아이콘 전환
                    Icon(
                        imageVector = if (isSelected) item.selectedIcon else item.unselectedIcon,
                        contentDescription = item.label
                    )
                },
                label = { Text(text = item.label) },
                colors = NavigationBarItemDefaults.colors(
                    selectedIconColor = MaterialTheme.colorScheme.primary,
                    selectedTextColor = MaterialTheme.colorScheme.primary,
                    indicatorColor = MaterialTheme.colorScheme.primaryContainer
                )
            )
        }
    }
}
```

---

#### Step 4. Scaffold에 통합

```kotlin
@Composable
fun AppScaffold() {
    // NavController: 화면 간 이동을 관리하는 핵심 객체
    val navController = rememberNavController()

    Scaffold(
        bottomBar = {
            BottomNavigationBar(navController = navController)
        }
    ) { innerPadding ->
        // NavHost: 라우트와 실제 화면을 연결하는 컨테이너
        NavHost(
            navController = navController,
            startDestination = BottomNavItem.Home.route,
            modifier = Modifier.padding(innerPadding)
        ) {
            composable(BottomNavItem.Home.route) { HomeScreen() }
            composable(BottomNavItem.Calendar.route) { CalendarScreen() }
            composable(BottomNavItem.Profile.route) { ProfileScreen() }
        }
    }
}
```

---

### 탭 뱃지(Badge) 추가하기

알림 개수 표시 등에 활용됩니다:

```kotlin
NavigationBarItem(
    icon = {
        BadgedBox(
            badge = {
                // unreadCount가 0보다 크면 숫자 뱃지, 아니면 점(dot) 뱃지
                if (unreadCount > 0) {
                    Badge { Text(text = unreadCount.toString()) }
                }
            }
        ) {
            Icon(imageVector = item.selectedIcon, contentDescription = null)
        }
    },
    // ...
)
```

---

### 핵심 정리

- `rememberNavController()`로 NavController 인스턴스 생성
- `currentBackStackEntryAsState()`로 현재 탭 상태를 반응형으로 감지
- `launchSingleTop + saveState + restoreState` 조합으로 탭 전환 UX 최적화
- `BadgedBox`로 뱃지(알림 수) 표시 가능

---

## 3. ✨ Shimmer Loading Effect

### 개념 설명

데이터를 불러오는 동안 실제 콘텐츠 대신 반짝이는 회색 플레이스홀더를 보여주는 UX 패턴입니다.
단순 로딩 스피너보다 사용자에게 콘텐츠 구조를 미리 보여줄 수 있어 체감 로딩 속도가 빠릅니다.

별도 라이브러리 없이 Compose의 `Canvas`, `Brush`, `InfiniteTransition`만으로 구현합니다.

---

#### Step 1. Shimmer 효과 Brush 생성

```kotlin
@Composable
fun rememberShimmerBrush(): Brush {
    // InfiniteTransition: 무한 반복 애니메이션을 관리하는 객체
    val transition = rememberInfiniteTransition(label = "shimmer")

    // translateAnim: -1000f → 1000f 를 1.3초 주기로 무한 반복
    // 이 값이 그라디언트의 X축 오프셋으로 사용되어 빛이 흐르는 효과 연출
    val translateAnim by transition.animateFloat(
        initialValue = -1000f,
        targetValue = 1000f,
        animationSpec = infiniteRepeatable(
            animation = tween(durationMillis = 1300, easing = LinearEasing),
            repeatMode = RepeatMode.Restart
        ),
        label = "shimmer_translate"
    )

    // LinearGradient: translateAnim 값이 변함에 따라 그라디언트 위치가 이동
    // → 빛이 왼쪽에서 오른쪽으로 흐르는 shimmer 효과
    return Brush.linearGradient(
        colors = listOf(
            Color(0xFFE0E0E0), // 어두운 회색
            Color(0xFFF5F5F5), // 밝은 회색 (빛 반사 부분)
            Color(0xFFE0E0E0)  // 다시 어두운 회색
        ),
        start = Offset(translateAnim - 1000f, 0f),
        end = Offset(translateAnim + 1000f, 0f)
    )
}
```

---

#### Step 2. Shimmer 기본 컴포저블

```kotlin
// 재사용 가능한 Shimmer 박스 컴포저블
@Composable
fun ShimmerBox(
    modifier: Modifier = Modifier,
    shape: Shape = RoundedCornerShape(8.dp)
) {
    val shimmerBrush = rememberShimmerBrush()

    Box(
        modifier = modifier
            .clip(shape)              // 지정된 shape으로 자르기
            .background(shimmerBrush) // Shimmer 그라디언트 배경 적용
    )
}
```

---

#### Step 3. 실제 카드 레이아웃을 모방한 Shimmer 스켈레톤

```kotlin
// 뉴스/포스트 카드 형태의 Shimmer 스켈레톤
@Composable
fun PostCardShimmer() {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 16.dp, vertical = 8.dp),
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
    ) {
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            // 프로필 이미지 자리 (원형)
            ShimmerBox(
                modifier = Modifier.size(56.dp),
                shape = CircleShape
            )

            Spacer(modifier = Modifier.width(12.dp))

            Column(modifier = Modifier.weight(1f)) {
                // 제목 자리 (너비 70%)
                ShimmerBox(
                    modifier = Modifier
                        .fillMaxWidth(0.7f)
                        .height(16.dp)
                )
                Spacer(modifier = Modifier.height(8.dp))
                // 부제목 자리 (너비 50%)
                ShimmerBox(
                    modifier = Modifier
                        .fillMaxWidth(0.5f)
                        .height(12.dp)
                )
                Spacer(modifier = Modifier.height(8.dp))
                // 본문 자리 (너비 90%)
                ShimmerBox(
                    modifier = Modifier
                        .fillMaxWidth(0.9f)
                        .height(12.dp)
                )
            }
        }
    }
}
```

---

#### Step 4. 로딩 상태에 따라 실제 콘텐츠와 전환

```kotlin
@Composable
fun PostListScreen(viewModel: PostViewModel = viewModel()) {
    // 로딩 상태 관찰
    val isLoading by viewModel.isLoading.collectAsStateWithLifecycle()
    val posts by viewModel.posts.collectAsStateWithLifecycle()

    LazyColumn {
        if (isLoading) {
            // 로딩 중: Shimmer 아이템 5개 표시
            items(5) {
                PostCardShimmer()
            }
        } else {
            // 로딩 완료: 실제 데이터 표시
            items(posts) { post ->
                PostCard(post = post)
            }
        }
    }
}
```

---

#### 심화: 다크모드 대응

```kotlin
@Composable
fun rememberShimmerBrush(): Brush {
    val transition = rememberInfiniteTransition(label = "shimmer")
    val translateAnim by transition.animateFloat(/* ... */)

    // isSystemInDarkTheme()으로 현재 테마 감지
    val isDarkMode = isSystemInDarkTheme()

    val shimmerColors = if (isDarkMode) {
        listOf(Color(0xFF2A2A2A), Color(0xFF3D3D3D), Color(0xFF2A2A2A))
    } else {
        listOf(Color(0xFFE0E0E0), Color(0xFFF5F5F5), Color(0xFFE0E0E0))
    }

    return Brush.linearGradient(
        colors = shimmerColors,
        start = Offset(translateAnim - 1000f, 0f),
        end = Offset(translateAnim + 1000f, 0f)
    )
}
```

---

### 핵심 정리

- `rememberInfiniteTransition` + `animateFloat`으로 무한 반복 애니메이션 생성
- `Brush.linearGradient`의 `start/end` 오프셋을 애니메이션 값으로 제어 → 흐르는 빛 효과
- `ShimmerBox`를 재사용 가능한 컴포저블로 분리해 다양한 스켈레톤 레이아웃 조합
- `isSystemInDarkTheme()`으로 다크모드 색상 분기 처리

---

## 📝 오늘의 학습 요약

| UI | 핵심 API | 기억할 포인트 |
|---|---|---|
| Splash Screen | `SplashScreen API`, `installSplashScreen()` | `super.onCreate()` 이전 호출 필수 |
| Bottom Nav Bar | `NavController`, `NavigationBar` | `launchSingleTop + saveState + restoreState` 조합 |
| Shimmer Effect | `InfiniteTransition`, `Brush.linearGradient` | 오프셋 애니메이션으로 흐르는 빛 표현 |

---

## 🔗 참고 자료

- [SplashScreen API 공식 문서](https://developer.android.com/develop/ui/views/launch/splash-screen)
- [Navigation Compose 공식 가이드](https://developer.android.com/jetpack/compose/navigation)
- [Compose Animation 공식 문서](https://developer.android.com/jetpack/compose/animation/introduction)
- [Material 3 NavigationBar](https://m3.material.io/components/navigation-bar/overview)
