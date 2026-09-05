---
layout: post
title: "Android ContentProvider 심화: URI 권한 위임·FileProvider·BatchOperation으로 앱 간 데이터 공유 완전 정복"
date: 2026-09-05
categories: [android]
tags: [android, contentprovider, fileprovider, contentresolver, uri, kotlin, jetpack]
---

## ContentProvider란 무엇인가

**ContentProvider**는 Android의 네 가지 핵심 컴포넌트(Activity, Service, BroadcastReceiver, ContentProvider) 중 하나로, 구조화된 데이터를 다른 앱에 **안전하게 노출**하기 위한 표준 인터페이스입니다. Android 보안 모델에서 각 앱은 별개의 프로세스와 샌드박스를 가지므로, 앱 간 직접 파일 접근이나 DB 공유는 불가능합니다. ContentProvider는 이 격벽을 유지하면서 **URI 기반 추상화 레이어**를 제공해 데이터를 안전하게 교환할 수 있게 합니다.

ContentProvider의 데이터 접근 모델은 `content://authority/path/id` 형식의 URI로 표현됩니다. 예를 들어 `content://com.example.myapp.provider/notes/42`는 `com.example.myapp.provider` 앱의 `notes` 테이블에서 id가 42인 레코드를 가리킵니다. 이 URI를 ContentResolver가 해석해 적절한 ContentProvider를 찾아 연결하는 구조입니다.

Android 시스템 자체도 연락처(`ContactsContract`), 캘린더(`CalendarContract`), 미디어 스토어(`MediaStore`) 등 핵심 데이터를 ContentProvider로 노출합니다. 앱이 `READ_CONTACTS` 권한을 가지면 ContactsProvider에 ContentResolver로 쿼리해 연락처를 읽는 것이 대표적인 사례입니다.

## 왜 필요한가: 앱 간 데이터 공유의 보안 모델

파일이나 DB를 직접 공유하는 방법 대신 ContentProvider를 사용해야 하는 이유는 크게 세 가지입니다.

**1. URI 기반 임시 권한 위임 (Temporary URI Permission)**  
ContentProvider는 `FLAG_GRANT_READ_URI_PERMISSION` / `FLAG_GRANT_WRITE_URI_PERMISSION` 플래그를 인텐트에 담아 특정 URI에 대한 접근 권한을 일시적으로 위임할 수 있습니다. 수신 앱이 해당 URI를 Intent Result로 받으면 권한이 자동 부여되고, Activity가 종료되면 자동으로 회수됩니다. 복잡한 권한 선언 없이 정밀한 접근 제어가 가능합니다.

**2. `file://` URI의 위험 제거**  
Android 7.0(API 24)부터 `file://` URI를 앱 외부에 노출하면 `FileUriExposedException`이 발생합니다. `FileProvider`(ContentProvider의 서브클래스)를 사용하면 내부 파일을 `content://` URI로 안전하게 변환해 공유할 수 있습니다.

**3. 원자적 배치 연산 (Batch Operation)**  
`ContentProviderOperation`을 활용하면 여러 삽입·수정·삭제를 하나의 트랜잭션으로 묶어 원자적으로 실행할 수 있습니다. 네트워크에서 받은 대량 데이터를 DB에 동기화할 때 중간 실패 시 롤백이 보장됩니다.

## 실제 구현 예제

### 예제 1: 커스텀 ContentProvider 구현 (Kotlin)

노트 앱의 데이터를 외부 앱에 노출하는 ContentProvider를 구현합니다.

