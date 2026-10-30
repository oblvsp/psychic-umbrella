# Архитектура приложения «Умные часы Senseit R500»
**С учётом TDD и атомарных задач**

> Архитектура спроектирована так, чтобы каждый компонент можно было разрабатывать через TDD-цикл, изолировать зависимости через интерфейсы, доменный слой — чистый Kotlin (тестируется на JVM без эмулятора).

> Соответствует промту `senseit_r500_prompt.md`: TDD-цикл «Красный — Зелёный — Синий (рефакторинг)», ID микро-задач вида `М-01` (кириллическая «М»), переключение экранов только тапом (без свайпов).

---

## 1. Общая схема (слои + тестируемость)

```
┌─────────────────────────────────────────────────────────────┐
│ UI Layer (Views)                                              │
│  MainActivity + Fragments + WatchFaceRenderer                │
│  LiveData / StateFlow → ViewModels → ScreenController       │
│  Переключение: тап → ScreenController.next() (без свайпов)  │
│  Тесты: инструментальные (Espresso) — минимально             │
├─────────────────────────────────────────────────────────────┤
│ Presentation Layer (ViewModels)                              │
│  ViewModels + ScreenController + AutoSwitchController       │
│  Тесты: JUnit + Turbine (Flow) + фейковые UseCase           │
├─────────────────────────────────────────────────────────────┤
│ Domain Layer (Pure Kotlin)                                   │
│  UseCases + Models + Repository Interfaces                  │
│  Тесты: JUnit — мгновенные (секунды, без Android SDK)       │
├─────────────────────────────────────────────────────────────┤
│ Data Layer                                                    │
│  RepositoryImpl + DataSources (API, Cache, Sensors, System) │
│  Тесты: JUnit + MockWebServer + Robolectric                 │
└─────────────────────────────────────────────────────────────┘
```

### Почему так важно для TDD:

| Слой | Преимущество для TDD |
|------|---------------------|
| **Domain** | Не зависит от Android → тесты мгновенные (секунды, не минуты на эмуляторе) |
| **Data** | Скрыт за интерфейсами → ViewModel тестируется с фейками, без сети и датчиков |
| **UI** | Тонкий, покрывается выборочно (на слабом железе эмулятор тормозит) |

---

## 2. Структура пакетов

```
com.senseit.smartwatch/
├── App.kt
├── ui/
│   ├── MainActivity.kt
│   ├── screens/
│   │   ├── BaseScreenFragment.kt
│   │   ├── ClockScreenFragment.kt
│   │   ├── WeatherScreenFragment.kt
│   │   ├── NotificationsScreenFragment.kt
│   │   └── CustomScreenFragment.kt
│   ├── watchface/
│   │   ├── WatchFaceRenderer.kt
│   │   └── SegmentFontView.kt
│   └── settings/
│       └── SettingsActivity.kt
├── presentation/
│   ├── ScreenController.kt          ← переключение экранов по тапу
│   ├── AutoSwitchController.kt      ← авто-перелистывание через TimeProvider
│   ├── ClockViewModel.kt
│   ├── WeatherViewModel.kt
│   ├── NotificationsViewModel.kt
│   └── ConfigViewModel.kt
├── domain/
│   ├── model/
│   │   ├── Weather.kt               ← вместе с WeatherResult (Success/Error)
│   │   ├── NotificationItem.kt
│   │   ├── Alarm.kt
│   │   └── ScreenConfig.kt
│   ├── repository/           ← ТОЛЬКО интерфейсы
│   │   ├── WeatherRepository.kt
│   │   ├── NotificationRepository.kt
│   │   └── SettingsRepository.kt
│   └── usecase/
│       ├── GetTimeUseCase.kt
│       ├── GetWeatherUseCase.kt
│       ├── GetNotificationsUseCase.kt
│       ├── ScheduleAlarmUseCase.kt
│       ├── LoadCustomScreensUseCase.kt
│       ├── SaveScreenConfigUseCase.kt
│       └── LaunchAssistantUseCase.kt
├── data/
│   ├── repository/
│   │   ├── WeatherRepositoryImpl.kt
│   │   ├── NotificationRepositoryImpl.kt
│   │   └── SettingsRepositoryImpl.kt
│   ├── source/
│   │   ├── remote/
│   │   │   └── OpenMeteoApi.kt
│   │   ├── local/
│   │   │   ├── WeatherCache.kt
│   │   │   ├── WatchFaceStorage.kt
│   │   │   └── ScreenConfigStorage.kt
│   │   ├── sensor/
│   │   │   └── SensorManagerWrapper.kt
│   │   └── system/
│   │       ├── AlarmScheduler.kt
│   │       └── TimeProvider.kt
│   └── service/
│       ├── AlarmReceiver.kt
│       ├── AlarmService.kt
│       └── NotificationListener.kt
├── power/
│   ├── PowerManagerHelper.kt
│   └── BurnInProtector.kt
└── hardware/
    └── HardwareButtonHandler.kt
```
### Тестовые директории

