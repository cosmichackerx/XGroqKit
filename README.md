# XGroqKit

A lightweight, asynchronous Android client for streaming Groq API responses using Kotlin Coroutines (`Flow`) and OkHttp.

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

## ⚙️ Installation & Setup

### 1. Add the Object

Copy the `XGroqKit.kt` file into your project's data or network layer.

Example path:

```text
com.yourname.app.data.api
```

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

# 🚀 Use Cases

Because `XGroqKit` streams responses in real time, it can power highly interactive AI experiences across Android apps.

---

## 🧠 1. AI Chat Assistant

Stream conversational replies like ChatGPT directly into Android apps.

**Example**

```kotlin
viewModelScope.launch {

    XGroqKit.streamChat(
        prompt = "Explain black holes simply."
    )
        .collect { token ->

            chatState.value += token
        }
}
```

**Great For**

- AI companions
- educational apps
- productivity assistants
- customer support bots

---

## 🌐 2. DNS Security Analyzer

Analyze suspicious domains or IPs in real time.

**Example**

```kotlin
val dnsPrompt = """
Analyze this DNS record:

IP: 185.143.223.12

Check for:
- phishing risk
- suspicious behavior
- hosting reputation
- security recommendations
"""

XGroqKit.streamChat(dnsPrompt)
```

**Streamed Output Example**

```text
⚠️ Suspicious Hosting Provider Detected
⚠️ Potential phishing infrastructure
✅ Recommend blocking outbound traffic
```

**Great For**

- cybersecurity apps
- SOC dashboards
- penetration testing tools
- DNS intelligence platforms

---

## 💻 3. AI Code Reviewer

Review source code while the user types.

**Example**

```kotlin
val codePrompt = """
Review this Kotlin code for:
- bugs
- memory leaks
- optimization opportunities

Code:
$sourceCode
"""

XGroqKit.streamChat(codePrompt)
```

**Great For**

- IDE assistants
- code review tools
- learning platforms
- debugging utilities

---

## 📄 4. README Generator

Generate GitHub documentation automatically.

**Example**

```kotlin
val prompt = """
Generate a professional README for:

Project Name: CosmicDNS
Description: AI-powered DNS security analyzer
"""

XGroqKit.streamChat(prompt)
```

**Great For**

- developer tools
- automation pipelines
- GitHub integrations

---

## 🎤 5. Voice Assistant Pipeline

Combine Whisper + XGroqKit.

**Architecture**

```text
Voice Input
    ↓
Whisper STT
    ↓
Raw Transcript
    ↓
XGroqKit
    ↓
Clean AI Response
```

**Example**

```kotlin
val transcript = speechRecognizerResult

XGroqKit.streamChat(
    prompt = "Summarize this meeting:\n$transcript"
)
```

**Great For**

- meeting assistants
- AI note-taking
- accessibility apps
- productivity tools

---

## 📊 6. Log File Analyzer

Stream AI analysis for server logs.

**Example**

```kotlin
val prompt = """
Analyze these logs for:
- crashes
- security threats
- anomalies

Logs:
$serverLogs
"""

XGroqKit.streamChat(prompt)
```

**Great For**

- DevOps dashboards
- monitoring systems
- SIEM platforms

---

## 🛡️ 7. Malware / URL Scanner

Analyze URLs and suspicious payloads.

**Example**

```kotlin
val prompt = """
Analyze this URL for malicious indicators:

https://unknown-domain.xyz/login
"""

XGroqKit.streamChat(prompt)
```

**AI Output Example**

```text
⚠️ Newly registered domain
⚠️ Suspicious login imitation
⚠️ Potential credential harvesting
```

---

## 📚 8. AI Study Tutor

Educational streaming assistant.

**Example**

```kotlin
XGroqKit.streamChat(
    prompt = "Teach me binary trees step-by-step."
)
```

**Great For**

- edtech apps
- CS learning platforms
- AI tutors

---

## ✍️ 9. Smart Writing Assistant

Generate polished writing in real time.

**Example**

```kotlin
val prompt = """
Write a professional email requesting internship opportunities.
"""

XGroqKit.streamChat(prompt)
```

**Great For**

- email assistants
- resume builders
- writing enhancement tools

---

## 🧬 10. AI Research Summarizer

Summarize large research content progressively.

**Example**

```kotlin
val prompt = """
Summarize this research paper:

$paperText
"""

XGroqKit.streamChat(prompt)
```

**Great For**

- scientific apps
- medical tools
- AI research platforms

---

## 🤖 11. Multi-turn AI Conversations

> **Roadmap:** This should be added in a future release.

Future-ready structure:

```kotlin
val messages = listOf(
    GroqMessage("system", "You are a cybersecurity expert."),
    GroqMessage("user", "Analyze this IP."),
    GroqMessage("assistant", "The IP appears suspicious."),
    GroqMessage("user", "Why?")
)
```

This transforms XGroqKit from a simple wrapper into real AI SDK territory.

---

## ⚡ 12. Streaming Markdown Renderer

Live-render markdown as AI streams.

**Example**

```kotlin
XGroqKit.streamChat(
    prompt = "Generate markdown documentation."
)
```

Then render with:

- Markdown Compose
- RichText
- WebView
- HTML parser

---

## 🔥 13. Terminal / Linux Assistant

Perfect for hacker-style tooling.

**Example**

```kotlin
val prompt = """
Explain this Linux error:

Permission denied while executing chmod
"""
```

**Great For**

- terminal apps
- developer consoles
- cyber tooling

---

## 🛰️ 14. Astronomy AI Assistant

Fits cosmic branding naturally.

**Example**

```kotlin
XGroqKit.streamChat(
    prompt = "Explain neutron stars scientifically."
)
```

**Could become**

- space education app
- telescope assistant
- astrophysics tutor

---

## 🎮 15. NPC Dialogue Generator

Real-time AI dialogue generation.

**Example**

```kotlin
val prompt = """
Generate a mysterious cyberpunk NPC dialogue.
"""
```

**Great For**

- indie games
- RPG systems
- AI storytelling

---

# ✨ Features

- Kotlin Coroutines + Flow
- Streaming token support
- Minimal dependencies
- Secure API configuration
- MVVM-friendly architecture
- Jetpack Compose compatible
- Graceful failure handling
- Lightweight & modular

---

# 👨‍💻 Author

### cosmichackerx 🚀

Passionate about:

- AI Engineering
- Android Development
- Streaming Architectures
- Cybersecurity
- Astronomy
- Real-time Systems