```kotlin
// NoteContract.kt — URI 상수 정의
object NoteContract {
    const val AUTHORITY = "com.example.noteapp.provider"
    val BASE_URI: Uri = Uri.parse("content://$AUTHORITY")

    object Notes : BaseColumns {
        val CONTENT_URI: Uri = Uri.withAppendedPath(BASE_URI, "notes")
        const val TABLE_NAME = "notes"
        const val COLUMN_TITLE = "title"
        const val COLUMN_BODY = "body"
        const val COLUMN_CREATED_AT = "created_at"

        const val MIME_TYPE_DIR =
            "vnd.android.cursor.dir/vnd.$AUTHORITY.notes"
        const val MIME_TYPE_ITEM =
            "vnd.android.cursor.item/vnd.$AUTHORITY.notes"
    }
}

// NoteProvider.kt
class NoteProvider : ContentProvider() {

    private lateinit var dbHelper: NoteDbHelper

    private val uriMatcher = UriMatcher(UriMatcher.NO_MATCH).apply {
        addURI(NoteContract.AUTHORITY, "notes",      URI_NOTES)
        addURI(NoteContract.AUTHORITY, "notes/#",   URI_NOTE_ID)
    }

    override fun onCreate(): Boolean {
        dbHelper = NoteDbHelper(context!!)
        return true
    }

    override fun getType(uri: Uri): String = when (uriMatcher.match(uri)) {
        URI_NOTES   -> NoteContract.Notes.MIME_TYPE_DIR
        URI_NOTE_ID -> NoteContract.Notes.MIME_TYPE_ITEM
        else        -> throw IllegalArgumentException("Unknown URI: $uri")
    }

    override fun query(
        uri: Uri,
        projection: Array<String>?,
        selection: String?,
        selectionArgs: Array<String>?,
        sortOrder: String?
    ): Cursor? {
        val db = dbHelper.readableDatabase
        val qb = SQLiteQueryBuilder().apply {
            tables = NoteContract.Notes.TABLE_NAME
            if (uriMatcher.match(uri) == URI_NOTE_ID) {
                appendWhere("${BaseColumns._ID} = ${uri.lastPathSegment}")
            }
        }
        return qb.query(db, projection, selection, selectionArgs, null, null, sortOrder)
            ?.also { it.setNotificationUri(context!!.contentResolver, uri) }
    }

    override fun insert(uri: Uri, values: ContentValues?): Uri? {
        if (uriMatcher.match(uri) != URI_NOTES)
            throw IllegalArgumentException("Invalid URI for insert: $uri")

        val db = dbHelper.writableDatabase
        val id = db.insertOrThrow(NoteContract.Notes.TABLE_NAME, null, values)
        context!!.contentResolver.notifyChange(uri, null)
        return ContentUris.withAppendedId(NoteContract.Notes.CONTENT_URI, id)
    }

    override fun update(
        uri: Uri, values: ContentValues?,
        selection: String?, selectionArgs: Array<String>?
    ): Int {
        val db = dbHelper.writableDatabase
        val count = when (uriMatcher.match(uri)) {
            URI_NOTES   -> db.update(NoteContract.Notes.TABLE_NAME, values, selection, selectionArgs)
            URI_NOTE_ID -> db.update(
                NoteContract.Notes.TABLE_NAME, values,
                "${BaseColumns._ID} = ?", arrayOf(uri.lastPathSegment)
            )
            else -> throw IllegalArgumentException("Unknown URI: $uri")
        }
        if (count > 0) context!!.contentResolver.notifyChange(uri, null)
        return count
    }

    override fun delete(uri: Uri, selection: String?, selectionArgs: Array<String>?): Int {
        val db = dbHelper.writableDatabase
        val count = when (uriMatcher.match(uri)) {
            URI_NOTES   -> db.delete(NoteContract.Notes.TABLE_NAME, selection, selectionArgs)
            URI_NOTE_ID -> db.delete(
                NoteContract.Notes.TABLE_NAME,
                "${BaseColumns._ID} = ?", arrayOf(uri.lastPathSegment)
            )
            else -> throw IllegalArgumentException("Unknown URI: $uri")
        }
        if (count > 0) context!!.contentResolver.notifyChange(uri, null)
        return count
    }

    companion object {
        private const val URI_NOTES   = 1
        private const val URI_NOTE_ID = 2
    }
}
```

`AndroidManifest.xml`에 다음을 추가합니다:

```xml
<provider
    android:name=".NoteProvider"
    android:authorities="com.example.noteapp.provider"
    android:exported="true"
    android:readPermission="com.example.noteapp.permission.READ"
    android:writePermission="com.example.noteapp.permission.WRITE" />

<permission
    android:name="com.example.noteapp.permission.READ"
    android:protectionLevel="normal" />
<permission
    android:name="com.example.noteapp.permission.WRITE"
    android:protectionLevel="signature" />
```

