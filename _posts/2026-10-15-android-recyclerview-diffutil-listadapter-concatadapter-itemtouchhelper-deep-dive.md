---
layout: post
title: "Android RecyclerView 심화: DiffUtil, ListAdapter, ConcatAdapter, ItemTouchHelper로 고성능 목록 UI 구현하기"
date: 2026-10-15
categories: [android]
tags: [android, recyclerview, diffutil, listadapter, concatadapter, itemtouchhelper, kotlin]
---

Android 앱 개발에서 목록(List) UI는 피할 수 없는 핵심 구성 요소입니다. 연락처 목록, 피드, 쇼핑 카트 등 거의 모든 앱에서 RecyclerView를 사용합니다. 그런데 많은 개발자가 `notifyDataSetChanged()`라는 "폭탄" 메서드를 습관적으로 사용하며 성능을 낭비하고 있습니다. 이 글에서는 `DiffUtil`, `ListAdapter`, `ConcatAdapter`, `ItemTouchHelper`를 활용하여 진정한 고성능 RecyclerView를 구현하는 방법을 심화 수준에서 다룹니다.

---

## 1. 왜 `notifyDataSetChanged()`는 피해야 하는가

`notifyDataSetChanged()`는 RecyclerView에게 "데이터가 바뀌었으니 전부 다시 그려"라고 지시합니다. 아이템이 100개든 1,000개든 상관없이 전체 목록을 무효화하고 재바인딩합니다. 이로 인해 다음과 같은 문제가 발생합니다.

- **불필요한 rebind**: 변경되지 않은 아이템도 `onBindViewHolder()`가 다시 호출됩니다.
- **애니메이션 소실**: 추가/삭제/이동에 대한 부드러운 애니메이션이 사라집니다.
- **스크롤 위치 초기화**: 사용자가 스크롤한 위치가 예기치 않게 변경될 수 있습니다.

이를 해결하기 위해 등장한 것이 **DiffUtil**입니다.

---

## 2. DiffUtil 개념: 두 리스트의 차이를 계산한다

`DiffUtil`은 두 리스트(이전 리스트 / 새 리스트) 사이의 **최소 업데이트 연산**을 계산합니다. 내부적으로 Eugene Myers의 diff 알고리즘을 사용하며, 시간 복잡도는 `O(N + D²)` (N: 항목 수, D: 편집 거리)입니다. 계산 결과를 `DiffUtil.DiffResult`로 반환하며, 이를 어댑터에 `dispatchUpdatesTo()`로 전달하면 `notifyItemInserted`, `notifyItemRemoved`, `notifyItemChanged`, `notifyItemMoved` 등의 세밀한 알림이 자동으로 처리됩니다.

DiffUtil을 사용하려면 `DiffUtil.Callback`을 구현해야 합니다.

```kotlin
class ProductDiffCallback(
    private val oldList: List<Product>,
    private val newList: List<Product>
) : DiffUtil.Callback() {

    override fun getOldListSize(): Int = oldList.size

    override fun getNewListSize(): Int = newList.size

    // 두 아이템이 같은 엔티티인지 (ID 비교)
    override fun areItemsTheSame(oldItemPosition: Int, newItemPosition: Int): Boolean {
        return oldList[oldItemPosition].id == newList[newItemPosition].id
    }

    // 같은 엔티티라면, 내용도 동일한지 (데이터 비교)
    override fun areContentsTheSame(oldItemPosition: Int, newItemPosition: Int): Boolean {
        return oldList[oldItemPosition] == newList[newItemPosition]
    }

    // (선택) 부분 변경 payload 반환 — 세밀한 rebind 최적화에 사용
    override fun getChangePayload(oldItemPosition: Int, newItemPosition: Int): Any? {
        val oldProduct = oldList[oldItemPosition]
        val newProduct = newList[newItemPosition]
        val changes = mutableMapOf<String, Any>()
        if (oldProduct.price != newProduct.price) changes["price"] = newProduct.price
        if (oldProduct.stock != newProduct.stock) changes["stock"] = newProduct.stock
        return if (changes.isEmpty()) null else changes
    }
}
```

`areItemsTheSame`은 주로 **고유 ID** 기반으로 비교하고, `areContentsTheSame`은 **모든 필드**가 같은지 비교합니다. `data class`를 사용하면 `equals()`가 자동 생성되므로 `areContentsTheSame` 구현이 간단해집니다.

---

## 3. ListAdapter: DiffUtil을 내장한 권장 어댑터

매번 `DiffUtil.Callback`을 수동으로 구현하고 `calculateDiff()`를 호출하는 것은 번거롭습니다. `ListAdapter`는 내부적으로 `AsyncListDiffer`를 사용하여 **백그라운드 스레드에서 diff를 자동 계산**하는 권장 어댑터입니다.

