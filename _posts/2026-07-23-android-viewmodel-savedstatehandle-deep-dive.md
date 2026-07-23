---
layout: post
title: "Android ViewModel & SavedStateHandle 심화: 프로세스 종료에서도 살아남는 UI 상태 관리 완전 정복"
date: 2026-07-23
categories: [android, flutter]
tags: [android, viewmodel, savedstatehandle, jetpack, kotlin, stateflow, hilt, architecture]
---

## 1. 들어가며

Android 앱 개발에서 가장 자주 맞닥뜨리는 문제 중 하나는 **상태(state) 관리**입니다. 사용자가 검색창에 텍스트를 입력하고, 목록을 스크롤하고, 탭을 선택하는 모든 인터랙션은 UI 상태를 만들어냅니다. 이 상태가 화면 회전이나 앱 일시 중단 후에도 유지되어야 한다면, 개발자는 언제 어떤 방법을 써야 하는지 명확히 이해해야 합니다.

Android Jetpack은 이를 위해 `ViewModel`과 `SavedStateHandle`을 제공합니다. 두 API는 겉으로 비슷해 보이지만, **각각 다른 종류의 상태 손실을 해결**합니다. 이 글에서는 두 컴포넌트의 내부 동작 원리부터 실전 패턴까지 심층적으로 파헤칩니다.

---

## 2. ViewModel의 내부 동작 원리

### ViewModelStore와 ViewModelStoreOwner

`ViewModel`은 단순한 데이터 홀더가 아닙니다. 내부적으로 `ViewModelStore`라는 컨테이너에 보관되며, `Activity`나 `Fragment`가 구현하는 `ViewModelStoreOwner` 인터페이스를 통해 접근됩니다.

```kotlin
// ViewModelStore 내부 구조 (단순화)
class ViewModelStore {
    private val map = HashMap<String, ViewModel>()

    fun put(key: String, viewModel: ViewModel) { map[key] = viewModel }
    fun get(key: String): ViewModel? = map[key]

    fun clear() {
        for (vm in map.values) vm.onCleared()
        map.clear()
    }
}
```

`ComponentActivity`는 `NonConfigurationInstance`를 통해 구성 변경(화면 회전, 다크 모드 전환 등) 시에도 `ViewModelStore`를 유지합니다. 시스템이 `Activity`를 파괴했다가 재생성할 때, `ViewModelStore`는 파괴되지 않고 새 `Activity` 인스턴스에 전달됩니다.

```kotlin
// ComponentActivity의 구성 변경 생존 메커니즘 (단순화)
class ComponentActivity : Activity(), ViewModelStoreOwner {
    private val viewModelStore = ViewModelStore()

    override fun onRetainNonConfigurationInstance(): Any {
        return viewModelStore // 구성 변경 시 살아남음
    }

    override fun onDestroy() {
        super.onDestroy()
        if (!isChangingConfigurations) {
            viewModelStore.clear() // 진짜 종료 시에만 정리
        }
    }
}
```

이 메커니즘 덕분에 `ViewModel`은 화면 회전에 살아남습니다. 그러나 **시스템이 프로세스 자체를 종료**하면(메모리 부족, 백그라운드 제한 등) `ViewModelStore`도 함께 사라집니다.

---

## 3. 구성 변경 vs 프로세스 종료: 결정적 차이

Android에서 상태가 손실되는 상황은 크게 두 가지입니다.

| 상황 | 설명 | ViewModel 생존 | SavedStateHandle 필요 |
|------|------|:---:|:---:|
| 구성 변경 | 화면 회전, 폴더블 펼치기 | ✅ | 불필요 |
| 시스템 프로세스 종료 | 메모리 부족, 백그라운드 제한 | ❌ | ✅ 필수 |
| 사용자 직접 종료 | 뒤로 버튼으로 스택 제거 | ❌ | 불필요 (의도적 종료) |

