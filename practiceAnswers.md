# Practice 7: Retrofit with GitHub REST API

---

## Part A. Theory Questions

### 1. What is Retrofit? Who develops it and which library sends the real HTTP request under it?
* **What it is:** Retrofit is a type-safe HTTP client for Android and Java/Kotlin that turns your REST API into a Kotlin/Java interface using annotations.
* **Developer:** It is developed and maintained by **Square**.
* **Underlying HTTP library:** **OkHttp** (also developed by Square) handles the actual network transport, connection pooling, and execution of the real HTTP requests under the hood.

---

### 2. Why is an API described as an interface in Retrofit? Who writes the implementation and which call gives it to you?
* **Why an interface:** Describing the API as an interface allows you to define *what* the HTTP requests look like (endpoints, methods, parameters, return types) declaratively without writing repetitive boilerplate networking code.
* **Who writes the implementation:** **Retrofit** generates the implementation at runtime using Java's dynamic proxy mechanism (`java.lang.reflect.Proxy`), which reads your method annotations and translates them into OkHttp `Call` objects.
* **Which call gives it to you:** Calling `retrofit.create(YourApiService::class.java)` returns the generated implementation instance.

---

### 3. Base URL is `https://api.github.com/api/v3/`. Which URL is called for `@GET("/users")` and for `@GET("users")`? Why must base URL end with `"/"`?
* **`@GET("/users")`:** Leading `/` tells Retrofit the path is relative to the **host root**, ignoring any path segments in the base URL.  
  **Called URL:** `https://api.github.com/users`
* **`@GET("users")`:** No leading `/` tells Retrofit to **append** the relative path directly onto the base URL.  
  **Called URL:** `https://api.github.com/api/v3/users`
* **Why base URL must end with `"/"`:** Retrofit follows standard RFC 3986 URI resolution rules (like `<a href>` in HTML). If the base URL did not end with `/`, the last segment (`v3`) would be treated as a file rather than a directory and would be replaced when resolving relative paths. Enforcing a trailing `/` prevents accidental path truncation.

---

### 4. What is the difference between `@Path` and `@Query`? Write one endpoint example for each.
* **`@Path`:** Replaces a dynamic placeholder segment directly inside the URL path (enclosed in `{}`). Used to identify a specific resource.
* **`@Query`:** Appends a key-value parameter to the end of the URL after `?` (and `&` for subsequent parameters). Used for filtering, sorting, or pagination.

```kotlin
// @Path example -> GET /users/octocat
@GET("users/{username}")
suspend fun getUser(@Path("username") username: String): User

// @Query example -> GET /search/repositories?q=kotlin
@GET("search/repositories")
suspend fun searchRepos(@Query("q") keyword: String): SearchResponse
```

---

### 5. When do you use `@Body` and when `@Field`? Which extra annotation does `@Field` require?
* **`@Body`:** Used when sending a serialized object (typically JSON or XML) as the raw HTTP request payload, commonly with `@POST`, `@PUT`, or `@PATCH`.
* **`@Field`:** Used when sending form-urlencoded key-value pairs (`application/x-www-form-urlencoded`), mimicking an HTML form submission.
* **Extra required annotation:** `@Field` requires the method to be annotated with **`@FormUrlEncoded`**.

---

### 6. What does a converter factory do? What happens if you add none? Name two converters.
* **What it does:** A converter factory serializes Kotlin/Java objects into HTTP request bodies (e.g., JSON) and deserializes HTTP response bodies back into Kotlin/Java data classes.
* **If none is added:** Retrofit only knows how to deserialize response bodies into OkHttp's raw `ResponseBody` type and can only accept `RequestBody` for `@Body`. Trying to use custom data classes will throw an `IllegalArgumentException` at runtime because Retrofit cannot find a matching converter.
* **Two converters:**
  1. `GsonConverterFactory` (`converter-gson`)
  2. `MoshiConverterFactory` (`converter-moshi`) *(also `KotlinxSerializationConverterFactory`)*

---