`readPermission`은 일반 앱도 선언하면 읽을 수 있도록 `normal`로, `writePermission`은 같은 서명의 앱만 쓰도록 `signature`로 설정해 세밀한 접근 제어를 구현했습니다.

### 예제 2: FileProvider로 파일 공유 + ContentProviderOperation 배치 동기화 (Kotlin)

카메라 앱으로 사진 촬영 결과를 공유하거나, 서버 동기화 시 배치 연산을 활용하는 두 가지 패턴을 보여줍니다.

```kotlin
// FileProvider를 사용한 파일 공유 (예: 카메라 인텐트)
fun takePictureWithFileProvider(activity: Activity, requestCode: Int) {
    val photoFile = File(
        activity.getExternalFilesDir(Environment.DIRECTORY_PICTURES),
        "photo_${System.currentTimeMillis()}.jpg"
    )
    val photoUri: Uri = FileProvider.getUriForFile(
        activity,
        "${activity.packageName}.fileprovider",   // authority
        photoFile
    )
    val intent = Intent(MediaStore.ACTION_IMAGE_CAPTURE).apply {
        putExtra(MediaStore.EXTRA_OUTPUT, photoUri)
        addFlags(Intent.FLAG_GRANT_WRITE_URI_PERMISSION)
    }
    activity.startActivityForResult(intent, requestCode)
}

// BatchOperation: 서버 데이터를 ContentProvider에 원자적으로 동기화
suspend fun syncNotesFromServer(
    resolver: ContentResolver,
    remoteNotes: List<NoteDto>
) = withContext(Dispatchers.IO) {
    val ops = ArrayList<ContentProviderOperation>()

    // 기존 데이터 전체 삭제 후 재삽입 (전략에 따라 upsert로 변경 가능)
    ops += ContentProviderOperation
        .newDelete(NoteContract.Notes.CONTENT_URI)
        .build()

    remoteNotes.forEach { note ->
        ops += ContentProviderOperation
            .newInsert(NoteContract.Notes.CONTENT_URI)
            .withValue(NoteContract.Notes.COLUMN_TITLE, note.title)
            .withValue(NoteContract.Notes.COLUMN_BODY, note.body)
            .withValue(NoteContract.Notes.COLUMN_CREATED_AT, note.createdAt)
            .build()
    }

    // 전체를 하나의 트랜잭션으로 적용 — 실패 시 전체 롤백
    try {
        resolver.applyBatch(NoteContract.AUTHORITY, ops)
    } catch (e: OperationApplicationException) {
        Log.e("Sync", "Batch sync failed, rolled back", e)
        throw e
    }
}

// ContentResolver로 데이터 쿼리 후 Flow로 노출 (외부 앱 또는 동일 앱)
fun observeNotes(resolver: ContentResolver): Flow<List<Note>> = callbackFlow {
    val observer = object : ContentObserver(Handler(Looper.getMainLooper())) {
        override fun onChange(selfChange: Boolean) { trySend(fetchNotes(resolver)) }
    }
    resolver.registerContentObserver(NoteContract.Notes.CONTENT_URI, true, observer)
    trySend(fetchNotes(resolver))   // 초기값 즉시 emit
    awaitClose { resolver.unregisterContentObserver(observer) }
}

private fun fetchNotes(resolver: ContentResolver): List<Note> {
    val notes = mutableListOf<Note>()
    resolver.query(
        NoteContract.Notes.CONTENT_URI,
        arrayOf(BaseColumns._ID, NoteContract.Notes.COLUMN_TITLE, NoteContract.Notes.COLUMN_BODY),
        null, null, "${NoteContract.Notes.COLUMN_CREATED_AT} DESC"
    )?.use { cursor ->
        val idIdx    = cursor.getColumnIndexOrThrow(BaseColumns._ID)
        val titleIdx = cursor.getColumnIndexOrThrow(NoteContract.Notes.COLUMN_TITLE)
        val bodyIdx  = cursor.getColumnIndexOrThrow(NoteContract.Notes.COLUMN_BODY)
        while (cursor.moveToNext()) {
            notes += Note(cursor.getLong(idIdx), cursor.getString(titleIdx), cursor.getString(bodyIdx))
        }
    }
    return notes
}
```