시스템 프로세스 종료는 사용자가 앱을 백그라운드에 두고 다른 앱을 사용할 때 발생합니다. 사용자는 앱이 종료된 사실을 모르기 때문에, 복귀 시 이전 상태가 그대로 있어야 자연스러운 UX를 제공할 수 있습니다.

`SavedStateHandle`은 내부적으로 `Activity`의 `onSaveInstanceState` 번들과 연결되어 작동합니다. 시스템이 프로세스를 종료하기 전에 상태를 직렬화하고, 재시작 시 복원합니다.

---

## 4. SavedStateHandle 기본 사용법

### Hilt 없이 SavedStateHandle 주입받기

`by viewModels()` 위임 프로퍼티를 통해 `SavedStateHandle`이 자동으로 주입됩니다. `ComponentActivity`와 `Fragment` 모두 `SavedStateRegistryOwner`를 구현하고 있어, 별도 팩토리 없이 간단하게 사용할 수 있습니다.

```kotlin
class SearchViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    // getStateFlow로 SavedStateHandle과 연동된 StateFlow 생성
    // 값 변경 시 SavedStateHandle에 자동 저장됨
    val searchQuery: StateFlow<String> =
        savedStateHandle.getStateFlow("query", initialValue = "")

    fun updateQuery(newQuery: String) {
        // StateFlow를 업데이트하면서 SavedStateHandle에도 저장
        savedStateHandle["query"] = newQuery
    }

    fun clearQuery() {
        savedStateHandle["query"] = ""
    }
}

// AbstractSavedStateViewModelFactory를 직접 사용하는 경우
class SearchViewModelFactory(
    owner: SavedStateRegistryOwner,
    defaultArgs: Bundle? = null
) : AbstractSavedStateViewModelFactory(owner, defaultArgs) {

    override fun <T : ViewModel> create(
        key: String,
        modelClass: Class<T>,
        handle: SavedStateHandle
    ): T {
        @Suppress("UNCHECKED_CAST")
        return SearchViewModel(handle) as T
    }
}

// Fragment에서 사용
class SearchFragment : Fragment() {
    // by viewModels()는 내부적으로 SavedStateHandle을 자동 연결
    private val viewModel: SearchViewModel by viewModels {
        SearchViewModelFactory(this)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.searchQuery.collect { query ->
                    binding.searchEditText.setText(query)
                }
            }
        }
    }
}
```

---

## 5. Hilt + SavedStateHandle + StateFlow 통합 패턴

실무에서는 Hilt를 사용하는 경우가 많습니다. `@HiltViewModel` 어노테이션과 함께 `SavedStateHandle`을 생성자에 선언하면 Hilt가 자동으로 주입합니다.

```kotlin
@HiltViewModel
class ProductDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val productRepository: ProductRepository
) : ViewModel() {

    // Navigation Component SafeArgs 인수를 SavedStateHandle로 직접 접근
    private val productId: Int = checkNotNull(savedStateHandle["productId"])

    // 비즈니스 로직 상태는 MutableStateFlow로 관리
    private val _uiState = MutableStateFlow(ProductUiState())
    val uiState: StateFlow<ProductUiState> = _uiState.asStateFlow()

    // 필터 선택 상태: 프로세스 종료 후에도 복원됨
    // getStateFlow가 반환하는 StateFlow는 변경 시 SavedStateHandle에 자동 저장
    val selectedFilter: StateFlow<FilterType> =
        savedStateHandle.getStateFlow("filter", FilterType.ALL)

    init {
        loadProduct()
        // 필터 변경 시마다 자동으로 상품 목록 갱신
        viewModelScope.launch {
            selectedFilter.collect { filter ->
                filterProducts(filter)
            }
        }
    }

    fun selectFilter(filter: FilterType) {
        // savedStateHandle에 저장하면 StateFlow도 자동으로 업데이트됨
        savedStateHandle["filter"] = filter
    }

    private fun loadProduct() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            try {
                val product = productRepository.getProduct(productId)
                _uiState.update { it.copy(isLoading = false, product = product) }
            } catch (e: Exception) {
                _uiState.update { it.copy(isLoading = false, error = e.message) }
            }
        }
    }

    private fun filterProducts(filter: FilterType) {
        viewModelScope.launch {
            val filtered = productRepository.getFilteredProducts(productId, filter)
            _uiState.update { it.copy(filteredItems = filtered) }
        }
    }
}

// 데이터 클래스
data class ProductUiState(
    val isLoading: Boolean = false,
    val product: Product? = null,
    val filteredItems: List<Product> = emptyList(),
    val error: String? = null
)

enum class FilterType { ALL, SALE, NEW, IN_STOCK }

// Compose에서 사용
@Composable
fun ProductDetailScreen(
    viewModel: ProductDetailViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val selectedFilter by viewModel.selectedFilter.collectAsStateWithLifecycle()

    Column {
        FilterRow(
            selectedFilter = selectedFilter,
            onFilterSelected = viewModel::selectFilter
        )
        when {
            uiState.isLoading -> CircularProgressIndicator()
            uiState.error != null -> Text("오류: ${uiState.error}")
            else -> ProductList(items = uiState.filteredItems)
        }
    }
}
```

