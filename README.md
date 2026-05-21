Here’s a polished section you can append directly into your `README.md` for **XGroqKit**:

````md
# XGroqKit
A lightweight, asynchronous Android client for streaming Groq API responses using Kotlin Coroutines (`Flow`) and OkHttp.

---

## ⚙️ Installation & Setup

### 1. Add the Object
Copy the `XGroqKit.kt` file into your project's data or network layer.

Example path:
```text
com.yourname.app.data.api
````

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
 * Modular XGroq API client.
 * Uses BuildConfig fields and fails gracefully when unset.
 */
object XGroqKit {

    private val gson = Gson()
    private val jsonMediaType =
        "application/json; charset=utf-8".toMediaType()

    private val client: OkHttpClient by lazy {
        OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(60, TimeUnit.SECONDS)
            .build()
    }

    val isAvailable: Boolean
        get() = try {
            BuildConfig.COSMIC_API_KEY.isNotBlank() &&
            BuildConfig.COSMIC_API_URL.isNotBlank()
        } catch (_: Throwable) {
            false
        }

    fun streamChat(
        prompt: String,
        model: String = "llama-3.3-70b-versatile",
        userId: String? = null
    ): Flow<String> = flow {

        if (!isAvailable) {
            throw IOException(
                "XGroqKit is not configured. Check your build.gradle fields."
            )
        }

        val requestBody = GroqRequest(
            model = model,
            messages = listOf(
                GroqMessage(
                    role = "user",
                    content = prompt
                )
            ),
            stream = true,
            user = userId
        )

        val request = Request.Builder()
            .url(BuildConfig.COSMIC_API_URL)
            .addHeader(
                "Authorization",
                "Bearer ${BuildConfig.COSMIC_API_KEY}"
            )
            .addHeader("Accept", "text/event-stream")
            .post(
                gson.toJson(requestBody)
                    .toRequestBody(jsonMediaType)
            )
            .build()

        client.newCall(request).execute().use { response ->

            if (!response.isSuccessful) {
                throw IOException(
                    "XGroqKit error ${response.code}: " +
                    "${response.body?.string()?.take(200)}"
                )
            }

            val source = response.body?.source()
                ?: return@flow

            while (!source.exhausted()) {

                val line = source.readUtf8Line() ?: break

                if (
                    line.startsWith("data: ") &&
                    line != "data: [DONE]"
                ) {
                    try {
                        val chunk = gson.fromJson(
                            line.substring(6),
                            GroqStreamChunk::class.java
                        )

                        val content =
                            chunk.choices
                                .firstOrNull()
                                ?.delta
                                ?.content

                        if (content != null) {
                            emit(content)
                        }

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

    private data class GroqMessage(
        val role: String,
        val content: String
    )

    private data class GroqStreamChunk(
        val choices: List<StreamChoice>
    )

    private data class StreamChoice(
        val delta: StreamDelta
    )

    private data class StreamDelta(
        val content: String?
    )
}
```

---

## 🔐 Configure API Keys Securely

Never hardcode API keys inside source code.

Add your credentials inside `local.properties`:

```properties
GROQ_API_KEY=gsk_your_api_key_here
GROQ_API_URL=https://api.groq.com/openai/v1/chat/completions
```

Then expose them through `app/build.gradle.kts`:

```kotlin
android {

    defaultConfig {

        val properties = java.util.Properties()

        properties.load(
            project.rootProject
                .file("local.properties")
                .inputStream()
        )

        buildConfigField(
            "String",
            "COSMIC_API_KEY",
            "\"${properties.getProperty("GROQ_API_KEY")}\""
        )

        buildConfigField(
            "String",
            "COSMIC_API_URL",
            "\"${properties.getProperty("GROQ_API_URL")}\""
        )
    }

    buildFeatures {
        buildConfig = true
    }
}
```

---

# 💻 Usage Tutorial

`XGroqKit` is designed for reactive Android architectures and works perfectly with `ViewModel`, `StateFlow`, and Jetpack Compose.

---

## Basic ViewModel Implementation

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

            XGroqKit.streamChat(
                prompt = prompt,
                model = "llama-3.3-70b-versatile"
            )
                .catch { exception ->
                    _aiResponse.value =
                        "Error: ${exception.message}"
                }
                .collect { token ->

                    // Append streamed tokens
                    _aiResponse.value += token
                }
        }
    }
}
```

---

## 🖼️ Collecting in Jetpack Compose

```kotlin
@Composable
fun ChatScreen(
    viewModel: ChatViewModel
) {

    val responseText by
        viewModel.aiResponse.collectAsState()

    Text(
        text = responseText,
        modifier = Modifier.padding(16.dp)
    )
}
```

---

# 🚀 Use Cases Inside a Project

Because `XGroqKit` streams responses in real-time, it can power highly interactive AI experiences.

---

## 1. Contextual AI Assistants

Create responsive chatbots and AI companions with instant token-by-token streaming.

Users see replies appear live instead of waiting for full responses.

---

## 2. Real-time Data Analyzers

Analyze logs, IP addresses, DNS data, or code snippets in real time.

### Example

A DNS security analyzer that streams:

* threat detection
* suspicious domains
* recommendations
* risk scoring

directly into the UI.

---

## 3. Voice-to-Text Refinement

Combine offline speech recognition engines like Whisper with `XGroqKit`.

Pipeline example:

```text
Speech → Whisper → Raw Transcript → XGroqKit → Clean AI Output
```

Use cases:

* grammar correction
* summaries
* actionable tasks
* meeting notes

---

## 4. Dynamic Content Generation

Generate:

* emails
* READMEs
* documentation
* AI code snippets
* project descriptions
* commit messages

in real time.

---

# ✨ Features

* Kotlin Coroutines + Flow
* Streaming token support
* Minimal dependencies
* Secure API configuration
* MVVM-friendly architecture
* Jetpack Compose compatible
* Graceful failure handling
* Lightweight & modular

---

# 📦 Recommended Dependencies

```kotlin
dependencies {

    implementation("com.squareup.okhttp3:okhttp:4.12.0")

    implementation("com.google.code.gson:gson:2.10.1")

    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.1")

    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")
}
```

---

# 👨‍💻 Author

### cosmichackerx 🚀

Passionate about:

* AI Engineering
* Android Development
* Streaming Architectures
* Cybersecurity
* Astronomy
* Real-time Systems

```
```
