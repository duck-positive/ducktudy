---
layout: post
title: "Android Scoped Storage와 MediaStore API 심화: 사진·동영상·파일 완전 정복"
date: 2026-09-06
categories: [android]
tags: [android, scoped-storage, mediastore, photo-picker, content-resolver, storage, kotlin]
---

Android 10(API 29)부터 도입된 **Scoped Storage**는 앱이 기기의 파일 시스템에 접근하는 방식을 근본적으로 바꿨습니다. 기존에 `READ_EXTERNAL_STORAGE` 권한 하나로 SD 카드 전체를 뒤질 수 있던 시절은 끝났고, 이제는 `MediaStore` API와 `Storage Access Framework(SAF)`를 통해 시스템이 중재하는 방식으로만 공유 파일에 접근할 수 있습니다. 이 변화는 처음에 귀찮게 느껴지지만, 사용자 프라이버시와 앱 안정성 측면에서 명백한 이점이 있습니다.

이 글에서는 Scoped Storage의 핵심 개념부터 실전 MediaStore 쿼리, Photo Picker 통합, 파일 생성/삭제, 배치 작업까지 Kotlin 코드와 함께 완전히 정복합니다.

---

## 1. Scoped Storage란 무엇인가

Android의 외부 저장소(External Storage)는 크게 두 영역으로 나뉩니다.

| 영역 | 설명 | 권한 필요 여부 |
|---|---|---|
| **앱 전용 디렉토리** | `getExternalFilesDir()` 반환 경로 | 불필요 |
| **공유 저장소** | `MediaStore` API로 접근하는 사진·음악·동영상·문서 | 필요 (제한적) |

Scoped Storage 이전에는 앱이 `/sdcard/` 전체에 직접 경로로 접근할 수 있었습니다. Android 10부터는 공유 저장소 접근이 `MediaStore` 또는 SAF(Storage Access Framework)를 통해서만 허용되며, 앱이 직접 생성하지 않은 파일을 수정·삭제하려면 사용자의 명시적 동의(`RecoverableSecurityException`)가 필요합니다.

Android 버전별 권한 변화:

- **Android 9 이하**: `READ_EXTERNAL_STORAGE`로 전체 저장소 읽기 가능
- **Android 10(Q)**: Scoped Storage 선택 적용 (`requestLegacyExternalStorage` 플래그로 우회 가능)
- **Android 11(R)**: Scoped Storage 강제 적용, `MANAGE_EXTERNAL_STORAGE`는 특수 앱만 허용
- **Android 13(T)**: 세분화된 미디어 권한 도입 (`READ_MEDIA_IMAGES`, `READ_MEDIA_VIDEO`, `READ_MEDIA_AUDIO`)
- **Android 14(U)**: Photo Picker 의무화 강화, 부분 접근 권한(`READ_MEDIA_VISUAL_USER_SELECTED`) 추가

---

## 2. 왜 Scoped Storage가 필요한가

### 기존 방식의 문제점

`READ_EXTERNAL_STORAGE` 하나로 SD 카드 전체에 접근하는 방식은 심각한 프라이버시 문제를 내포했습니다. 날씨 앱이 연락처 사진에 접근하거나, 게임 앱이 은행 앱의 로컬 캐시를 읽는 것이 기술적으로 가능했습니다.

### Scoped Storage의 이점

1. **프라이버시 보호**: 앱은 자신이 만든 파일과, 사용자가 명시적으로 선택한 파일에만 접근 가능
2. **권한 최소화**: 불필요한 스토리지 권한 요청 감소
3. **데이터 정리**: 앱 삭제 시 앱이 생성한 파일도 함께 정리됨
4. **시스템 안정성**: 잘못된 파일 조작으로 인한 시스템 손상 방지

---

## 3. 권한 선언과 요청

`AndroidManifest.xml`에 필요한 권한을 선언합니다:

```xml
<!-- Android 13+ 세분화 권한 -->
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
<uses-permission android:name="android.permission.READ_MEDIA_AUDIO" />

<!-- Android 14+ 부분 접근 권한 -->
<uses-permission android:name="android.permission.READ_MEDIA_VISUAL_USER_SELECTED" />

<!-- Android 12 이하 폴백 -->
<uses-permission
    android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="32" />

<!-- EXIF 위치 정보 접근 -->
<uses-permission android:name="android.permission.ACCESS_MEDIA_LOCATION" />
```

런타임 권한 요청 로직:

