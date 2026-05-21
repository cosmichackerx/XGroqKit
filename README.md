# XGroqKit
A lightweight, asynchronous Android client for streaming Groq API responses using Kotlin Coroutines (Flow) and OkHttp.

### ⚙️ Installation & Setup
**1. Add the Object**
Copy the GroqKit.kt file into your project's data or network layer (e.g., com.yourname.app.data.api).

```kotlin
package com.example.cosmicdns.data.api

import com.example.cosmicdns.BuildConfig
import com.google.gson.Gson
import java.io.IOException
import java.util.concurrent.TimeUnit
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.flow.flowOn
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.OkHttpClient
import okhttp3.Request
import okhttp3.RequestBody.Companion.toRequestBody

/**
 * Modular XGroq API client. Uses [BuildConfig] fields and fails gracefully when unset.
 */
object XGroqKit {

    private val gson = Gson()
    private val jsonMediaType = "application/json; charset=utf-8".toMediaType()

    private val client: OkHttpClient by lazy {
        OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(60, TimeUnit.SECONDS)
            .build()
    }

    val isAvailable: Boolean
        get() = try {
            BuildConfig.COSMIC_API_KEY.isNotBlank() && BuildConfig.COSMIC_API_URL.isNotBlank()
        } catch (_: Throwable) {
            false
        }

    fun streamChat(
        prompt: String,
        model: String = "llama-3.3-70b-versatile",
        userId: String? = null
    ): Flow<String> = flow {
        if (!isAvailable) {
            throw IOException("XGroqKit is not configured. Check your build.gradle fields.")
        }

        val requestBody = GroqRequest(
            model = model,
            messages = listOf(GroqMessage(role = "user", content = prompt)),
            stream = true,
            user = userId
        )

        val request = Request.Builder()
            .url(BuildConfig.COSMIC_API_URL)
            .addHeader("Authorization", "Bearer ${BuildConfig.COSMIC_API_KEY}")
            .addHeader("Accept", "text/event-stream")
            .post(gson.toJson(requestBody).toRequestBody(jsonMediaType))
            .build()

        client.newCall(request).execute().use { response ->
            if (!response.isSuccessful) {
                throw IOException(
                    "XGroqKit error ${response.code}: ${response.body?.string()?.take(200)}"
                )
            }

            val source = response.body?.source() ?: return@flow
            while (!source.exhausted()) {
                val line = source.readUtf8Line() ?: break
                if (line.startsWith("data: ") && line != "data: [DONE]") {
                    try {
                        val chunk = gson.fromJson(line.substring(6), GroqStreamChunk::class.java)
                        val content = chunk.choices.firstOrNull()?.delta?.content
                        if (content != null) emit(content)
                    } catch (_: Exception) {
                        // Skip malformed chunks
                    }
                }
            }
        }
    }.flowOn(Dispatchers.IO)

    private data class GroqRequest(
        val model: String,
        val messages: List<GroqMessage>,
        val stream: Boolean,
        val user: String?
    )

    private data class GroqMessage(val role: String, val content: String)

    private data class GroqStreamChunk(val choices: List<StreamChoice>)
    private data class StreamChoice(val delta: StreamDelta)
    private data class StreamDelta(val content: String?)
}
```

**2. Configure API Keys (Securely)**
Do not hardcode your API keys. Add them to your **local.properties** file:
```local.properties
GROQ_API_KEY=gsk_your_api_key_here
GROQ_API_URL=https://api.groq.com/openai/v1/chat/completions
```

Then, expose them in your app/build.gradle.kts:

```kotlin
android {
    defaultConfig {
        // Read from local.properties
        val properties = java.util.Properties()
        properties.load(project.rootProject.file("local.properties").inputStream())
        
        buildConfigField("String", "COSMIC_API_KEY", "\\"${properties.getProperty("GROQ_API_KEY")}\\"")
        buildConfigField("String", "COSMIC_API_URL", "\\"${properties.getProperty("GROQ_API_URL")}\\"")
    }
    buildFeatures {
        buildConfig = true
    }
}
```
### 💻 Usage Tutorial
**GroqKit** is best used within a ViewModel to update your UI state as the AI types its response.

**Basic Implementation**
```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.catch
import kotlinx.coroutines.launch

class ChatViewModel : ViewModel() {

    private val _aiResponse = MutableStateFlow("")
    val aiResponse = _aiResponse.asStateFlow()

    fun fetchAnswer(prompt: String) {
        // Clear previous response
        _aiResponse.value = ""

        viewModelScope.launch {
            // GroqKit returns a Flow<String> of tokens
            GroqKit.streamChat(prompt = prompt, model = "llama-3.3-70b-versatile")
                .catch { exception ->
                    _aiResponse.value = "Error: ${exception.message}"
                }
                .collect { token ->
                    // Append each incoming chunk to the state
                    _aiResponse.value += token
                }
        }
    }
}
```
**Collecting in Jetpack Compose**
```kotlin
@Composable
fun ChatScreen(viewModel: ChatViewModel) {
    val responseText by viewModel.aiResponse.collectAsState()

    Text(
        text = responseText,
        modifier = Modifier.padding(16.dp)
    )
}
```
