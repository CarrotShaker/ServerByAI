# 📖 Supabase 안드로이드 실전 구현 스터디 문서 (MVVM + Compose 적용)
이 문서는 시놀로지에 세팅된 Supabase를 백엔드로 활용하여 안드로이드 앱에서 CRUD(생성/조회/수정/삭제)를 구현할 때 필수적인, **API Key 보안 통제**와 **ViewModel 및 Compose UDF(단방향 데이터 흐름) 아키텍처**를 다룹니다.
---
## 1. PostgREST 엔진의 동작 원리 (Retrofit의 대체)
Supabase 플랫폼에서 백엔드 개발 없이 즉각적인 API 통신이 가능한 이유는 내장된 **PostgREST** 서비스 덕분입니다. 
PostgREST는 PostgreSQL 데이터베이스의 스키마를 읽어들여 웹 API(RESTful API)를 자동으로 호스팅하는 오픈소스 컴포넌트입니다.
* **핵심 이점:** 기존 안드로이드 개발 방식에서 요구되었던 **Retrofit Interface 작성, OkHttp 클라이언트 구축, 직렬화 파서 연동 과정이 전면 생략**됩니다. 
* 안드로이드 개발자가 사용하는 `supabase-kt` 라이브러리는 내부에 **Ktor 엔진**을 탑재하고 있어, `insert(객체)` 메서드 호출 시 내부적으로 직렬화와 HTTP POST 통신을 자동화하여 서버로 전송합니다.
---
## 2. API Key 보안 관리 (`local.properties`) 🔐
Supabase 서버 인스턴스 주소(URL)와 접근 권한 키(Anon Key)는 버전 관리 시스템(GitHub 등)에 노출되지 않아야 합니다. 이를 위해 **`local.properties`** 와 **`BuildConfig`** 를 활용하여 중요 자격 증명을 분리해야 합니다.
### 1단계: `local.properties` 에 환경 변수 저장
안드로이드 프로젝트 최상단에 위치한 `local.properties` 파일 최하단에 서버 주소와 키를 추가합니다. 이 파일은 `.gitignore`에 기본 등록되어 있어 저장소에 포함되지 않습니다.
```properties
# local.properties 파일
SUPABASE_URL="https://내시놀로지도메인.synology.me"
SUPABASE_ANON_KEY="내_아논_키_무작위_문자열"
```
### 2단계: `build.gradle.kts (Module: app)` 에서 BuildConfig 연동
앱 수준의 `build.gradle.kts` 파일 내에서 Properties 객체를 초기화하여 `BuildConfig` 상수로 매핑합니다. 
```kotlin
import java.util.Properties
val properties = Properties()
properties.load(project.rootProject.file("local.properties").inputStream())
android {
    // ... 생략 ...
    buildFeatures {
        buildConfig = true // BuildConfig 클래스 자동 생성 옵션 활성화
    }
    defaultConfig {
        // 읽어들인 프로퍼티 값을 안드로이드 BuildConfig 상수로 맵핑
        buildConfigField("String", "SUPABASE_URL", properties.getProperty("SUPABASE_URL"))
        buildConfigField("String", "SUPABASE_ANON_KEY", properties.getProperty("SUPABASE_ANON_KEY"))
    }
}
```
### 3단계: Supabase 인스턴스 초기화
안전하게 격리된 `BuildConfig` 안의 변수를 참조하여 앱 전역에서 1회만 초기화하여 재사용 가능한 `Client` 객체를 구성합니다. 실무에서는 DI 프레임워크인 Dagger Hilt 등을 사용하여 싱글톤으로 제공받습니다.
```kotlin
// 추후 Hilt 모듈 등에서 싱글톤으로 인스턴스 제공
val supabaseClient = createSupabaseClient(
    supabaseUrl = BuildConfig.SUPABASE_URL,
    supabaseKey = BuildConfig.SUPABASE_ANON_KEY
) {
    install(PostgREST) // DB 서비스 모듈 로드
}
```
---
## 3. 안드로이드 클라이언트 구현: ViewModel 및 UI 상태 통제
비동기로 처리되는 CRUD 네트워크 호출을 `ViewModel` 계층에서 관리하고, `Jetpack Compose` UI 계층에서는 옵저빙(Observing)만 수행하는 **단방향 데이터 흐름 (UDF)** 패턴으로 구축해야 상태 이상 발생 확률이 낮아집니다.
### 1) 데이터 모델(Model) 구성
`todos` 테이블 아키텍처와 매핑되는 DTO 및 데이터 보관 모델을 작성합니다. 
```kotlin
import kotlinx.serialization.Serializable
import kotlinx.serialization.SerialName
@Serializable
data class Todo(
    val id: String? = null, 
    val title: String,
    @SerialName("is_completed") 
    val isCompleted: Boolean = false
)
```
### 2) ViewModel 상태 관리 로직
`supabaseClient` 를 주입받은 뷰모델 내에서 코루틴(`viewModelScope`)을 선언하여 C, R, U, D 작업을 명시적으로 수행합니다. 네트워크 I/O 등의 무거운 작업은 Compose 함수 컴포넌트 밖에서 처리해야 합니다.
```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
// ...
class TodoViewModel(private val supabase: SupabaseClient = supabaseClient) : ViewModel() {
    // 1. UI (Compose) 단이 구독할 읽기 전용 상태 (StateFlow)
    private val _todoList = MutableStateFlow<List<Todo>>(emptyList())
    val todoList: StateFlow<List<Todo>> = _todoList
    init { fetchTodos() }
    // -------------- CRUD 비동기 로직 --------------
    fun fetchTodos() {
        viewModelScope.launch {
            try {
                // PostgREST를 통한 API 호출: SELECT * FROM todos;
                val list = supabase.postgrest["todos"].select().decodeList<Todo>()
                _todoList.value = list // 데이터 수신 시 UI Re-composition 자동 유발
            } catch (e: Exception) {
                // 네트워크 타임아웃, RLS 예외 처리
            }
        }
    }
    fun addTodo(title: String) {
        viewModelScope.launch {
            val newTodo = Todo(title = title)
            // PostgREST: INSERT INTO todos; 전송
            supabase.postgrest["todos"].insert(newTodo)
            fetchTodos() // 통신 완료 시 데이터 동기화를 위해 리스트 갱신
        }
    }
    fun markCompleted(todoId: String) {
        viewModelScope.launch {
            supabase.postgrest["todos"].update({ Todo::isCompleted setTo true }) {
                filter { eq("id", todoId) }
            }
            fetchTodos()
        }
    }
    fun deleteTodo(todoId: String) {
        viewModelScope.launch {
            supabase.postgrest["todos"].delete {
                filter { eq("id", todoId) }
            }
            fetchTodos()
        }
    }
}
```
### 3) Jetpack Compose 연동을 통한 UDF 구현
Compose UI는 뷰모델의 네트워크 타임아웃 등에 관여하지 않고, 오직 `StateFlow` 값의 변화에만 종속되어 뷰를 리렌더링(`collectAsState()`)합니다.
```kotlin
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.*
import androidx.compose.runtime.*
@Composable
fun TodoScreen(viewModel: TodoViewModel = androidx.lifecycle.viewmodel.compose.viewModel()) {
    // ViewModel 상태 리스트 구독
    val todos by viewModel.todoList.collectAsState()
    
    // 할일 입력 State
    var textInput by remember { mutableStateOf("") }
    Column(modifier = Modifier.padding(16.dp)) {
        // UI: 입력 폼 영역 (Create 대응)
        Row {
            OutlinedTextField(
                value = textInput,
                onValueChange = { textInput = it },
                label = { Text("새로운 할 일 입력") }
            )
            Button(onClick = { 
                viewModel.addTodo(textInput) 
                textInput = "" // 입력 Field 초기화 
            }) { Text("추가") }
        }
        Spacer(Modifier.height(16.dp))
        // UI: 리스트 뷰 영역 (Read, Update, Delete 대응)
        LazyColumn {
            items(todos) { todo ->
                Row(
                    modifier = Modifier.fillMaxWidth().padding(8.dp),
                    horizontalArrangement = Arrangement.SpaceBetween
                ) {
                    Text(
                        text = todo.title,
                        color = if (todo.isCompleted) MaterialTheme.colorScheme.outline else MaterialTheme.colorScheme.onSurface
                    )
                    
                    Row {
                        if (!todo.isCompleted && todo.id != null) {
                            Button(onClick = { viewModel.markCompleted(todo.id) }) {
                                Text("완료")
                            }
                        }
                        if (todo.id != null) {
                            Button(onClick = { viewModel.deleteTodo(todo.id) }) {
                                Text("삭제")
                            }
                        }
                    }
                }
            }
        }
    }
}
```
---
## 요약
1. **API 키 관리**: `local.properties`와 `BuildConfig`를 활용하여 소스 코드 저장소에 중요 자격 증명이 누출되지 않도록 방지합니다.
2. **비동기 단방향 아키텍처 (UDF)**: 프론트엔드 Compose UI 범위 밖에서, 모든 API 네트워크 워크로드를 **ViewModel Coroutine** 단에서 독립적으로 수행하도록 역할을 철저히 분리했습니다.
3. **통신 보일러플레이트 극복**: 모델(Table) 작성만으로 즉각적인 API를 활성화하는 기반 기술을 구현함으로써, 비효율적인 인터페이스 구현 과정을 생략한 풀스택 파이프라인(BaaS)을 구축하였습니다.