```kotlin
class StoragePermissionManager(private val activity: ComponentActivity) {

    private val requiredPermissions: Array<String>
        get() = when {
            Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE -> arrayOf(
                READ_MEDIA_IMAGES,
                READ_MEDIA_VIDEO,
                READ_MEDIA_VISUAL_USER_SELECTED
            )
            Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU -> arrayOf(
                READ_MEDIA_IMAGES,
                READ_MEDIA_VIDEO
            )
            else -> arrayOf(READ_EXTERNAL_STORAGE)
        }

    private val permissionLauncher = activity.registerForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { results ->
        val allGranted = results.values.all { it }
        val partiallyGranted = Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE &&
            results[READ_MEDIA_VISUAL_USER_SELECTED] == true

        when {
            allGranted -> onFullAccess()
            partiallyGranted -> onPartialAccess()
            else -> onDenied()
        }
    }

    fun requestPermissions() = permissionLauncher.launch(requiredPermissions)

    private fun onFullAccess() { /* 전체 미디어 접근 허용 */ }
    private fun onPartialAccess() { /* 선택한 미디어만 접근 허용 */ }
    private fun onDenied() { /* 접근 거부 */ }
}
```

---

## 4. Photo Picker — 가장 권장되는 방식

Android 13+에서 도입되고 이전 버전(API 21+)에도 Google Play 서비스를 통해 백포팅된 **Photo Picker**는 권한 요청 없이 사용자가 직접 선택한 미디어에만 접근할 수 있는 가장 프라이버시 친화적인 방법입니다.

```kotlin
class PhotoPickerActivity : AppCompatActivity() {

    // 단일 이미지 선택
    private val pickSingleImage = registerForActivityResult(
        ActivityResultContracts.PickVisualMedia()
    ) { uri: Uri? ->
        uri?.let { handleSelectedMedia(it) }
    }

    // 복수 이미지/동영상 선택 (최대 9개)
    private val pickMultipleMedia = registerForActivityResult(
        ActivityResultContracts.PickMultipleVisualMedia(maxItems = 9)
    ) { uris: List<Uri> ->
        if (uris.isNotEmpty()) {
            handleMultipleMedia(uris)
        }
    }

    fun launchImagePicker() {
        pickSingleImage.launch(
            PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)
        )
    }

    fun launchVideoOrImagePicker() {
        pickMultipleMedia.launch(
            PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageAndVideo)
        )
    }

    private fun handleSelectedMedia(uri: Uri) {
        // URI가 일시적으로만 유효함 — 장기 보관 시 persistableUriPermission 필요
        contentResolver.takePersistableUriPermission(
            uri,
            Intent.FLAG_GRANT_READ_URI_PERMISSION
        )

        // 썸네일 로딩 (Coil 사용 예시)
        binding.imageView.load(uri) {
            crossfade(true)
            size(400, 400)
        }
    }

    private fun handleMultipleMedia(uris: List<Uri>) {
        // URI 목록을 ViewModel로 전달
        viewModel.onMediaSelected(uris)
    }
}
```

---

## 5. MediaStore API — 기기 미디어 쿼리

앱이 기기에 있는 전체 미디어 라이브러리를 탐색해야 할 때 `MediaStore`를 사용합니다.

### 5-1. 이미지 쿼리