`savedStateHandle.getStateFlow()`가 반환하는 `StateFlow`는 값이 변경될 때마다 자동으로 `SavedStateHandle`을 업데이트합니다. 단, **저장 가능한 타입**이어야 하며, `Enum`은 기본적으로 `Parcelable`을 구현하지 않으므로 `@Parcelize`를 추가하거나 `String`으로 변환하여 저장해야 합니다.

---

## 6. 복잡한 객체 저장: Parcelable과 @Serializable

`SavedStateHandle`은 `Bundle`이 지원하는 타입만 저장할 수 있습니다. 커스텀 데이터 클래스를 저장하려면 `Parcelable`을 구현하거나, `@Serializable` 어노테이션을 활용합니다.

```kotlin
// @Parcelize로 Parcelable 자동 구현 (kotlin-parcelize 플러그인 필요)
@Parcelize
data class FilterState(
    val sortOrder: SortOrder = SortOrder.NEWEST,
    val minPrice: Int = 0,
    val maxPrice: Int = 100_000,
    val inStockOnly: Boolean = false
) : Parcelable

@Parcelize
enum class SortOrder : Parcelable { NEWEST, PRICE_ASC, PRICE_DESC, POPULAR }

@HiltViewModel
class FilterViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle
) : ViewModel() {

    // Parcelable 객체를 StateFlow로 노출
    val filterState: StateFlow<FilterState> =
        savedStateHandle.getStateFlow("filterState", FilterState())

    fun updateFilter(newFilter: FilterState) {
        savedStateHandle["filterState"] = newFilter
    }

    fun resetFilter() {
        savedStateHandle["filterState"] = FilterState()
    }
}
```

---

## 7. Navigation Component와의 통합

Navigation Component를 사용할 때, SafeArgs로 전달되는 인수는 `SavedStateHandle`을 통해 ViewModel에서 직접 접근할 수 있습니다. 이는 Fragment 인수 파싱 코드를 크게 단순화합니다.

```kotlin
// nav_graph.xml에 정의된 인수
// <argument android:name="productId" app:argType="integer" android:defaultValue="-1" />

@HiltViewModel
class ProductViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val repo: ProductRepository
) : ViewModel() {

    // Navigation 인수에 직접 접근 — Fragment.arguments 파싱 불필요
    private val productId: Int = checkNotNull(savedStateHandle["productId"]) {
        "productId는 Navigation 인수로 반드시 전달되어야 합니다."
    }

    // 여러 백스택 엔트리 간 공유 ViewModel을 사용하는 경우
    // by navGraphViewModels(R.id.checkout_graph) 사용
}
```

---

## 8. 주의사항 및 실전 팁

### 8.1 저장 가능한 데이터 크기 제한

`SavedStateHandle`은 `Bundle`을 기반으로 하므로, IPC 트랜잭션 크기 제한(통상 1MB)이 적용됩니다. 대용량 데이터(이미지, 전체 목록 등)를 저장하지 말고, 검색어나 선택된 ID처럼 **최소한의 식별자**만 저장하세요.