```
app/src/test/java/com/senseit/smartwatch/          # JVM-тесты (быстрые)
├── domain/
│   ├── model/           (WeatherTest, AlarmTest, ...)
│   ├── usecase/         (GetWeatherUseCaseTest, ...)
│   └── fake/            ← Фейки для тестов Domain/Presentation
│       ├── FakeWeatherRepository.kt
│       ├── FakeNotificationRepository.kt
│       ├── FakeSettingsRepository.kt
│       └── FakeTimeProvider.kt
├── data/
│   ├── repository/      (WeatherRepositoryImplTest, ...)
│   └── source/          (WeatherCacheTest, ScreenConfigStorageTest, ...)
└── presentation/
    ├── ScreenControllerTest.kt
    ├── AutoSwitchControllerTest.kt
    └── WeatherViewModelTest.kt

app/src/androidTest/java/com/senseit/smartwatch/   # Инструментальные (медленные)
├── ui/
│   ├── ClockScreenTest.kt
│   ├── WeatherScreenTest.kt
│   └── MainActivityNavigationTest.kt
└── service/
    ├── AlarmServiceTest.kt
    └── NotificationListenerTest.kt
```

---

## 3. Ключевые компоненты и TDD-стратегия

### 3.1. Domain Layer (тестируется первым — самый быстрый TDD)

**TDD-цикл для UseCase:**

```
КРАСНЫЙ: GetWeatherUseCaseTest.returnsCachedData() — фейк возвращает stale-данные
ЗЕЛЁНЫЙ: Минимальный UseCase просто пробрасывает вызов
СИНИЙ:   Рефакторинг — обработка ошибок, кэширование, retry (тесты остаются зелёными)
```

**Правило:** UseCase — класс с интерфейсами в конструкторе (DI через конструктор), без зависимостей от Android.

```kotlin
// domain/repository/WeatherRepository.kt
interface WeatherRepository {
    suspend fun getWeather(): WeatherResult
}

// app/src/test/java/.../domain/fake/FakeWeatherRepository.kt
class FakeWeatherRepository(
    var result: WeatherResult = WeatherResult.Success(Weather.stub())
) : WeatherRepository {
    override suspend fun getWeather() = result
}
```

---

### 3.2. Data Layer (тестируется с MockWebServer + Robolectric)

**WeatherRepositoryImpl** — тестируется без сети через `MockWebServer` (OkHttp).

```kotlin
// data/repository/WeatherRepositoryImplTest.kt
class WeatherRepositoryImplTest {
    private val mockWebServer = MockWebServer()
    private val api = OpenMeteoApi(mockWebServer.url("/").toString())
    private val cache = WeatherCache(TemporaryFolder())
    private val repo = WeatherRepositoryImpl(api, cache)

    @Test
    fun returnsCacheWhenNetworkFails() = runTest { ... }

    @Test
    fun updatesCacheOnSuccess() = runTest { ... }

    @Test
    fun usesCacheWithinTtl() = runTest { ... }
}
```