```kotlin
data class MediaImage(
    val id: Long,
    val name: String,
    val contentUri: Uri,
    val dateModified: Long,
    val size: Long,
    val mimeType: String,
    val width: Int,
    val height: Int
)

class MediaRepository(private val context: Context) {

    fun queryImages(): List<MediaImage> {
        val images = mutableListOf<MediaImage>()

        val collection = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            MediaStore.Images.Media.getContentUri(MediaStore.VOLUME_EXTERNAL)
        } else {
            MediaStore.Images.Media.EXTERNAL_CONTENT_URI
        }

        val projection = arrayOf(
            MediaStore.Images.Media._ID,
            MediaStore.Images.Media.DISPLAY_NAME,
            MediaStore.Images.Media.DATE_MODIFIED,
            MediaStore.Images.Media.SIZE,
            MediaStore.Images.Media.MIME_TYPE,
            MediaStore.Images.Media.WIDTH,
            MediaStore.Images.Media.HEIGHT
        )

        val sortOrder = "${MediaStore.Images.Media.DATE_MODIFIED} DESC"
        val selection = "${MediaStore.Images.Media.SIZE} > 0"

        context.contentResolver.query(
            collection,
            projection,
            selection,
            null,
            sortOrder
        )?.use { cursor ->
            // 컬럼 인덱스를 미리 캐싱하면 성능 향상
            val idColumn = cursor.getColumnIndexOrThrow(MediaStore.Images.Media._ID)
            val nameColumn = cursor.getColumnIndexOrThrow(MediaStore.Images.Media.DISPLAY_NAME)
            val dateColumn = cursor.getColumnIndexOrThrow(MediaStore.Images.Media.DATE_MODIFIED)
            val sizeColumn = cursor.getColumnIndexOrThrow(MediaStore.Images.Media.SIZE)
            val mimeColumn = cursor.getColumnIndexOrThrow(MediaStore.Images.Media.MIME_TYPE)
            val widthColumn = cursor.getColumnIndexOrThrow(MediaStore.Images.Media.WIDTH)
            val heightColumn = cursor.getColumnIndexOrThrow(MediaStore.Images.Media.HEIGHT)

            while (cursor.moveToNext()) {
                val id = cursor.getLong(idColumn)
                val contentUri = ContentUris.withAppendedId(collection, id)

                images += MediaImage(
                    id = id,
                    name = cursor.getString(nameColumn),
                    contentUri = contentUri,
                    dateModified = cursor.getLong(dateColumn),
                    size = cursor.getLong(sizeColumn),
                    mimeType = cursor.getString(mimeColumn),
                    width = cursor.getInt(widthColumn),
                    height = cursor.getInt(heightColumn)
                )
            }
        }

        return images
    }

    // Flow 기반 반응형 쿼리 — MediaStore 변경 시 자동 갱신
    fun observeImages(): Flow<List<MediaImage>> = callbackFlow {
        val observer = object : ContentObserver(null) {
            override fun onChange(selfChange: Boolean) {
                trySend(queryImages())
            }
        }

        context.contentResolver.registerContentObserver(
            MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
            true,
            observer
        )

        send(queryImages()) // 초기값 전송

        awaitClose {
            context.contentResolver.unregisterContentObserver(observer)
        }
    }
}
```

### 5-2. 파일 생성 — IS_PENDING 패턴

새 이미지를 저장할 때는 `IS_PENDING` 플래그를 활용하여 파일이 완전히 쓰여지기 전에 다른 앱이 접근하는 것을 차단합니다:

```kotlin
suspend fun saveImageToGallery(
    context: Context,
    bitmap: Bitmap,
    displayName: String
): Uri? = withContext(Dispatchers.IO) {

    val contentValues = ContentValues().apply {
        put(MediaStore.Images.Media.DISPLAY_NAME, "$displayName.jpg")
        put(MediaStore.Images.Media.MIME_TYPE, "image/jpeg")
        put(MediaStore.Images.Media.DATE_ADDED, System.currentTimeMillis() / 1000)
        put(MediaStore.Images.Media.DATE_MODIFIED, System.currentTimeMillis() / 1000)

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            // Pictures/AppName/ 하위에 저장
            put(MediaStore.Images.Media.RELATIVE_PATH, "Pictures/MyApp/")
            // 쓰기 중인 파일임을 표시 — 다른 앱이 볼 수 없음
            put(MediaStore.Images.Media.IS_PENDING, 1)
        }
    }

    val resolver = context.contentResolver
    val collection = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        MediaStore.Images.Media.getContentUri(MediaStore.VOLUME_EXTERNAL_PRIMARY)
    } else {
        MediaStore.Images.Media.EXTERNAL_CONTENT_URI
    }

    val uri = resolver.insert(collection, contentValues) ?: return@withContext null

    try {
        resolver.openOutputStream(uri)?.use { outputStream ->
            bitmap.compress(Bitmap.CompressFormat.JPEG, 95, outputStream)
        }

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            // 쓰기 완료 — IS_PENDING 해제
            contentValues.clear()
            contentValues.put(MediaStore.Images.Media.IS_PENDING, 0)
            resolver.update(uri, contentValues, null, null)
        }

        uri
    } catch (e: Exception) {
        // 실패 시 삽입된 레코드 삭제
        resolver.delete(uri, null, null)
        null
    }
}
```

---

## 6. 배치 작업 — 사용자 동의 처리

Android 11+에서 앱이 소유하지 않은 미디어 파일을 수정·삭제할 때는 `MediaStore.createDeleteRequest()` 또는 `createWriteRequest()`를 사용하여 사용자의 동의를 받아야 합니다.

