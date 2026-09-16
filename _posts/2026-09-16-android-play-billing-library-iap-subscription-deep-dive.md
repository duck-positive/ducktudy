---
layout: post
title: "Android Google Play Billing Library 심화: 인앱 결제·구독·소비성 상품 완전 정복"
date: 2026-09-16
categories: [android, flutter]
tags: [android, billing, iap, in-app-purchase, subscription, kotlin, google-play]
---

인앱 결제(In-App Purchase)는 현대 모바일 앱 수익화의 핵심 수단입니다. Google Play Billing Library(GPBL)는 Android 앱에서 안전하고 일관된 결제 경험을 제공하기 위해 Google이 공식 지원하는 라이브러리입니다. 이 글에서는 GPBL v7/v8을 기준으로, BillingClient 초기화부터 구매 확인, 구독 관리, 백엔드 검증까지 실전에서 바로 쓸 수 있는 수준으로 심층 분석합니다.

## 왜 Google Play Billing Library인가?

Android 앱을 Google Play에 출시하면서 디지털 상품을 판매하려면 **반드시** Google Play Billing Library를 사용해야 합니다. 직접 결제 처리(Stripe, 자체 SDK 등)는 Google Play 정책에 위배됩니다. 2026년 8월 31일부로 신규 앱 및 업데이트는 **Billing Library v8 이상**을 사용해야 합니다.

GPBL이 처리하는 핵심 기능은 다음과 같습니다:

- **일회성 상품(INAPP)**: 유료 앱 콘텐츠, 게임 아이템 등
- **소비성 상품(Consumable)**: 반복 구매 가능한 코인, 에너지 등
- **비소비성 상품(Non-consumable)**: 광고 제거, 영구 잠금 해제 등
- **구독(SUBS)**: 월정액, 연정액 반복 결제

---

## 1. 결제 라이프사이클 개요

GPBL의 구매 흐름은 단순해 보이지만, 각 단계에서 실수가 생기면 매출 손실이나 사용자 불만으로 직결됩니다.

```
앱 시작
  └─ BillingClient 초기화 & Google Play 연결
        └─ 상품 정보 조회 (queryProductDetailsAsync)
              └─ 구매 플로우 시작 (launchBillingFlow)
                    └─ onPurchasesUpdated 콜백
                          └─ 백엔드 구매 검증
                                └─ 권한 부여
                                      └─ 구매 확인 (acknowledgePurchase) ← 3일 이내 필수
```

**3일 규칙**: 구매 후 3일 이내에 `acknowledgePurchase()`를 호출하지 않으면 Google Play가 자동으로 환불 처리합니다.

---

## 2. 프로젝트 설정

`build.gradle.kts`에 의존성을 추가합니다:

```kotlin
dependencies {
    implementation("com.android.billingclient:billing:7.1.1")
    // Kotlin Coroutine 확장 (권장)
    implementation("com.android.billingclient:billing-ktx:7.1.1")
}
```

---

## 3. BillingClient 초기화 및 연결

BillingClient는 Google Play 결제 서비스와 통신하는 메인 인터페이스입니다. 앱 전체에서 **하나의 인스턴스**만 유지하는 것이 권장됩니다.

```kotlin
class BillingManager(
    private val context: Context,
    private val coroutineScope: CoroutineScope
) : PurchasesUpdatedListener {

    private lateinit var billingClient: BillingClient
    
    // 구매 상태 스트림
    private val _purchases = MutableStateFlow<List<Purchase>>(emptyList())
    val purchases: StateFlow<List<Purchase>> = _purchases.asStateFlow()
    
    // 연결 상태
    private val _isConnected = MutableStateFlow(false)
    val isConnected: StateFlow<Boolean> = _isConnected.asStateFlow()

    init {
        billingClient = BillingClient.newBuilder(context)
            .setListener(this)
            .enablePendingPurchases(
                PendingPurchasesParams.newBuilder()
                    .enableOneTimeProducts()
                    .enablePrepaidPlans()  // 구독 선불 플랜 지원
                    .build()
            )
            .enableAutoServiceReconnection()  // v7 신기능: 자동 재연결
            .build()

        startConnection()
    }

    private fun startConnection() {
        billingClient.startConnection(object : BillingClientStateListener {
            override fun onBillingSetupFinished(result: BillingResult) {
                if (result.responseCode == BillingClient.BillingResponseCode.OK) {
                    _isConnected.value = true
                    // 연결 성공 시 미처리 구매 조회
                    coroutineScope.launch { queryExistingPurchases() }
                }
            }

            override fun onBillingServiceDisconnected() {
                _isConnected.value = false
                // enableAutoServiceReconnection()이 있으면 자동으로 재시도
            }
        })
    }

    // PurchasesUpdatedListener 구현
    override fun onPurchasesUpdated(result: BillingResult, purchases: List<Purchase>?) {
        when (result.responseCode) {
            BillingClient.BillingResponseCode.OK -> {
                purchases?.let { handlePurchases(it) }
            }
            BillingClient.BillingResponseCode.USER_CANCELED -> {
                // 사용자가 직접 취소 — 오류 처리 불필요
            }
            BillingClient.BillingResponseCode.ITEM_ALREADY_OWNED -> {
                // 이미 소유한 상품 — 미확인 구매 복구 로직 필요
                coroutineScope.launch { queryExistingPurchases() }
            }
            else -> {
                // 기타 오류 처리
            }
        }
    }

    fun destroy() {
        billingClient.endConnection()
    }
}
```

