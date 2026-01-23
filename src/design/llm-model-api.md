# LLM Model API

For an enterprise **Agent-as-a-Service** platform in Java, the choice between REST and SDKs depends on how much control you want over the "plumbing" versus how fast you want to move.

However, since you mentioned supporting **multiple LLMs** and building a **generic engine**, there is actually a **third option** which is the industry standard for Java: **LangChain4j**.

Here is the breakdown of your options:

---

### 1. Direct REST API (Using Java `HttpClient` or `WebClient`)
You call the LLM endpoints (e.g., `https://api.openai.com/v1/chat/completions`) directly using JSON.

*   **Pros:**
    *   **No Dependency Bloat:** You don't need to import 10 different vendor libraries.
    *   **Full Control:** You have total control over timeouts, headers, and retries.
    *   **Gateway Ready:** If you use an internal **AI Gateway** (as we discussed), REST is the natural way to talk to it.
    *   **Always Up to Date:** You don't have to wait for an SDK update to use a new model feature.
*   **Cons:**
    *   **Boilerplate:** You have to write Java POJOs (DTOs) for every request and response type for every model.
    *   **Complex Streaming:** Handling Server-Sent Events (SSE) for streaming text responses in Java is tedious.

### 2. Vendor Java SDKs (e.g., OpenAI Java, Azure OpenAI SDK)
You use a library provided by the model vendor or the community.

*   **Pros:**
    *   **Type Safety:** You get nice Java objects like `ChatCompletionRequest`.
    *   **Faster Setup:** Auth and streaming are handled out of the box.
*   **Cons:**
    *   **Vendor Lock-in:** The code you write for an OpenAI SDK won't work for Anthropic or Gemini.
    *   **Library Conflicts:** Different SDKs might depend on different versions of `Jackson` or `OkHttp`, leading to "Jar Hell."

### 3. The "Generic Agent" Winner: AI Orchestration Frameworks
Since you are building a generic engine that must support multiple models, you should use **LangChain4j** (or **Spring AI**). These act as a "JDBC driver" for LLMs.

#### Why LangChain4j is the best fit for your Java Agent:
It defines a generic interface. You write your agent logic once, and you can swap the "model" in your database.

*   **Pros:**
    *   **Unified API:** You call a method like `chatLanguageModel.generate(message)` and it works for OpenAI, Claude, Gemini, or even a local Ollama instance.
    *   **Built-in Tools:** It already has the logic to handle the **Function Calling** loop (crucial for your Agent Skills).
    *   **Extensible:** It integrates perfectly with your "Generic Agent" philosophy.

---

### How to support Multiple Models (The Strategy)

To keep your Java code "Generic" and "defined from the database," you should implement a **Strategy Pattern** combined with a **Factory**.

#### 1. Define the Config in the Database
Your `workflow.json` or Database record should contain the provider details:
```json
"model": {
  "provider": "anthropic",
  "name": "claude-3-5-sonnet",
  "apiKeySecret": "ANTHROPIC_KEY"
}
```

#### 2. Create a Model Factory in Java
Using LangChain4j, your factory would look like this:

```java
public class ModelFactory {
    
    public ChatLanguageModel getModel(String provider, String modelName, String apiKey) {
        return switch (provider.toLowerCase()) {
            case "openai" -> OpenAiChatModel.builder()
                                .apiKey(apiKey)
                                .modelName(modelName)
                                .build();
            case "anthropic" -> AnthropicChatModel.builder()
                                .apiKey(apiKey)
                                .modelName(modelName)
                                .build();
            default -> throw new IllegalArgumentException("Unknown provider");
        };
    }
}
```

#### 3. Use the Model in your Generic Agent
Your agent code doesn't care which model it is talking to:

```java
public void runAgent(AgentConfig config) {
    // 1. Get model from factory
    ChatLanguageModel model = factory.getModel(config.provider, config.name, config.key);
    
    // 2. Get prompt from your skill hierarchy (Markdown from DB)
    String systemMessage = skillService.getMergedSkills(config.skillIds);
    
    // 3. Invoke (The interface is the same regardless of the provider!)
    String response = model.generate(SystemMessage.from(systemMessage), UserMessage.from(userInput)).content().text();
}
```

---

### Summary Recommendation

1.  **Do not use individual Vendor SDKs.** It will make your "Generic Agent" code messy with `if/else` blocks.
2.  **Use LangChain4j.** It is the most mature Java framework for this. It handles the REST calls, JSON parsing, and streaming for you, but provides a single Java interface for all models.
3.  **If you have an AI Gateway:** Use LangChain4j's "OpenAI" provider but point the `baseUrl` to your **Internal Gateway**. Most LLM providers (and gateways) now support the "OpenAI-compatible" API format, making this very easy.

This approach ensures that if a new "Super Model" is released tomorrow, you just add one entry to your `ModelFactory` and a record to your Database, and all your agents can use it immediately.