```kotlin
class MediaBatchOperator(private val activity: AppCompatActivity) {

    private val intentSenderLauncher = activity.registerForActivityResult(
        ActivityResultContracts.StartIntentSenderForResult()
    ) { result ->
        if (result.resultCode == Activity.RESULT_OK) {
            // 사용자가 동의함 — 실제 삭제/수정 진행
            onUserApproved()
        }
    }

    @RequiresApi(Build.VERSION_CODES.R)
    fun requestDeleteImages(imageUris: List<Uri>) {
        val pendingIntent = MediaStore.createDeleteRequest(
            activity.contentResolver,
            imageUris
        )
        intentSenderLauncher.launch(
            IntentSenderRequest.Builder(pendingIntent.intentSender).build()
        )
    }

    @RequiresApi(Build.VERSION_CODES.R)
    fun requestModifyImages(imageUris: List<Uri>) {
        val pendingIntent = MediaStore.createWriteRequest(
            activity.contentResolver,
            imageUris
        )
        intentSenderLauncher.launch(
            IntentSenderRequest.Builder(pendingIntent.intentSender).build()
        )
    }

    // Android 10 이하 — RecoverableSecurityException 처리
    fun deleteImageLegacy(uri: Uri) {
        try {
            activity.contentResolver.delete(uri, null, null)
        } catch (e: SecurityException) {
            val recoverableException = e as? RecoverableSecurityException ?: throw e
            intentSenderLauncher.launch(
                IntentSenderRequest.Builder(
                    recoverableException.userAction.actionIntent.intentSender
                ).build()
            )
        }
    }

    private fun onUserApproved() {
        // 사용자 동의 후 처리 로직
    }
}
```

---

## 7. EXIF 위치 정보 접근

사진의 GPS 좌표를 읽으려면 `ACCESS_MEDIA_LOCATION` 권한과 함께 특수한 URI가 필요합니다:

```kotlin
@RequiresApi(Build.VERSION_CODES.Q)
fun getImageLocation(context: Context, imageUri: Uri): Pair<Double, Double>? {
    // ACCESS_MEDIA_LOCATION 없이는 위치 정보가 제거됨
    val locationUri = MediaStore.setRequireOriginal(imageUri)

    return try {
        context.contentResolver.openInputStream(locationUri)?.use { stream ->
            val exif = ExifInterface(stream)
            val latLong = FloatArray(2)
            if (exif.getLatLong(latLong)) {
                Pair(latLong[0].toDouble(), latLong[1].toDouble())
            } else null
        }
    } catch (e: UnsupportedOperationException) {
        // 클라우드 백업 이미지 등 원본에 직접 접근 불가한 경우
        null
    }
}
```

---

## 8. Storage Access Framework — 문서 파일 접근

이미지·동영상이 아닌 일반 문서(PDF, TXT, JSON 등)는 SAF를 통해 접근합니다:

```kotlin
class DocumentHandler(private val activity: AppCompatActivity) {

    // 파일 열기
    private val openDocument = activity.registerForActivityResult(
        ActivityResultContracts.OpenDocument()
    ) { uri: Uri? ->
        uri?.let { readDocument(it) }
    }

    // 파일 만들기
    private val createDocument = activity.registerForActivityResult(
        ActivityResultContracts.CreateDocument("application/json")
    ) { uri: Uri? ->
        uri?.let { writeDocument(it) }
    }

    // 디렉토리 선택 (영구 접근권)
    private val openDirectory = activity.registerForActivityResult(
        ActivityResultContracts.OpenDocumentTree()
    ) { treeUri: Uri? ->
        treeUri?.let { grantPersistentAccess(it) }
    }

    fun pickPdfOrText() {
        openDocument.launch(arrayOf("application/pdf", "text/plain"))
    }

    fun createJsonFile() {
        createDocument.launch("export_${System.currentTimeMillis()}.json")
    }

    private fun readDocument(uri: Uri) {
        activity.contentResolver.openInputStream(uri)?.bufferedReader()?.use { reader ->
            val content = reader.readText()
            // 문서 내용 처리
        }
    }

    private fun writeDocument(uri: Uri) {
        activity.contentResolver.openOutputStream(uri)?.bufferedWriter()?.use { writer ->
            writer.write("""{"timestamp": ${System.currentTimeMillis()}}""")
        }
    }

    private fun grantPersistentAccess(treeUri: Uri) {
        // 앱 재시작 후에도 유지되는 영구 접근권 획득
        activity.contentResolver.takePersistableUriPermission(
            treeUri,
            Intent.FLAG_GRANT_READ_URI_PERMISSION or Intent.FLAG_GRANT_WRITE_URI_PERMISSION
        )
        // DocumentFile API로 디렉토리 내 파일 조작 가능
        val documentTree = DocumentFile.fromTreeUri(activity, treeUri)
        documentTree?.listFiles()?.forEach { file ->
            println("${file.name}: ${file.length()} bytes")
        }
    }
}
```

---

## 9. 주의사항과 실전 팁

### 팁 1: MediaStore 버전 캐싱