```kotlin
// ❌ 잘못된 예: 전체 목록을 SavedStateHandle에 저장
savedStateHandle["products"] = productList // TransactionTooLargeException 위험

// ✅ 올바른 예: 식별자만 저장, 실제 데이터는 Repository가 관리
savedStateHandle["selectedProductId"] = product.id
savedStateHandle["searchQuery"] = query
```

### 8.2 Activity stopped 이후 쓰기 주의

Compose 환경에서 `Activity`가 `onStop` 상태가 된 이후 `SavedStateHandle`에 쓴 값은, 다음 `onStart`-`onStop` 사이클이 완성되기 전까지 프로세스 종료 시 저장되지 않을 수 있습니다. 중요한 상태 변경은 사용자 인터랙션이 활성 중일 때 즉시 저장하세요.

### 8.3 ViewModel 스코프와 Navigation 백스택

Navigation의 백스택 엔트리별로 별도의 `ViewModel` 인스턴스가 생성됩니다. 동일한 nav graph 내 여러 화면에서 상태를 공유해야 할 때는 `by navGraphViewModels()`를 활용하세요.

```kotlin
// 체크아웃 플로우 전체에서 공유되는 ViewModel
private val checkoutViewModel: CheckoutViewModel by navGraphViewModels(
    R.id.checkout_graph
) { defaultViewModelProviderFactory }
```

### 8.4 테스트 전략

유닛 테스트에서는 `SavedStateHandle`을 직접 인스턴스화하여 초기 상태를 설정할 수 있습니다.

```kotlin
@Test
fun `필터 상태가 프로세스 재시작 후 복원된다`() = runTest {
    val savedStateHandle = SavedStateHandle(
        initialState = mapOf(
            "filter" to FilterType.SALE,
            "productId" to 42
        )
    )
    val viewModel = ProductDetailViewModel(
        savedStateHandle = savedStateHandle,
        productRepository = fakeRepository
    )

    assertThat(viewModel.selectedFilter.value).isEqualTo(FilterType.SALE)
}

@Test
fun `필터 변경 시 SavedStateHandle에 저장된다`() = runTest {
    val savedStateHandle = SavedStateHandle()
    val viewModel = ProductDetailViewModel(savedStateHandle, fakeRepository)

    viewModel.selectFilter(FilterType.NEW)

    assertThat(savedStateHandle.get<FilterType>("filter")).isEqualTo(FilterType.NEW)
}
```

---

## 9. 상태 관리 옵션 비교 정리

| 기준 | ViewModel | SavedStateHandle | rememberSaveable (Compose) |
|------|-----------|:---:|:---:|
| 구성 변경 대응 | ✅ | ✅ | ✅ |
| 프로세스 종료 대응 | ❌ | ✅ | ✅ |
| 저장 용량 | 제한 없음 | ~1MB | ~1MB |
| 저장 가능 타입 | 모두 | Bundle 지원 타입 | Saver 지원 타입 |
| 비즈니스 로직 포함 | ✅ | ✅ | ❌ |
| UI 전용 임시 상태 | ❌ | △ | ✅ |

`ViewModel`은 구성 변경 시 데이터를 보호하고, `SavedStateHandle`은 시스템 프로세스 종료에서 UI 상태를 복원합니다. 두 API를 목적에 맞게 조합하고, 저장할 데이터는 최소한의 식별자만 담는 것이 핵심 원칙입니다. Hilt와 `getStateFlow()`를 함께 활용하면 보일러플레이트 없이 견고한 상태 관리 레이어를 구축할 수 있습니다.

---

## 참고 자료
- [ViewModel overview - Android Developers](https://developer.android.com/topic/libraries/architecture/viewmodel)
- [Saved State module for ViewModel - Android Developers](https://developer.android.com/topic/libraries/architecture/viewmodel/viewmodel-savedstate)
- [Save UI states - Android Developers](https://developer.android.com/topic/libraries/architecture/saving-states)