---

## 4. 상품 정보 조회

구매 전 반드시 Google Play에서 최신 상품 정보(가격, 이름 등)를 조회해야 합니다. 하드코딩된 가격 표시는 사용자 지역 통화를 반영하지 못합니다.

```kotlin
// 상품 ID 정의 (Play Console에서 설정한 ID와 일치해야 함)
private val IN_APP_PRODUCTS = listOf("premium_upgrade", "coin_100")
private val SUBSCRIPTIONS = listOf("monthly_sub", "yearly_sub")

suspend fun queryProducts(): List<ProductDetails> {
    val productList = buildList {
        // 일회성 상품
        addAll(IN_APP_PRODUCTS.map { productId ->
            QueryProductDetailsParams.Product.newBuilder()
                .setProductId(productId)
                .setProductType(BillingClient.ProductType.INAPP)
                .build()
        })
        // 구독 상품
        addAll(SUBSCRIPTIONS.map { productId ->
            QueryProductDetailsParams.Product.newBuilder()
                .setProductId(productId)
                .setProductType(BillingClient.ProductType.SUBS)
                .build()
        })
    }

    val params = QueryProductDetailsParams.newBuilder()
        .setProductList(productList)
        .build()

    // billing-ktx의 코루틴 확장 사용
    val (billingResult, productDetailsList) = billingClient.queryProductDetails(params)

    if (billingResult.responseCode != BillingClient.BillingResponseCode.OK) {
        throw BillingException("상품 조회 실패: ${billingResult.debugMessage}")
    }

    return productDetailsList ?: emptyList()
}
```

조회된 `ProductDetails`에서 구독 상품의 가격 옵션을 파싱하는 방법:

```kotlin
fun extractSubscriptionOffers(productDetails: ProductDetails): List<OfferInfo> {
    val subscriptionOfferDetails = productDetails.subscriptionOfferDetails
        ?: return emptyList()

    return subscriptionOfferDetails.map { offerDetail ->
        val pricingPhases = offerDetail.pricingPhases.pricingPhaseList
        val freeTrialPhase = pricingPhases.firstOrNull { it.priceAmountMicros == 0L }
        val paidPhase = pricingPhases.lastOrNull { it.priceAmountMicros > 0L }

        OfferInfo(
            offerToken = offerDetail.offerToken,
            offerId = offerDetail.offerId,
            hasFreeTrial = freeTrialPhase != null,
            freeTrialDays = freeTrialPhase?.billingPeriod?.let { parseDays(it) } ?: 0,
            price = paidPhase?.formattedPrice ?: "",
            billingPeriod = paidPhase?.billingPeriod ?: ""
        )
    }
}

data class OfferInfo(
    val offerToken: String,
    val offerId: String?,
    val hasFreeTrial: Boolean,
    val freeTrialDays: Int,
    val price: String,
    val billingPeriod: String
)
```

---

## 5. 구매 플로우 실행

상품 정보를 얻은 뒤, 실제 결제 다이얼로그를 띄웁니다. `launchBillingFlow()`는 반드시 **Activity**에서 호출해야 합니다.

```kotlin
fun launchPurchase(
    activity: Activity,
    productDetails: ProductDetails,
    offerToken: String? = null,  // 구독의 경우 필수
    oldPurchaseToken: String? = null  // 업그레이드/다운그레이드 시
): BillingResult {

    val productDetailsParamsList = listOf(
        BillingFlowParams.ProductDetailsParams.newBuilder()
            .setProductDetails(productDetails)
            .apply {
                // 구독 상품인 경우 offerToken 필수
                offerToken?.let { setOfferToken(it) }
            }
            .build()
    )

    val billingFlowParams = BillingFlowParams.newBuilder()
        .setProductDetailsParamsList(productDetailsParamsList)
        .apply {
            // 구독 업그레이드/다운그레이드
            oldPurchaseToken?.let { token ->
                setSubscriptionUpdateParams(
                    BillingFlowParams.SubscriptionUpdateParams.newBuilder()
                        .setOldPurchaseToken(token)
                        .setSubscriptionReplacementMode(
                            // 즉시 변경 + 잔여 기간 비례 환급 (업그레이드)
                            BillingFlowParams.SubscriptionUpdateParams.ReplacementMode.WITH_TIME_PRORATION
                        )
                        .build()
                )
            }
            // EU 개인화 가격 표시 여부
            setIsOfferPersonalized(false)
            // 사용자 식별자 (서버 검증용, 개인정보 비포함)
            setObfuscatedAccountId(generateObfuscatedAccountId())
        }
        .build()

    return billingClient.launchBillingFlow(activity, billingFlowParams)
}
```

