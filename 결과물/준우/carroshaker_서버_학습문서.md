# 📖 Supabase 실전 학습 레퍼런스 문서 (Synology + Android Compose)
이 문서는 시놀로지에 세팅된 Supabase를 백엔드로 활용하여 안드로이드 앱에서 CRUD(생성/조회/수정/삭제) API를 다루기 위해 알아야 할 기능 명세와 코드 예제를 담은 실무 문서입니다.
---
## 1. PostgREST 엔진의 동작 원리
Supabase 플랫폼에서 백엔드 개발 없이 즉각적인 API 통신이 가능한 이유는 내장된 **PostgREST** 서비스 덕분입니다. 
PostgREST는 PostgreSQL 데이터베이스의 스키마를 읽어들여 웹 API(RESTful API)를 자동으로 호스팅하는 오픈소스 컴포넌트입니다. 별도의 ORM 작성이나 미들웨어 코딩 없이 데이터베이스 테이블 구조와 HTTP 메서드를 1:1로 매핑합니다.
*   **동작 흐름 명세:**
    1.  안드로이드(클라이언트) 단에서 HTTP 요청을 전송합니다. (예: `GET /rest/v1/todos`)
    2.  PostgREST 엔진이 이 HTTP Request를 수신하여, 내부적으로 `SELECT * FROM todos;` 와 같은 SQL 쿼리로 변환해 PostgreSQL로 전달합니다.
    3.  PostgreSQL에서 처리된 결과를 PostgREST가 다시 JSON 형태로 직렬화하여 클라이언트에 응답(Response)합니다.