```kotlin
data class Product(val id: Long, val name: String, val price: Int, val stock: Int)

class ProductAdapter(
    private val onItemClick: (Product) -> Unit,
    private val onAddToCart: (Product) -> Unit
) : ListAdapter<Product, ProductAdapter.ViewHolder>(ProductDiffItemCallback) {

    companion object {
        val ProductDiffItemCallback = object : DiffUtil.ItemCallback<Product>() {
            override fun areItemsTheSame(oldItem: Product, newItem: Product): Boolean =
                oldItem.id == newItem.id

            override fun areContentsTheSame(oldItem: Product, newItem: Product): Boolean =
                oldItem == newItem

            override fun getChangePayload(oldItem: Product, newItem: Product): Any? {
                val changes = mutableMapOf<String, Any>()
                if (oldItem.price != newItem.price) changes["price"] = newItem.price
                if (oldItem.stock != newItem.stock) changes["stock"] = newItem.stock
                return changes.ifEmpty { null }
            }
        }
    }

    inner class ViewHolder(private val binding: ItemProductBinding) :
        RecyclerView.ViewHolder(binding.root) {

        init {
            binding.root.setOnClickListener {
                val product = getItem(adapterPosition)
                onItemClick(product)
            }
            binding.btnAddToCart.setOnClickListener {
                val product = getItem(adapterPosition)
                onAddToCart(product)
            }
        }

        fun bind(product: Product) {
            binding.tvName.text = product.name
            binding.tvPrice.text = "₩${product.price}"
            binding.tvStock.text = "재고: ${product.stock}개"
        }

        // payload를 받는 부분 rebind — 전체 rebind 방지
        fun bindPayload(payload: Map<String, Any>) {
            payload["price"]?.let { binding.tvPrice.text = "₩$it" }
            payload["stock"]?.let { binding.tvStock.text = "재고: ${it}개" }
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val binding = ItemProductBinding.inflate(
            LayoutInflater.from(parent.context), parent, false
        )
        return ViewHolder(binding)
    }

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        holder.bind(getItem(position))
    }

    @Suppress("UNCHECKED_CAST")
    override fun onBindViewHolder(
        holder: ViewHolder,
        position: Int,
        payloads: MutableList<Any>
    ) {
        if (payloads.isEmpty()) {
            holder.bind(getItem(position))
        } else {
            // payload가 있으면 변경된 필드만 업데이트
            val payload = payloads.filterIsInstance<Map<String, Any>>()
                .reduceOrNull { acc, map -> acc + map } ?: emptyMap()
            holder.bindPayload(payload)
        }
    }
}
```

Fragment나 ViewModel에서 사용하는 방식:

```kotlin
// Fragment
class ProductListFragment : Fragment() {
    private val viewModel: ProductViewModel by viewModels()
    private val adapter = ProductAdapter(
        onItemClick = { product -> navigateToDetail(product.id) },
        onAddToCart = { product -> viewModel.addToCart(product) }
    )

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding.recyclerView.adapter = adapter

        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.products.collect { products ->
                    // submitList()가 AsyncListDiffer를 통해 백그라운드에서 diff 계산
                    adapter.submitList(products)
                }
            }
        }
    }
}
```

`submitList()`는 항상 **새 리스트 인스턴스**를 전달해야 합니다. 기존 리스트를 수정해서 넘기면 diff가 동작하지 않습니다. ViewModel에서 `StateFlow<List<Product>>`를 사용할 때 `copy()`나 `toMutableList().also { ... }` 패턴을 사용하는 이유가 이것입니다.

---

## 4. ConcatAdapter: 여러 어댑터를 하나로 합치기

실무에서는 단순 목록 외에 **헤더, 광고 배너, 푸터, 로딩 인디케이터** 등 다양한 타입의 섹션을 하나의 RecyclerView에 표시해야 하는 경우가 많습니다. 과거에는 하나의 어댑터 안에서 `viewType`으로 분기하는 복잡한 코드를 작성했지만, `ConcatAdapter`를 사용하면 각 섹션을 독립적인 어댑터로 분리할 수 있습니다.