### 7. Compare `execute()`, `enqueue()` and `suspend fun`. Which one crashes on the Android main thread and why?
* **`execute()`:** Synchronous blocking call. It blocks the current thread until the HTTP response arrives and returns a `Response<T>`.
* **`enqueue()`:** Asynchronous callback-based call. It runs the network request on a background OkHttp dispatcher thread and delivers the result via `onResponse` or `onFailure` callbacks.
* **`suspend fun`:** Kotlin Coroutine integration. It suspends the coroutine without blocking the underlying thread while OkHttp executes the request in the background, then resumes with the parsed result `T` (or throws an exception).
* **Which crashes on Android main thread:** **`execute()`** crashes with a `NetworkOnMainThreadException` because performing synchronous, blocking network I/O on the UI thread freezes rendering and triggers Android's strict thread policy.

---

### 8. Why do we build Retrofit and OkHttpClient once (`ApiInstance`) and not in every screen? Give two reasons.
1. **Resource & Connection Pooling Efficiency:** Each `OkHttpClient` maintains its own internal thread pool and HTTP/2 connection pool. Creating a new instance per screen wastes memory, spawns redundant threads, and prevents TCP/TLS socket reuse (increasing latency).
2. **Reflection Caching:** `Retrofit` caches parsed method annotations and parameter adapters internally. Rebuilding `Retrofit` repeatedly forces it to re-run expensive reflection on every call.

---

### 9. What is an interceptor? Where do you add the `Authorization` header for all requests and what does `Authenticator` do differently?
* **What an interceptor is:** An OkHttp component (`Interceptor`) that sits in the request/response pipeline to observe, modify, retry, or short-circuit outgoing HTTP requests and incoming responses.
* **Where to add `Authorization` header:** Inside an **Application Interceptor** added to `OkHttpClient.Builder` via `.addInterceptor { chain -> ... }`, where you clone the original request with `.newBuilder().header("Authorization", "Bearer $token").build()`.
* **What `Authenticator` does differently:** While an `Interceptor` proactively attaches headers to *every* outgoing request, an `Authenticator` is reactive—it is **only triggered when the server responds with `401 Unauthorized`**, allowing you to refresh an expired access token and automatically retry the failed request.

---

### 10. A `suspend` call fails. When do you get `IOException` and when `HttpException`? Why should you never catch `Throwable`?
* **`IOException`:** Thrown when a **network-level or transport failure** occurs before a valid HTTP response is received (e.g., no internet connection, DNS resolution failure, socket timeout, or SSL handshake error).
* **`HttpException`:** Thrown when the server **successfully responds**, but with a non-2xx HTTP status code (e.g., `403 Forbidden`, `404 Not Found`, `422 Unprocessable Entity`, `500 Internal Server Error`).
* **Why never catch `Throwable`:**
  1. In Kotlin Coroutines, catching `Throwable` catches `CancellationException`, which breaks structured concurrency and prevents coroutines from cancelling properly.
  2. `Throwable` also catches fatal JVM errors (`Error` subclasses like `OutOfMemoryError` or `StackOverflowError`) that an application cannot and should not attempt to recover from.

---

## Part B. Practical Work (Kotlin Console Application)

### 1. Gradle Dependencies (`build.gradle.kts`)
```kotlin
plugins {
    kotlin("jvm") version "1.9.22"
    application
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    implementation("com.squareup.retrofit2:converter-gson:2.11.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.0")
}

application {
    mainClass.set("MainKt")
}
```

### 2. Data Models (`Models.kt`)
```kotlin
import com.google.gson.annotations.SerializedName

data class User(
    val login: String,
    val name: String?,
    @SerializedName("public_repos") val publicRepos: Int,
    val followers: Int
)

data class Repo(
    val name: String,
    @SerializedName("full_name") val fullName: String,
    val description: String?,
    @SerializedName("stargazers_count") val stargazersCount: Int
)

data class SearchResponse(
    @SerializedName("total_count") val totalCount: Int,
    val items: List<Repo>
)
```