결과적으로, 안드로이드 개발자가 사용하는 공식 라이브러리(`supabase-kt`)는 안드로이드의 데이터 객체를 이 PostgREST 규격에 맞춘 HTTP 요청으로 감싸서 전송하는 HTTP 클라이언트 래퍼 역할을 합니다.
🔗 **[공식 자료: PostgREST 공식 문서 (심화)](https://postgrest.org/en/stable/)**
---
## 2. 데이터베이스 테이블(Table) 생성
PostgREST가 동작하려면 기준이 될 실제 테이블이 데이터베이스에 존재해야 합니다. 테이블 생성(스키마 정의) 자체는 자동으로 되지 않으며, 개발자가 명시적으로 구성해야 합니다. 
### 1) 테이블 구성 방법: SQL Editor
시놀로지에 띄운 **Supabase Studio(대시보드)** 에 접속합니다. `todos` 테이블을 생성하는 과정은 다음과 같습니다.
**[SQL 쿼리 실행]**
Studio의 **SQL Editor** 메뉴에서 DDL(Data Definition Language) 쿼리를 실행합니다.
```sql
CREATE TABLE todos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(), -- 고유 식별자 (PK)
    title TEXT NOT NULL,                           -- 할 일 텍스트
    is_completed BOOLEAN DEFAULT FALSE,            -- 완료 상태
    created_at TIMESTAMPTZ DEFAULT NOW()           -- Timezone 포함 생성 시간
);
```
위 쿼리가 성공적으로 실행되면, PostgREST 서비스가 이를 감지하여 `https://내서버/rest/v1/todos` 엔드포인트를 즉각적으로 활성화합니다.
🔗 **[공식 가이드: Database 셋업 가이드](https://supabase.com/docs/guides/database)**
---
## 3. Row Level Security (RLS) 설정 🚨
Supabase 환경에서는 데이터베이스가 외부 API로 노출되므로, 권한 없는 접속 시도를 방어하기 위해 **Row Level Security (RLS)** 기반의 인가 통제가 필수적으로 요구됩니다. RLS를 활성화하면 기본적으로 해당 테이블에 대한 모든 접근이 Deny 처리됩니다.
**[RLS Policy 작성 예시]**
SQL 문법을 사용하여 각 액션(SELECT, INSERT, UPDATE, DELETE)에 대한 통과 조건(Policy)을 부여합니다.
```sql
-- 1. 조건 없는 데이터 조회 허용
CREATE POLICY "전체 공지사항 조회 허용" ON notices FOR SELECT USING (true);
-- 2. 로그인 세션(JWT 토큰 검증) 검증 후 Insert 허용
CREATE POLICY "회원 전용 글 등록 허용" ON todos FOR INSERT WITH CHECK (auth.role() = 'authenticated');
-- 3. 데이터의 소유자(user_id)와 요청자의 세션 ID(auth.uid()) 일치 여부 검증 후 Update 허용
CREATE POLICY "작성자 본인 수정 허용" ON todos FOR UPDATE USING (auth.uid() = user_id);
```
클라이언트 앱에서 상태 코드 `4xx` (Unauthorized 또는 Forbidden) 에러가 반환된다면, 해당 테이블에 대한 RLS 정책 누락 여부를 가장 먼저 확인해야 합니다.
🔗 **[공식 가이드: Row Level Security(RLS) 완벽 가이드](https://supabase.com/docs/guides/auth/row-level-security)**
---
## 4. 안드로이드 클라이언트 구현: CRUD 오퍼레이션 (supabase-kt)
활성화된 API 엔드포인트에 맞춰 `supabase-kt` 코틀린 라이브러리를 통해 명시적인 CRUD(Create, Read, Update, Delete) 메서드를 호출하는 구현부입니다.
### 1) 라이브러리 의존성 구성
**app/build.gradle.kts**
네트워크 통신용 엔진(Ktor)과 직렬화 도구(Kotlinx-serialization)가 연계되어야 합니다.
```kotlin
dependencies {
    implementation("io.github.jan-tennert.supabase:postgrest-kt:3.0.0") 
    implementation("io.ktor:ktor-client-android:2.3.12")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.3")
}
```
**모델 (Data Class) 검증**
데이터베이스의 컬럼 타입과 코틀린의 자료형이 엄격하게 일치해야 합니다. `@Serializable` 어노테이션을 통해 직렬화를 선언합니다.
```kotlin
import kotlinx.serialization.Serializable
import kotlinx.serialization.SerialName
@Serializable
data class Todo(
    // Insert 시 DB에서 Default 값으로 자동 삽입되므로 초기값은 Null 허용
    val id: String? = null, 
    val title: String,
    @SerialName("is_completed") // DB 컬럼(snake_case)과 속성 명시적 매핑
    val isCompleted: Boolean = false
)
```
### 2) Create (데이터 삽입)
`insert()` 함수를 통해 직렬화된 객체의 HTTP POST 요청을 전송합니다.
```kotlin
suspend fun createTodo(todoTitle: String) {
    val newTodo = Todo(title = todoTitle)
    supabase.postgrest["todos"].insert(newTodo)
}
```
### 3) Read (데이터 조회)
`select()` 함수를 통해 HTTP GET 요청을 전송하며, 람다 블록 내 `filter` 함수로 파라미터를 추가할 수 있습니다.
```kotlin
suspend fun fetchTodos(): List<Todo> {
    // 1. 전체 조회 (필터 없음)
    // val allTodos = supabase.postgrest["todos"].select().decodeList<Todo>()
    
    // 2. 조건 필터링 조회 (is_completed == false)
    val unfinishedTodos = supabase.postgrest["todos"]
        .select {
            filter { eq("is_completed", false) }
        }.decodeList<Todo>()
        
    return unfinishedTodos 
}
```
### 4) Update (데이터 수정)
특정 레코드를 갱신합니다. `update()` 의 첫 번째 인자로 변경할 값을 전달하고, 두 번째 인자로 대상을 특정하는 `filter`(WHERE 구문 역할)를 전달합니다.
```kotlin
suspend fun markAsCompleted(todoId: String) {
    supabase.postgrest["todos"].update(
        { Todo::isCompleted setTo true } // 변경 사항 반환
    ) {
        filter { eq("id", todoId) }      // 조건절 반환
    }
}
```
### 5) Delete (데이터 삭제)
`delete()` 함수 블록 안에서 갱신과 동일하게 대상을 특정하는 `filter` 구문을 전달합니다.
```kotlin
suspend fun deleteTodo(todoId: String) {
    supabase.postgrest["todos"].delete {
        filter { eq("id", todoId) }
    }
}
```
🔗 **[공식 가이드: supabase-kt 코틀린 초기화 및 API 레퍼런스](https://supabase.com/docs/reference/kotlin/introduction)**
---
## 5. 시놀로지(Synology) Self-Hosting 인프라 셋업 포인트
시놀로지의 Container Manager (Docker)에서 직접 런타임을 구성할 때 점검해야 할 3가지 인프라 요구사항입니다.
1. **포트 할당 및 충돌 방지 (`docker-compose.yml`)**: Supabase 환경은 다수의 컨테이너 그룹으로 실행됩니다. 핵심 포트는 API 게이트웨이(Kong)의 `8000`번, 데이터베이스 대시보드(Studio)의 `3000`번입니다. 시놀로지의 기존 애플리케이션과 포트 충돌이 발생하는 경우 `docker-compose.yml` 포트 매핑 옵션을(`- "8100:8000"`) 수정해야 합니다.
2. **보안 인프라 키 (`.env` 구성)**: SaaS형 플랫폼과 달리 인증 키워드를 직접 생성해야 합니다.
   * `JWT_SECRET`: 시스템의 가장 큰 취약점이 될 수 있으므로, 암호학적으로 안전한 32바이트 이상의 시크릿 문자열을 할당합니다.
   * `ANON_KEY`: 생성한 `JWT_SECRET` 기반으로 서명된 JWT 토큰입니다. 이 키 값이 안드로이드 클라이언트의 초기화 구성 시 주입됩니다.
3. **통신 암호화 레이어 (Reverse Proxy)**: 안드로이드 플랫폼 규정에 따라 `Cleartext (HTTP)` 트래픽은 기본 차단됩니다. 
   * 시놀로지의 외부 도메인을 발급(DDNS) 받은 뒤 인증서를 적용(Let's Encrypt)합니다.
   * 제어판의 '로그인 포털 > 고급 > 역방향 프록시'에서 외부 443(HTTPS) 포트 요청을 로컬망의 Supabase Kong 게이트웨이 포트(예: 8000번)로 리다이렉트 처리해야 정상 교신이 성립됩니다.
🔗 **[공식 가이드: Docker를 사용한 Supabase 셀프 호스팅 가이드](https://supabase.com/docs/guides/self-hosting/docker)**
🔗 **[공식 가이드: 셀프 호스트 런타임 JWT 및 키 환경변수 셋업](https://supabase.com/docs/guides/self-hosting/docker#generating-api-keys)**