---

## 6. 구매 처리 및 권한 부여

`onPurchasesUpdated()`에서 호출되는 핵심 처리 로직입니다.

```kotlin
private fun handlePurchases(purchases: List<Purchase>) {
    coroutineScope.launch {
        purchases.forEach { purchase ->
            processPurchase(purchase)
        }
    }
}

private suspend fun processPurchase(purchase: Purchase) {
    // PENDING 상태면 아직 처리 중 (현금 결제 등)
    if (purchase.purchaseState != Purchase.PurchaseState.PURCHASED) return

    // 1단계: 백엔드에서 구매 검증 (필수!)
    val isValid = verifyPurchaseOnServer(
        purchaseToken = purchase.purchaseToken,
        products = purchase.products
    )

    if (!isValid) {
        // 검증 실패: 권한 부여 금지, 로그 기록
        return
    }

    // 2단계: 사용자에게 권한 부여
    grantEntitlement(purchase.products)

    // 3단계: 구매 확인 (3일 이내 필수!)
    if (!purchase.isAcknowledged) {
        val acknowledgePurchaseParams = AcknowledgePurchaseParams.newBuilder()
            .setPurchaseToken(purchase.purchaseToken)
            .build()

        val ackResult = billingClient.acknowledgePurchase(acknowledgePurchaseParams)
        if (ackResult.responseCode != BillingClient.BillingResponseCode.OK) {
            // 확인 실패 시 재시도 로직 필요
        }
    }

    // 소비성 상품의 경우 consumePurchase() 호출
    if (purchase.products.any { it in CONSUMABLE_PRODUCTS }) {
        val consumeParams = ConsumeParams.newBuilder()
            .setPurchaseToken(purchase.purchaseToken)
            .build()
        billingClient.consumePurchase(consumeParams)
    }
}
```

---

## 7. 앱 재시작 시 미처리 구매 복구

앱 크래시, 네트워크 오류 등으로 처리되지 않은 구매를 앱 재시작 시 복구하는 것이 필수입니다.

```kotlin
private suspend fun queryExistingPurchases() {
    // 일회성 상품 조회
    val inAppResult = billingClient.queryPurchasesAsync(
        QueryPurchasesParams.newBuilder()
            .setProductType(BillingClient.ProductType.INAPP)
            .build()
    )

    // 구독 조회
    val subsResult = billingClient.queryPurchasesAsync(
        QueryPurchasesParams.newBuilder()
            .setProductType(BillingClient.ProductType.SUBS)
            .build()
    )

    val allPurchases = buildList {
        if (inAppResult.billingResult.responseCode == BillingClient.BillingResponseCode.OK) {
            addAll(inAppResult.purchasesList)
        }
        if (subsResult.billingResult.responseCode == BillingClient.BillingResponseCode.OK) {
            addAll(subsResult.purchasesList)
        }
    }

    _purchases.value = allPurchases
    handlePurchases(allPurchases)
}
```

---

## 8. 백엔드 서버 검증 (필수 보안 단계)

클라이언트에서만 구매를 검증하면 위변조 공격에 취약합니다. **Google Play Developer API**를 통한 서버 검증이 필수입니다.

```kotlin
// 클라이언트: 서버로 검증 요청
private suspend fun verifyPurchaseOnServer(
    purchaseToken: String,
    products: List<String>
): Boolean = withContext(Dispatchers.IO) {
    try {
        val response = apiService.verifyPurchase(
            VerifyPurchaseRequest(
                purchaseToken = purchaseToken,
                productIds = products,
                packageName = BuildConfig.APPLICATION_ID
            )
        )
        response.isValid
    } catch (e: Exception) {
        false
    }
}
```

서버 측 검증 로직 (예시: Node.js/TypeScript):

```kotlin
// 서버 예시는 Kotlin으로 표현한 의사 코드
// 실제는 각 서버 언어의 Google API Client를 사용
suspend fun verifyPurchaseOnPlayStore(
    packageName: String,
    productId: String,
    purchaseToken: String
): PurchaseVerificationResult {
    // Google Play Developer API v3 호출
    // GET https://androidpublisher.googleapis.com/androidpublisher/v3/
    //     applications/{packageName}/purchases/products/{productId}/tokens/{token}
    
    val purchaseData = googlePlayApi.purchases.products.get(
        packageName = packageName,
        productId = productId,
        token = purchaseToken
    )
    
    return PurchaseVerificationResult(
        isValid = purchaseData.purchaseState == 0,  // 0 = 구매 완료
        purchaseTimeMillis = purchaseData.purchaseTimeMillis,
        orderId = purchaseData.orderId
    )
}
```