```kotlin
// 헤더 어댑터: 배너 이미지 1개
class BannerAdapter(private val bannerUrl: String) :
    RecyclerView.Adapter<BannerAdapter.ViewHolder>() {

    inner class ViewHolder(binding: ItemBannerBinding) :
        RecyclerView.ViewHolder(binding.root) {
        val imageView = binding.ivBanner
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int) =
        ViewHolder(ItemBannerBinding.inflate(LayoutInflater.from(parent.context), parent, false))

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        Glide.with(holder.imageView).load(bannerUrl).into(holder.imageView)
    }

    override fun getItemCount() = 1
}

// 로딩 푸터 어댑터
class LoadingAdapter(private var isLoading: Boolean = false) :
    RecyclerView.Adapter<LoadingAdapter.ViewHolder>() {

    inner class ViewHolder(binding: ItemLoadingBinding) :
        RecyclerView.ViewHolder(binding.root)

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int) =
        ViewHolder(ItemLoadingBinding.inflate(LayoutInflater.from(parent.context), parent, false))

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {}

    override fun getItemCount() = if (isLoading) 1 else 0

    fun showLoading() {
        if (!isLoading) { isLoading = true; notifyItemInserted(0) }
    }

    fun hideLoading() {
        if (isLoading) { isLoading = false; notifyItemRemoved(0) }
    }
}

// Fragment에서 ConcatAdapter로 결합
class ShopFragment : Fragment() {
    private val bannerAdapter = BannerAdapter("https://example.com/banner.png")
    private val productAdapter = ProductAdapter(onItemClick = {}, onAddToCart = {})
    private val loadingAdapter = LoadingAdapter()

    private val concatAdapter = ConcatAdapter(
        ConcatAdapter.Config.Builder()
            .setIsolateViewTypes(true) // 각 어댑터의 viewType 격리 (충돌 방지)
            .build(),
        bannerAdapter,
        productAdapter,
        loadingAdapter
    )

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding.recyclerView.adapter = concatAdapter

        // 페이지네이션: 스크롤 끝 감지
        binding.recyclerView.addOnScrollListener(object : RecyclerView.OnScrollListener() {
            override fun onScrolled(recyclerView: RecyclerView, dx: Int, dy: Int) {
                val layoutManager = recyclerView.layoutManager as LinearLayoutManager
                val lastVisible = layoutManager.findLastVisibleItemPosition()
                val total = concatAdapter.itemCount
                if (lastVisible >= total - 3) {
                    viewModel.loadNextPage()
                }
            }
        })

        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    productAdapter.submitList(state.products)
                    if (state.isLoadingNextPage) loadingAdapter.showLoading()
                    else loadingAdapter.hideLoading()
                }
            }
        }
    }
}
```

`ConcatAdapter.Config.setIsolateViewTypes(true)`(기본값)는 각 어댑터의 ViewType 정수가 서로 충돌하지 않도록 내부적으로 매핑합니다. 여러 `ListAdapter`를 독립적으로 개발하고 안전하게 결합할 수 있습니다.

---

## 5. ItemTouchHelper: 스와이프 삭제와 드래그 재정렬

목록에서 좌우 스와이프로 삭제하거나 아이템을 드래그로 재정렬하는 기능은 `ItemTouchHelper`로 구현합니다.

```kotlin
class ProductItemTouchCallback(
    private val adapter: ProductAdapter,
    private val onSwipeDelete: (Product) -> Unit
) : ItemTouchHelper.SimpleCallback(
    ItemTouchHelper.UP or ItemTouchHelper.DOWN, // 드래그 방향
    ItemTouchHelper.LEFT or ItemTouchHelper.RIGHT // 스와이프 방향
) {
    override fun onMove(
        recyclerView: RecyclerView,
        viewHolder: RecyclerView.ViewHolder,
        target: RecyclerView.ViewHolder
    ): Boolean {
        val from = viewHolder.adapterPosition
        val to = target.adapterPosition
        // 아이템 순서 변경 (어댑터에서 리스트를 조작 후 submitList)
        adapter.moveItem(from, to)
        return true
    }

    override fun onSwiped(viewHolder: RecyclerView.ViewHolder, direction: Int) {
        val position = viewHolder.adapterPosition
        val product = adapter.currentList[position]
        onSwipeDelete(product)
    }

    // 스와이프 시 배경 그리기 (빨간색 삭제 아이콘)
    override fun onChildDraw(
        c: Canvas,
        recyclerView: RecyclerView,
        viewHolder: RecyclerView.ViewHolder,
        dX: Float, dY: Float,
        actionState: Int,
        isCurrentlyActive: Boolean
    ) {
        if (actionState == ItemTouchHelper.ACTION_STATE_SWIPE) {
            val itemView = viewHolder.itemView
            val paint = Paint().apply { color = Color.RED }
            if (dX < 0) { // 왼쪽 스와이프
                c.drawRect(
                    itemView.right + dX, itemView.top.toFloat(),
                    itemView.right.toFloat(), itemView.bottom.toFloat(), paint
                )
            } else { // 오른쪽 스와이프
                c.drawRect(
                    itemView.left.toFloat(), itemView.top.toFloat(),
                    itemView.left + dX, itemView.bottom.toFloat(), paint
                )
            }
        }
        super.onChildDraw(c, recyclerView, viewHolder, dX, dY, actionState, isCurrentlyActive)
    }
}

// Fragment에서 연결
val touchCallback = ProductItemTouchCallback(
    adapter = productAdapter,
    onSwipeDelete = { product ->
        viewModel.deleteProduct(product)
        // Snackbar로 실행 취소 제공
        Snackbar.make(binding.root, "${product.name} 삭제됨", Snackbar.LENGTH_LONG)
            .setAction("실행 취소") { viewModel.restoreProduct(product) }
            .show()
    }
)
ItemTouchHelper(touchCallback).attachToRecyclerView(binding.recyclerView)
```