### 3. Retrofit Service Interface (`GitHubService.kt`)
```kotlin
import retrofit2.http.GET
import retrofit2.http.Path
import retrofit2.http.Query

interface GitHubService {

    // 1. Get a user
    @GET("users/{username}")
    suspend fun getUser(
        @Path("username") username: String
    ): User

    // 2. List repositories of a user
    @GET("users/{username}/repos")
    suspend fun getUserRepos(
        @Path("username") username: String,
        @Query("per_page") perPage: Int,
        @Query("sort") sort: String = "updated"
    ): List<Repo>

    // 3. Search repositories
    @GET("search/repositories")
    suspend fun searchRepositories(
        @Query("q") keyword: String
    ): SearchResponse

    // 4a. Get a repository
    @GET("repos/{owner}/{repo}")
    suspend fun getRepository(
        @Path("owner") owner: String,
        @Path("repo") repo: String
    ): Repo

    // 4b. List repository languages
    @GET("repos/{owner}/{repo}/languages")
    suspend fun getRepositoryLanguages(
        @Path("owner") owner: String,
        @Path("repo") repo: String
    ): Map<String, Int>
}
```

### 4. Singleton `ApiInstance` with Logging & Retry Helper (`ApiInstance.kt`)
```kotlin
import kotlinx.coroutines.delay
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import retrofit2.HttpException
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import java.io.IOException

object ApiInstance {
    private const val BASE_URL = "https://api.github.com/"

    private val loggingInterceptor = HttpLoggingInterceptor().apply {
        level = HttpLoggingInterceptor.Level.BASIC
    }

    private val okHttpClient: OkHttpClient = OkHttpClient.Builder()
        .addInterceptor(loggingInterceptor)
        .build()

    val api: GitHubService by lazy {
        Retrofit.Builder()
            .baseUrl(BASE_URL)
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(GitHubService::class.java)
    }

    /**
     * Executes a network call with n = 3 retries on IOException (no internet)
     * and handles HTTP 403 rate limiting globally.
     */
    suspend fun <T> safeApiCall(
        maxRetries: Int = 3,
        onHttpError: ((HttpException) -> Unit)? = null,
        block: suspend () -> T
    ): T? {
        var attempt = 0
        while (true) {
            try {
                return block()
            } catch (e: IOException) {
                attempt++
                if (attempt > maxRetries) {
                    println("Check your connection")
                    return null
                }
                println("Connection failed. Retrying ($attempt/$maxRetries)...")
                delay(1000L)
            } catch (e: HttpException) {
                if (e.code() == 403) {
                    println("Limit reached, try later")
                } else {
                    onHttpError?.invoke(e) ?: println("HTTP Error: ${e.code()}")
                }
                return null
            }
        }
    }
}
```