- **WeatherCache** — обычный JSON-файл в `filesDir`, тестируется чисто на JVM (TemporaryFolder) — не нужен Android.
- **ScreenConfigStorage** — тестируется на JVM: читает JSON из временной папки, парсит в `ScreenConfig`.
- **NotificationListener** — тестируется инструментально (эмулятор), т.к. требует системного сервиса. Микро-задачи для него — последние в плане.

---

### 3.3. Presentation Layer (тестируется с фейками UseCase)

**ScreenController** — ключевой класс навигации. Тестируется без Android:

```kotlin
// presentation/ScreenControllerTest.kt
class ScreenControllerTest {
    @Test
    fun next_movesToNextScreen() { /* КРАСНЫЙ: контроллер с 3 экранами */ }

    @Test
    fun next_wrapsAround() { ... }

    @Test
    fun goTo_switchesToSpecifiedScreen() { ... }
}
```

**AutoSwitchController** — тестируется через фейковый `TimeProvider` (инъекция времени):

```kotlin
// data/source/system/TimeProvider.kt
interface TimeProvider {
    fun currentTimeMs(): Long
}

// app/src/test/java/.../domain/fake/FakeTimeProvider.kt
class FakeTimeProvider : TimeProvider {
    var currentMs = 0L
    override fun currentTimeMs() = currentMs
}
```

Это позволяет проверить «через 10 секунд экран переключился» без реального ожидания.

**ViewModel** — тестируется с Turbine (Flow) + фейковые UseCase.

---

### 3.4. UI Layer (минимум тестов — тонкий слой)

- **SegmentFontView** — тестируется через Robolectric (проверка: `onDraw` не падает, размеры корректны).
- **MainActivity** — 1-2 инструментальных теста навигации (тап → экран сменился).
- **ClockScreenFragment** — 1 тест: время обновилось через LiveData/StateFlow.
---

## 4. Соответствие архитектуры микро-задачам декомпозиции

Микро-задачи описываются в формате из п.13 промта `senseit_r500_prompt.md`.

| ID    | Микро-задача                              | Файл                                      |
|-------|-------------------------------------------|-------------------------------------------|
| М-01  | Domain-модель Weather                     | `domain/model/Weather.kt`                 |
| М-02  | Интерфейс WeatherRepository               | `domain/repository/WeatherRepository.kt`  |
| М-03  | GetWeatherUseCase                         | `domain/usecase/GetWeatherUseCase.kt`     |
| М-04  | FakeWeatherRepository                     | `app/src/test/java/com/senseit/smartwatch/domain/fake/FakeWeatherRepository.kt` |
| М-05  | WeatherCache (local JSON)                 | `data/source/local/WeatherCache.kt`       |
| М-06  | WeatherRepositoryImpl                     | `data/repository/WeatherRepositoryImpl.kt`|
| М-07  | WeatherViewModel                          | `presentation/WeatherViewModel.kt`        |
| М-08  | WeatherScreenFragment                     | `ui/screens/WeatherScreenFragment.kt`     |

**Правило:** каждая микро-задача = один файл + один тест. Локальная LLM получает задачу в формате из п.13 промта и не может «убежать» в сторону.

---

## 5. Технологический стек (все OpenSource)

| Компонент       | Библиотека                                      |
|-----------------|-------------------------------------------------|
| Язык            | Kotlin + Coroutines + Flow                      |
| UI              | AndroidX (core, fragment, viewpager2*, lifecycle)|
| Сеть            | Retrofit + OkHttp                               |
| JSON            | kotlinx.serialization                           |
| Тесты JVM       | JUnit 4 + Mockito-Kotlin + Turbine              |
| Тесты HTTP      | MockWebServer (OkHttp)                          |
| Тесты Android   | Robolectric                                     |
| Тесты UI        | Espresso                                        |
| Время           | TimeProvider (кастомный, инъектируемый)         |
| Шрифт           | DSEG7 (7-segment display font)                  |

**Не используется:** Firebase, GMS, Hilt, Root (JSON-файлы достаточно), WorkManager, React Native, Flutter.

