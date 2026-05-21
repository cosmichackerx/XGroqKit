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