MediaStore는 데이터가 바뀌면 버전 번호가 변경됩니다. 이를 활용하면 불필요한 전체 쿼리를 줄일 수 있습니다:

```kotlin
fun hasMediaStoreChanged(context: Context, cachedVersion: String?): Boolean {
    val currentVersion = MediaStore.getVersion(context)
    return currentVersion != cachedVersion
}
```

### 팁 2: 썸네일 로딩 최적화

Android 10+에서는 `ContentResolver.loadThumbnail()`을 사용하세요. 기존의 `MediaStore.Images.Thumbnails.getThumbnail()`보다 훨씬 효율적입니다:

```kotlin
@RequiresApi(Build.VERSION_CODES.Q)
fun loadThumbnail(context: Context, uri: Uri, size: Size): Bitmap? {
    return try {
        context.contentResolver.loadThumbnail(uri, size, null)
    } catch (e: IOException) {
        null
    }
}
```

### 팁 3: MANAGE_EXTERNAL_STORAGE는 꼭 필요할 때만

파일 관리자, 백업 앱, 바이러스 백신 같은 특수 목적 앱이 아니라면 `MANAGE_EXTERNAL_STORAGE`를 신청하지 마세요. Google Play 정책상 사용 목적을 명시해야 하며, 심사에서 반려될 수 있습니다.

### 팁 4: `IS_PENDING` 처리 누락 주의

파일 저장 중 앱이 크래시되면 `IS_PENDING=1` 상태의 고아 레코드가 남습니다. 앱 시작 시 자신이 생성한 미완료 파일을 정리하는 로직을 추가하세요:

```kotlin
fun cleanupPendingFiles(context: Context) {
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.Q) return

    val selection = "${MediaStore.Images.Media.IS_PENDING} = 1"
    val projection = arrayOf(MediaStore.Images.Media._ID, MediaStore.Images.Media.DISPLAY_NAME)

    context.contentResolver.query(
        MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
        projection, selection, null, null
    )?.use { cursor ->
        while (cursor.moveToNext()) {
            val id = cursor.getLong(cursor.getColumnIndexOrThrow(MediaStore.Images.Media._ID))
            val uri = ContentUris.withAppendedId(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, id)
            context.contentResolver.delete(uri, null, null)
        }
    }
}
```

### 팁 5: FileProvider를 통한 앱 간 파일 공유

앱 전용 저장소의 파일을 다른 앱과 공유할 때는 `FileProvider`를 사용합니다. 직접 경로 노출은 Android 7.0부터 `FileUriExposedException`을 발생시킵니다:

```kotlin
val file = File(context.filesDir, "export.pdf")
val uri = FileProvider.getUriForFile(
    context,
    "${context.packageName}.provider",
    file
)
val shareIntent = Intent(Intent.ACTION_SEND).apply {
    type = "application/pdf"
    putExtra(Intent.EXTRA_STREAM, uri)
    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
}
context.startActivity(Intent.createChooser(shareIntent, "파일 공유"))
```

---

## 10. 정리

| 시나리오 | 권장 방법 |
|---|---|
| 갤러리에서 사진 선택 | Photo Picker (PickVisualMedia) |
| 기기 전체 미디어 탐색 | MediaStore + ContentResolver |
| 갤러리에 사진 저장 | MediaStore.insert + IS_PENDING |
| 타인의 미디어 삭제/수정 | createDeleteRequest / createWriteRequest |
| PDF·문서 열기 | SAF (OpenDocument) |
| 앱 파일을 외부와 공유 | FileProvider |

Scoped Storage는 처음에는 장벽처럼 느껴지지만, `MediaStore`와 `Photo Picker`를 제대로 이해하면 오히려 더 명확하고 안전한 파일 처리 코드를 작성할 수 있습니다. 특히 Photo Picker는 권한 요청조차 필요 없어 사용자 거부 처리가 사라지고 UX가 단순해지는 이점이 있습니다. 앞으로 신규 앱 개발 시에는 Photo Picker를 우선 검토하고, 더 세밀한 제어가 필요한 경우에만 MediaStore로 내려가는 전략을 권장합니다.

---

## 참고 자료
- [MediaStore로 공유 저장소 미디어 파일 접근 (Android Developers)](https://developer.android.com/training/data-storage/shared/media)
- [Android Storage 사용 사례와 모범 사례 (Android Developers)](https://developer.android.com/training/data-storage/use-cases)
- [Photo Picker 통합 가이드 (Android Developers)](https://developer.android.com/training/data-storage/shared/photopicker)
- [데이터 및 파일 저장소 개요 (Android Developers)](https://developer.android.com/training/data-storage)