\* ViewPager2 — только как контейнер Fragments, с `isUserInputEnabled = false`: свайпы запрещены п.2 промта, переключение экранов — тапом через `ScreenController`.

---

## 6. Жизненный цикл и экономия ресурсов (с учётом тестируемости)

| Проблема      | Решение                                                          |
|---------------|------------------------------------------------------------------|
| RAM           | `offscreenPageLimit = 1` для ViewPager2                          |
| CPU           | Часы — 1 Гц, погода — только на `onResume` / по расписанию       |
| Батарея       | `Handler` + `TimeProvider` (инъекция) вместо `Timer`/`Thread`   |
| Doze mode     | `setExactAndAllowWhileIdle` + разрешение «Без оптимизации»       |
| Burn-in       | `BurnInProtector` (сдвиг ±1-2 px каждую минуту)              |

---

## 7. Диаграмма компонентов (с границами тестов)

```
┌─────────────────────────────────────────────────────────────────┐
│ MainActivity (androidTest)                                      │
│   └─ ScreenController                                           │
│       ├─ AutoSwitchController  ← TimeProvider (fake в тестах)   │
│       └─ ViewModels (JVM test + Turbine)                        │
│           ├─ ClockViewModel       → GetTimeUseCase (domain)     │
│           ├─ WeatherViewModel     → GetWeatherUseCase (domain)  │
│           ├─ NotificationsVM      → GetNotificationsUseCase     │
│           └─ ConfigViewModel      → Load/SaveScreenConfig       │
│                                                                  │
│ [Интерфейсы репозиториев] ←────────────────────── Domain Layer   │
│   ├─ WeatherRepository     → WeatherRepositoryImpl (MockWebSrv) │
│   ├─ NotificationRepository→ NotificationRepositoryImpl         │
│   └─ SettingsRepository    → SettingsRepositoryImpl             │
│                                                                  │
│ Data Sources (JVM + Robolectric):                               │
│   ├─ OpenMeteoApi         → MockWebServer                       │
│   ├─ WeatherCache         → TemporaryFolder (JUnit)             │
│   ├─ NotificationListener → androidTest (эмулятор)              │
│   ├─ AlarmScheduler       → Robolectric (AlarmManager)          │
│   ├─ SensorManagerWrapper → Robolectric                         │
│   └─ TimeProvider         → FakeTimeProvider (JVM)              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. Важное при реализации (TDD-специфика)

1. **Начинай с Domain-слоя.** Модели и UseCase тестируются на JVM за секунды — это и есть TDD. UI и сервисы — в последнюю очередь.

2. **Все Android-зависимости — за интерфейсами.**
   - `TimeProvider`, `SensorManagerWrapper`, `AlarmScheduler` — интерфейсы
   - Реальная реализация — в `data/source/system/`
   - Фейк — в `test/.../fake/`

3. **NotificationListenerService** — единственный компонент, который нельзя нормально протестировать JVM-тестами.
   - Для него: 1 инструментальный тест + ручная проверка на устройстве.

4. **AlarmService** — тестируется через Robolectric (эмуляция AlarmManager) + 1 инструментальный тест на реальном срабатывании.

5. **Каждая микро-задача — отдельный коммит** с сообщением `М-XX: [описание]` (КРАСНЫЙ → ЗЕЛЁНЫЙ → СИНИЙ), чтобы в истории был виден TDD-цикл.

6. **TDD_LOG.md** (в корне репозитория) ведётся как часть «Результата» — фиксация циклов КРАСНЫЙ / ЗЕЛЁНЫЙ / СИНИЙ для каждой задачи.

---

## 9. Следующие шаги (по запросу)

Могу подготовить:

1. **TASKS.md** — полный список микро-задач по этой архитектуре (с ID, файлами, тестами, порядком выполнения)
2. **TDD_LOG.md** — шаблон для фиксации TDD-циклов
3. **Детальную реализацию одного модуля** (например, `ScreenController` + `AutoSwitchController`) с ПОЛНЫМ TDD-логом (КРАСНЫЙ → ЗЕЛЁНЫЙ → СИНИЙ)

---