---

## 9. 구독 업그레이드/다운그레이드

구독 변경 시 `ReplacementMode`에 따라 사용자 경험이 달라집니다:

| 모드 | 동작 | 적합한 경우 |
|------|------|-------------|
| `WITH_TIME_PRORATION` | 즉시 변경, 잔여 기간 다음 청구에 적용 | 업그레이드 기본 |
| `CHARGE_PRORATED_PRICE` | 즉시 변경, 차액만 즉시 청구 | 업그레이드 + 청구일 유지 |
| `CHARGE_FULL_PRICE` | 즉시 변경, 전액 즉시 청구 | 새 청구 주기 시작 |
| `DEFERRED` | 현재 주기 만료 후 변경 | 다운그레이드 |
| `WITHOUT_PRORATION` | 즉시 변경, 다음 갱신 시 새 요금 | 무료 체험 유지하며 업그레이드 |

---

## 10. 구독 결제 실패 처리 (InAppMessage)

결제 실패 시 Google Play가 자동으로 인앱 메시지를 표시하도록 연동할 수 있습니다:

```kotlin
fun showSubscriptionStatusMessage(activity: Activity) {
    val inAppMessageParams = InAppMessageParams.newBuilder()
        .addInAppMessageCategoryToShow(InAppMessageParams.InAppMessageCategoryId.TRANSACTIONAL)
        .build()

    billingClient.showInAppMessages(
        activity,
        inAppMessageParams
    ) { inAppMessageResult ->
        if (inAppMessageResult.responseCode == 
            InAppMessageResult.InAppMessageResponseCode.SUBSCRIPTION_STATUS_UPDATED) {
            // 구독 상태 변경됨 — 권한 재확인 필요
            val updatedToken = inAppMessageResult.purchaseToken
            coroutineScope.launch { queryExistingPurchases() }
        }
    }
}
```

---

## 주의사항 및 실전 팁

### 1. BillingClient는 싱글톤으로 관리하세요

```kotlin
// Hilt를 사용한 싱글톤 주입 예시
@Module
@InstallIn(SingletonComponent::class)
object BillingModule {
    @Provides
    @Singleton
    fun provideBillingManager(
        @ApplicationContext context: Context,
        @ApplicationScope scope: CoroutineScope
    ): BillingManager = BillingManager(context, scope)
}
```

### 2. 테스트 계정 활용

Play Console에서 테스트 계정을 추가하면 실제 결제 없이 전체 흐름을 테스트할 수 있습니다. 단, **라이선스 테스터 계정**은 실제 구글 계정이어야 하며 기기에 로그인되어 있어야 합니다.

### 3. 구독 구독 취소 시 처리

구독 취소는 즉시 만료가 아닌 **현재 결제 주기 종료 시** 만료됩니다. `expiryTimeMillis`를 기준으로 권한을 관리해야 합니다.

### 4. RTDN(Real-Time Developer Notifications)

서버 측에서는 Google Cloud Pub/Sub을 통한 RTDN을 구독해 구독 상태 변화(갱신, 취소, 만료 등)를 실시간으로 처리해야 합니다.

### 5. 오프라인 구매 처리

네트워크 오프라인 상태에서도 구매가 완료될 수 있습니다 (PENDING 상태). `onPurchasesUpdated()`뿐 아니라 앱 포그라운드 진입 시마다 `queryPurchasesAsync()`를 호출하는 패턴이 필수입니다.

---

## 마무리

Google Play Billing Library는 단순한 결제 SDK가 아닙니다. 구매 상태 관리, 서버 검증, 구독 생명주기, 오류 복구까지 포함한 전체 수익화 아키텍처를 설계해야 합니다. 특히 **3일 확인 규칙**, **서버 검증 필수**, **앱 재시작 시 미처리 구매 복구** 세 가지는 매출 손실을 막는 핵심 원칙입니다. 2026년 8월 기준 Billing Library v8로의 마이그레이션도 서둘러 준비하세요.

## 참고 자료
- [Google Play Billing Library 통합 가이드](https://developer.android.com/google/play/billing/integrate)
- [구독 관리 완전 가이드](https://developer.android.com/google/play/billing/subscriptions)
- [Google Play Billing Library 릴리즈 노트](https://developer.android.com/google/play/billing/release-notes)
- [Play Billing Library v7 마이그레이션 가이드](https://developer.android.com/google/play/billing/migrate-gpblv7)
