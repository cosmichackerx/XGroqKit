# XGroqKit
A lightweight, asynchronous Android client for streaming Groq API responses using Kotlin Coroutines (Flow) and OkHttp.

### ⚙️ Installation & Setup
**1. Add the Object**
Copy the GroqKit.kt file into your project's data or network layer (e.g., com.yourname.app.data.api).

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