`callbackFlow`와 `ContentObserver`를 결합하면 데이터 변경 시 자동으로 Flow에서 새 값을 emit하는 반응형 패턴을 만들 수 있습니다.

`FileProvider`는 `res/xml/file_paths.xml`과 Manifest 선언이 필요합니다:

```xml
<!-- res/xml/file_paths.xml -->
<paths>
    <external-files-path name="pictures" path="Pictures/" />
</paths>

<!-- AndroidManifest.xml -->
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

## 주의사항 및 실전 팁

**1. `query()`에서 반드시 `use {}` 또는 `close()` 호출**  
`ContentResolver.query()`가 반환하는 `Cursor`는 DB 커서를 래핑합니다. 닫지 않으면 파일 디스크립터 누수가 발생해 `SQLiteDatabaseCorruptException`이나 `EMFILE` 에러로 이어집니다. `use { }` 블록이나 `AutoCloseable`을 활용하세요.

**2. 메인 스레드에서 ContentResolver 쿼리 금지**  
`ContentResolver.query()`는 I/O 작업입니다. 메인 스레드에서 호출하면 ANR이 발생합니다. 반드시 `Dispatchers.IO` 코루틴 컨텍스트나 백그라운드 스레드에서 호출하세요.

**3. `android:exported` 보안 설정**  
Android 12(API 31)부터 `exported` 속성 명시가 필수입니다. 외부 앱이 접근해야 하면 `true`로 설정하되, 반드시 별도의 `readPermission` / `writePermission`을 함께 선언해 무분별한 접근을 차단하세요. FileProvider는 항상 `exported="false"`로 설정하고 URI 임시 권한만으로 공유합니다.

**4. UriMatcher 초기화 성능**  
`UriMatcher`는 Provider당 단 한 번만 초기화해야 합니다. 클래스 레벨의 `companion object` 안에서 초기화하거나, `lazy`를 사용해 최초 `query()` 호출 시 한 번만 생성되도록 하세요.

**5. MIME 타입 규칙 준수**  
`getType()` 구현 시 디렉토리 URI는 `vnd.android.cursor.dir/`, 단일 항목 URI는 `vnd.android.cursor.item/`으로 반환해야 합니다. 잘못된 MIME 타입은 일부 앱(특히 파일 관리 앱)과의 호환성 문제를 야기합니다.

**6. `notifyChange()` 호출로 데이터 변경 전파**  
insert, update, delete 후 `contentResolver.notifyChange(uri, null)`를 호출해야 등록된 `ContentObserver`(또는 `CursorLoader`)가 UI를 자동으로 갱신합니다. 누락 시 데이터는 변경됐지만 UI가 갱신되지 않는 버그가 발생합니다.

**7. 배치 연산의 back-reference 활용**  
`ContentProviderOperation.newInsert()`로 삽입한 레코드의 ID를 다음 연산에서 참조할 때 `withValueBackReference()`를 사용하면 결과 ID를 코드로 추적하지 않아도 됩니다. 연락처 동기화처럼 부모-자식 관계가 있는 데이터 삽입에 매우 유용합니다.

## 정리

ContentProvider는 Android 생태계에서 앱 간 데이터 공유의 유일한 공식 채널입니다. 보안 격리를 유지하면서 URI 권한 위임, FileProvider, 배치 연산을 올바르게 활용하면 연락처 앱, 파일 공유, 알람/캘린더 동기화 같은 시스템 수준의 통합을 안전하게 구현할 수 있습니다. 단일 앱 내부에서만 데이터를 사용한다면 Room + Repository 패턴으로 충분하지만, 다른 앱 또는 시스템과 데이터를 교환해야 하는 순간 ContentProvider는 피할 수 없는 선택입니다.

## 참고 자료
- [Content provider 생성 — Android Developers 공식 가이드](https://developer.android.com/guide/topics/providers/content-provider-creating)
- [FileProvider API 레퍼런스 — Jetpack Androidx](https://developer.android.com/reference/kotlin/androidx/core/content/FileProvider)
- [Content provider 기초 — Android Developers](https://developer.android.com/guide/topics/providers/content-provider-basics)
- [파일 공유 설정 (FileProvider 실전) — Android Developers](https://developer.android.com/training/secure-file-sharing/setup-sharing)