`ListAdapter`에 `moveItem(from, to)` 헬퍼를 추가합니다.

```kotlin
// ProductAdapter 내부에 추가
fun moveItem(from: Int, to: Int) {
    val current = currentList.toMutableList()
    val item = current.removeAt(from)
    current.add(to, item)
    submitList(current)
}
```

---

## 6. 주의사항 및 팁

### 리스트 불변성 유지
`submitList()`에 넘기는 리스트는 반드시 새 인스턴스여야 합니다. 기존 리스트를 `add()`나 `remove()`로 수정한 후 같은 참조를 넘기면 `AsyncListDiffer`가 "변경 없음"으로 판단합니다.

```kotlin
// 잘못된 예
val list = _products.value.toMutableList()
list.add(newProduct)
adapter.submitList(list) // 이전 currentList와 참조가 달라도, 내부 캐시 문제 발생 가능

// 올바른 예
adapter.submitList(_products.value + newProduct)
```

### `submitList(null)` vs `submitList(emptyList())`
`submitList(null)`은 리스트를 비우지만 `null`로 설정합니다. 이후 `currentList`가 `emptyList()`를 반환합니다. 명시적으로 비우려면 `submitList(emptyList())`를 사용하는 것이 의도가 더 명확합니다.

### `getItem(adapterPosition)` 사용 시 주의
`adapterPosition`은 `-1`을 반환할 수 있습니다 (아이템이 분리될 때). `bindingAdapterPosition`을 사용하거나, `-1` 체크를 추가하세요. API 29부터는 `bindingAdapterPosition` 사용이 권장됩니다.

### ConcatAdapter와 RecycledViewPool 공유
여러 어댑터가 같은 ViewType을 공유하는 경우 `RecycledViewPool`을 명시적으로 설정하면 ViewHolder를 재사용할 수 있어 성능이 향상됩니다.

```kotlin
val sharedPool = RecyclerView.RecycledViewPool()
binding.recyclerView.setRecycledViewPool(sharedPool)
binding.recyclerView.adapter = ConcatAdapter(
    ConcatAdapter.Config.Builder().setIsolateViewTypes(false).build(), // 타입 공유 허용
    adapter1, adapter2
)
```

### ItemTouchHelper와 Snackbar 조합
스와이프 삭제 후 실행 취소(Undo) 패턴은 사용성에서 필수입니다. `Snackbar`의 `setAction`에서 삭제된 아이템을 원래 위치에 복원하는 로직을 ViewModel에 구현하고, 복원 시 `submitList()`를 다시 호출하면 DiffUtil이 최소 애니메이션으로 아이템을 되살립니다.

---

## 7. 정리

| 도구 | 역할 |
|---|---|
| `DiffUtil.Callback` | 두 리스트의 차이를 계산하는 사용자 정의 규칙 |
| `AsyncListDiffer` | 백그라운드에서 diff를 자동 계산 |
| `ListAdapter` | `AsyncListDiffer`를 내장한 권장 기본 어댑터 |
| `ConcatAdapter` | 여러 어댑터를 하나로 합쳐 섹션 기반 목록 구현 |
| `ItemTouchHelper` | 스와이프·드래그 제스처 처리 |

RecyclerView는 Android UI의 심장부입니다. `notifyDataSetChanged()`를 제거하고 `ListAdapter`와 `DiffUtil`을 도입하는 것만으로도 리스트 업데이트 성능과 사용자 경험이 눈에 띄게 향상됩니다. 여기에 `ConcatAdapter`로 코드 모듈화, `ItemTouchHelper`로 풍부한 제스처를 추가하면 유지보수가 쉽고 성능도 뛰어난 RecyclerView를 완성할 수 있습니다.

---

## 참고 자료
- [androidx.recyclerview.widget 패키지 레퍼런스](https://developer.android.com/reference/kotlin/androidx/recyclerview/widget/package-summary)
- [ListAdapter API 레퍼런스](https://developer.android.com/reference/androidx/recyclerview/widget/ListAdapter)
- [ConcatAdapter API 레퍼런스](https://developer.android.com/reference/androidx/recyclerview/widget/ConcatAdapter)