### 5. Console Application Entry Point (`Main.kt`)
```kotlin
import kotlinx.coroutines.runBlocking

fun main() = runBlocking {
    val api = ApiInstance.api

    // Task 1: Get a user on startup
    print("Enter your GitHub login: ")
    val initialLogin = readlnOrNull()?.trim().orEmpty()
    fetchUser(initialLogin)

    while (true) {
        println("\n=== GitHub REST API Console ===")
        println("1. Get a user")
        println("2. List repositories of a user")
        println("3. Search repositories")
        println("4. Get a repository and its languages")
        println("0. Exit")
        print("Select an option: ")

        when (readlnOrNull()?.trim()) {
            "1" -> {
                print("Enter GitHub username: ")
                val username = readlnOrNull()?.trim().orEmpty()
                fetchUser(username)
            }

            "2" -> {
                print("Enter GitHub username: ")
                val username = readlnOrNull()?.trim().orEmpty()
                val repos = ApiInstance.safeApiCall(
                    onHttpError = { e ->
                        if (e.code() == 404) println("User not found")
                        else println("Error: ${e.code()}")
                    }
                ) {
                    api.getUserRepos(username = username, perPage = 5, sort = "updated")
                }

                if (repos != null) {
                    if (repos.isEmpty()) {
                        println("No repos yet")
                    } else {
                        repos.forEach { repo ->
                            println("- ${repo.name} (★ ${repo.stargazersCount})")
                        }
                    }
                }
            }

            "3" -> {
                print("Enter search keyword: ")
                val keyword = readlnOrNull()?.trim().orEmpty()
                val searchResult = ApiInstance.safeApiCall(
                    onHttpError = { e ->
                        if (e.code() == 422) println("Keyword required")
                        else println("Error: ${e.code()}")
                    }
                ) {
                    api.searchRepositories(keyword)
                }

                searchResult?.items?.take(3)?.forEach { repo ->
                    println("- ${repo.fullName} (★ ${repo.stargazersCount})")
                }
            }

            "4" -> {
                print("Enter owner (e.g., square): ")
                val owner = readlnOrNull()?.trim().orEmpty()
                print("Enter repo (e.g., retrofit): ")
                val repoName = readlnOrNull()?.trim().orEmpty()

                val repo = ApiInstance.safeApiCall(
                    onHttpError = { e ->
                        if (e.code() == 404) println("Repo not found")
                        else println("Error: ${e.code()}")
                    }
                ) {
                    api.getRepository(owner, repoName)
                }

                if (repo != null) {
                    val languages = ApiInstance.safeApiCall {
                        api.getRepositoryLanguages(owner, repoName)
                    } ?: emptyMap()

                    println("Description: ${repo.description ?: "No description"}")
                    println("Stars: ${repo.stargazersCount}")
                    println("Languages: $languages")
                }
            }

            "0" -> {
                println("Exiting...")
                break
            }

            else -> println("Invalid option.")
        }
    }
}

private suspend fun fetchUser(username: String) {
    val user = ApiInstance.safeApiCall(
        onHttpError = { e ->
            if (e.code() == 404) println("User not found")
            else println("Error: ${e.code()}")
        }
    ) {
        ApiInstance.api.getUser(username)
    }

    if (user != null) {
        println("Login: ${user.login}")
        println("Name: ${user.name ?: "N/A"}")
        println("Public Repos: ${user.publicRepos}")
        println("Followers: ${user.followers}")
    }
}
```

---

## How Retrofit Turns `GitHubService` into a Working HTTP Call

```text
GitHubService (Interface)
       │
       ▼  retrofit.create(GitHubService::class.java)
Java Dynamic Proxy (Runtime Implementation)
       │
       ▼  Reads annotations (@GET, @Path, @Query) & method signature
ServiceMethod / RequestFactory
       │
       ▼  Builds relative URL, encodes parameters & creates OkHttp Request
OkHttpClient (Runs HttpLoggingInterceptor & opens TCP/TLS socket)
       │
       ▼  Receives raw JSON ResponseBody from api.github.com
GsonConverterFactory
       │
       ▼  Deserializes JSON into Kotlin data classes (User, Repo, etc.)
Resumes suspend function with typed result (or throws HttpException / IOException)
```

1. **Dynamic Proxy Generation:** Because `GitHubService` is an interface with no implementation class written by us, `retrofit.create()` uses Java's `Proxy.newProxyInstance()` to generate a runtime object that intercepts every method invocation on the interface.
2. **Annotation Parsing:** When a method like `getUser("octocat")` is invoked, Retrofit inspects the method's annotations (`@GET("users/{username}")`, `@Path("username")`) and caches a `ServiceMethod` blueprint that knows how to substitute `{username}` with `"octocat"` and combine it with `BASE_URL`.
3. **Call Adaptation (Coroutines):** Because the method is marked `suspend`, Retrofit's built-in suspend call adapter wraps an `OkHttpCall` inside a cancellable coroutine (`suspendCancellableCoroutine`), enqueuing the work asynchronously without blocking the calling thread.
4. **Network Execution via OkHttp:** OkHttp passes the constructed `Request` through any registered interceptors (such as `HttpLoggingInterceptor`, which logs the HTTP method, URL, status code, and execution time) and transmits the bytes over the network.
5. **Deserialization:** Once the HTTP `200 OK` response arrives, `GsonConverterFactory` reads the JSON stream and maps the fields into the requested Kotlin return type (`User`), resuming the suspended coroutine with the final object—or throwing `HttpException` / `IOException` if an error occurred.
