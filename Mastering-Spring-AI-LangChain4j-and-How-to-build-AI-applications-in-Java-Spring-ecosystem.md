# Mastering Spring AI, LangChain4j, and How to Build AI Applications in the Java / Spring Ecosystem

> A practical guide for experienced Java / Spring Boot backend developers.
> No basic Java. No basic Spring. No AI/ML math. Just: **What is it? → Why do we need it? → How does it work? → Java example → Real project.**

**Versions used in this guide (verified current as of September 2026):**

| Thing | Version |
|---|---|
| Java | 17+ (examples use 21) |
| Spring Boot | 3.5.x |
| Spring AI | **2.0.0 GA** (released June 2026) — BOM `org.springframework.ai:spring-ai-bom:2.0.0` |
| LangChain4j | **1.19.0** — BOM `dev.langchain4j:langchain4j-bom:1.19.0` |

> ⚠️ **Version note:** Spring AI 2.0 made two changes you'll hit immediately if you copy old tutorials:
> 1. Config keys dropped the `.options` segment: `spring.ai.openai.chat.options.model` → **`spring.ai.openai.chat.model`**.
> 2. The tool-calling loop is now an **advisor** in the chain (auto-registered), not embedded in the model call.
> This guide uses the 2.0 way throughout.

---

## Table of Contents

1. [AI Application Basics](#1-ai-application-basics)
2. [Calling an LLM From Java (the raw way)](#2-calling-an-llm-from-java-the-raw-way)
3. [Spring AI](#3-spring-ai)
4. [LangChain4j](#4-langchain4j)
5. [RAG](#5-rag)
6. [Tool Calling](#6-tool-calling)
7. [Memory](#7-memory)
8. [Agents](#8-agents)
9. [Spring AI vs LangChain4j](#9-spring-ai-vs-langchain4j)
10. [Practical Projects](#10-practical-projects)
11. [MCP — Simple Introduction](#11-mcp--simple-introduction)
12. [Final Project — AI Customer Support Assistant](#12-final-project--ai-customer-support-assistant)
13. [Learning Style Recap](#13-learning-style-recap)
14. [Final Summary — Mental Map](#14-final-summary--mental-map)

---

# 1. AI Application Basics

Before touching any framework, you need a correct mental model. As a backend developer, the single most useful reframe is this:

> **An LLM is just a remote, stateless, non-deterministic function you call over HTTP.**
> You send text in, you get text out. That's it. Everything else — memory, RAG, tools, agents — is *plumbing you build around that function.*

The core flow you will build over and over:

```text
Java application
      ↓          (you build an HTTP request with your text + config)
Spring AI  / LangChain4j
      ↓          (framework formats the request, adds API key, calls the provider)
LLM provider   (OpenAI, Anthropic, Google, AWS Bedrock, Ollama...)
      ↓          (routes to the actual model, runs inference)
LLM            (gpt-4o, claude, gemini, llama...)
      ↓          (generates text token by token)
Response       (JSON with the generated text)
      ↓
Java application  (you parse it and do something useful)
```

Keep this picture in your head. Frameworks just make each arrow shorter to write.

Now the vocabulary. I'll define each term **only to the depth you need to build apps.**

---

### LLM (Large Language Model)

**What is it?** A very large neural network trained on huge amounts of text. Given some text, it predicts the most likely next chunk of text. Chat, summaries, code, answers — all of it is "predict the next token, repeatedly."

**Why you care:** It's the engine. But it is **stateless** (forgets everything after each call), **non-deterministic** (same input can give different output), and **knows nothing about your data or the current time.** Those three limitations are *why* the rest of this guide exists.

---

### AI model

**What is it?** A specific, named, versioned LLM you can call. `gpt-4o-mini`, `claude-sonnet-5`, `gemini-2.0-flash`, `llama-3.3`.

**Why you care:** In your config you always name a model. Models differ in **quality, speed, cost, and context size.** A common pattern: cheap/fast model for classification and summaries, expensive/smart model for reasoning and tool use.

---

### AI provider

**What is it?** The company/service that hosts the model behind an API: OpenAI, Anthropic, Google, AWS Bedrock, Azure OpenAI, or a local runtime like **Ollama** (runs models on your own machine — great for dev with no API cost).

**Why you care:** The provider gives you the endpoint URL and requires an **API key**. Frameworks let you swap providers by changing a dependency + config, not your code.

---

### Prompt

**What is it?** The text (the input) you send to the model. "Prompt" can mean the whole request or just your instruction. It's the equivalent of the arguments you pass to a function.

**Why you care:** Prompt quality *is* output quality. "Prompt engineering" mostly means: be specific, give examples, tell the model its role, and tell it the output format you want.

---

### System / User / Assistant messages

Chat models don't take one blob of text — they take a **list of messages**, each with a role:

| Role | Meaning | Example |
|---|---|---|
| **system** | Instructions / persona / rules. Set once, applies to the whole conversation. | "You are a support agent for AcmeBank. Never give legal advice." |
| **user** | What the human said. | "Why was I charged $9.99?" |
| **assistant** | What the model said previously. | "That's your monthly subscription fee." |

**Why you care:** This message list is the actual API payload. **Conversation memory = you resending the previous messages.** RAG = you injecting retrieved text into a message. Tool results = a special message. Once you see everything as "manipulating the message list," AI apps stop being mysterious.

---

### Tokens

**What is it?** Models don't see characters or words — they see **tokens** (word pieces). Roughly **1 token ≈ 4 characters ≈ ¾ of a word** in English. "unbelievable" might be 3 tokens.

**Why you care — two very practical reasons:**
1. **You pay per token** (input + output).
2. There's a **maximum** number of tokens per request (see context window).

So token count is your *budget*. RAG, summarization, and memory-trimming all exist to keep token counts under control.

---

### Context window

**What is it?** The maximum number of tokens a model can consider in **one request** — everything: system message + full conversation history + retrieved documents + the model's answer, all counted together. Modern models range from ~128K to 1M+ tokens.

**Why you care:** You cannot "just send the whole database" or "the entire 500-page PDF." When history or documents get big, you overflow the window (error or truncation). This limitation is *the entire reason RAG exists* — you retrieve only the relevant chunks instead of sending everything.

```text
[ system ][ conversation history ][ retrieved docs ][ user question ][ room for answer ]
└──────────────── must all fit inside the context window ────────────────┘
```

---

### Temperature

**What is it?** A number (typically 0.0–1.0+) controlling randomness.
- **Low (0.0–0.3):** focused, deterministic, repeatable. Use for extraction, classification, structured output, factual Q&A.
- **High (0.7–1.0):** creative, varied. Use for brainstorming, marketing copy, story writing.

**Why you care:** For most backend/business apps you want **low temperature.** You almost never want a "creative" answer to "what's this customer's balance?"

---

### Embeddings

**What is it?** A function that turns text into a **list of numbers (a vector)**, e.g. 1536 floats. Texts with similar *meaning* produce vectors that are close together in space. "car" and "automobile" land near each other; "car" and "banana" land far apart.

**Why you care:** Embeddings let you do **semantic search** — find text by meaning, not keywords. This is the foundation of RAG. You don't need to understand the math; you need to understand: *text in → vector out → similar meaning = nearby vector.*

```text
"How do I reset my password?"   →  [0.12, -0.98, 0.33, ...]
"I forgot my login credentials" →  [0.11, -0.95, 0.30, ...]   ← close = similar meaning
"What's the capital of France?" →  [0.88,  0.10, -0.7, ...]   ← far   = unrelated
```

---

### Vector database

**What is it?** A database optimized to store embeddings and answer: *"give me the N stored items whose vectors are closest to this query vector."* Examples: **pgvector** (a PostgreSQL extension — our choice, because you already run Postgres), Chroma, Qdrant, Pinecone, Redis, Milvus.

**Why you care:** It's where your knowledge lives for RAG. As a Java dev the nice part: with pgvector it's *just a table in PostgreSQL* with a special column type and index.

---

### RAG (Retrieval-Augmented Generation)

**What is it?** A technique: before asking the LLM, **retrieve** relevant text from your own data and paste it into the prompt, so the model answers *from your data* instead of from its training memory.

**Why you care:** It solves the two biggest LLM problems at once:
- **Knowledge gap** — the model doesn't know your internal docs, product catalog, or last week's data.
- **Hallucination** — given real source text, the model makes things up far less.

Full section later. The one-liner: **RAG = "open-book exam" for the LLM.**

---

### Tool calling (a.k.a. function calling)

**What is it?** You describe some functions to the LLM (name, description, parameters). When answering, the model can say *"I want to call `getWeather("Tashkent")`."* **Your code** runs the real function and hands the result back; the model then writes the final answer.

**Why you care:** This is how an LLM *acts* instead of just *talks* — query a database, call a REST API, do math, send an email. The LLM decides *what* and *when*; your Java code stays in control of *how*.

---

### Memory

**What is it?** Making the model "remember" earlier turns of a conversation.

**Why you care — the key insight:** The LLM has **no built-in memory** (it's stateless). "Memory" is an illusion *you* create by **resending previous messages** with each new request. Frameworks automate storing and replaying that history.

---

### Agents

**What is it?** An LLM in a **loop**: it decides an action (usually a tool call), sees the result, decides the next action, and repeats until the task is done — instead of answering in a single shot.

**Why you care:** Agents handle multi-step tasks ("check inventory, then create the order, then email the customer") without you hard-coding the steps. **Without hype:** an agent is *a while-loop where an LLM picks the next step.* That's the honest definition. Full section later.

---

### Putting it together

```text
                         ┌─────────────────────────────────────────┐
                         │            Your Java Application          │
                         │                                           │
   User question ───────►│  1. Build message list (system + user)    │
                         │  2. (RAG) retrieve chunks from vector DB  │◄──── Vector DB (pgvector)
                         │  3. (Memory) prepend past messages        │◄──── Memory store
                         │  4. Send to LLM via framework             │
                         └───────────────┬───────────────────────────┘
                                         │  HTTP + API key
                                         ▼
                                   LLM Provider ──► LLM
                                         │
                          ┌──────────────┴───────────────┐
                          │ Answer   OR  "call tool X"    │
                          └──────────────┬───────────────┘
                                         │ if tool: your Java runs it, result goes back
                                         ▼
                                   Final answer ──► User
```

Everything in this guide is a variation on this diagram. Now let's build it.

---

# 2. Calling an LLM From Java (the raw way)

Before frameworks, see the bare metal. This demystifies everything that follows — **frameworks generate exactly this HTTP call for you.**

### The anatomy of an LLM API call

- **Endpoint (HTTP POST):** `https://api.openai.com/v1/chat/completions`
- **API key:** sent in an `Authorization: Bearer <key>` header. Proves who you are; ties usage to your billing. **Never hard-code it — use an env var.**
- **Request body (JSON):** the model name + the message list + options.
- **Response body (JSON):** the generated message + token usage.

Request JSON:

```json
{
  "model": "gpt-4o-mini",
  "messages": [
    { "role": "system", "content": "You are a concise assistant." },
    { "role": "user",   "content": "Explain what a REST API is in one sentence." }
  ],
  "temperature": 0.2
}
```

Response JSON (trimmed):

```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "A REST API is an HTTP-based interface for reading and modifying resources using standard methods like GET and POST."
      }
    }
  ],
  "usage": { "prompt_tokens": 23, "completion_tokens": 27, "total_tokens": 50 }
}
```

### Doing it with plain Java (`java.net.http.HttpClient`)

```java
import java.net.URI;
import java.net.http.*;

public class RawLlmCall {

    public static void main(String[] args) throws Exception {
        String apiKey = System.getenv("OPENAI_API_KEY"); // never hard-code

        // Note: normally you'd build this JSON with Jackson; inlined here for clarity.
        String requestBody = """
            {
              "model": "gpt-4o-mini",
              "messages": [
                { "role": "system", "content": "You are a concise assistant." },
                { "role": "user",   "content": "Explain what a REST API is in one sentence." }
              ],
              "temperature": 0.2
            }
            """;

        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("https://api.openai.com/v1/chat/completions"))
            .header("Content-Type", "application/json")
            .header("Authorization", "Bearer " + apiKey)
            .POST(HttpRequest.BodyPublishers.ofString(requestBody))
            .build();

        HttpResponse<String> response = HttpClient.newHttpClient()
            .send(request, HttpResponse.BodyHandlers.ofString());

        // response.body() is the raw JSON; you'd parse it with Jackson to pull out
        // choices[0].message.content
        System.out.println(response.body());
    }
}
```

That's the whole magic. An LLM app is HTTP + JSON + an API key.

### So why do we need frameworks?

Do the above in a real app and you immediately face:

- **JSON marshalling/unmarshalling** for every request and response shape.
- **Provider lock-in** — OpenAI, Anthropic, and Bedrock have *different* JSON shapes. Switching means rewriting.
- **Streaming** (Server-Sent Events) is fiddly to parse by hand.
- **Tool calling** requires a multi-step request/response dance you'd hand-code.
- **Structured output** — you want a Java object, not a `String` you regex.
- **Memory, RAG, retries, observability** — all boilerplate.

**Spring AI** and **LangChain4j** are that boilerplate, solved and tested:

```java
// The entire raw example above becomes:
String answer = chatClient.prompt()
    .system("You are a concise assistant.")
    .user("Explain what a REST API is in one sentence.")
    .call()
    .content();
```

Same HTTP call under the hood — you just stopped writing plumbing. Now let's learn both frameworks.

---

# 3. Spring AI

Spring AI is the official Spring project for building AI apps. If your world is Spring Boot, it *feels* native: auto-configuration, `application.properties`, dependency injection, starters. You inject a client and call it — exactly like `JdbcTemplate` or `RestClient`.

### Project setup (once, reused by every Spring AI example below)

`pom.xml` — import the BOM, add the model starter:

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.ai</groupId>
      <artifactId>spring-ai-bom</artifactId>
      <version>2.0.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <!-- Pick your provider. This one is OpenAI. -->
  <dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
  </dependency>
</dependencies>
```

`application.properties` (Spring AI **2.0** keys — no `.options` segment):

```properties
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.model=gpt-4o-mini
spring.ai.openai.chat.temperature=0.2
```

> 💡 **Zero-cost local dev:** swap the starter for `spring-ai-starter-model-ollama`, run `ollama run llama3.2` locally, and point config at it. No API key, no bill. Everything below works the same.

Now the concepts. For each: **What → Why → How → Example.**

---

## 3.1 ChatModel

**What is it?** The low-level interface representing one AI chat model. `chatModel.call(prompt)` in, `ChatResponse` out.

**Why do we need it?** It's the raw building block — the thin, portable wrapper over the provider's HTTP API. In Spring AI 2.0 you rarely use it directly; you use `ChatClient`. But it's what `ChatClient` sits on top of, and it's the abstraction that makes providers swappable.

**How does it work?** The starter auto-configures a `ChatModel` bean for your provider. It handles the JSON + API key + endpoint.

**Java example:**

```java
@Component
class LowLevelExample {
    private final ChatModel chatModel;
    LowLevelExample(ChatModel chatModel) { this.chatModel = chatModel; }

    String ask() {
        return chatModel.call("Say hello in Uzbek."); // convenience String overload
    }
}
```

> **Rule of thumb (Spring AI 2.0):** Use **`ChatClient`** for real work. Reach for `ChatModel` only when you need the lowest level.

---

## 3.2 ChatClient

**What is it?** The main, high-level, **fluent** API for talking to models. This is the one you'll use 95% of the time.

**Why do we need it?** It bundles everything — messages, options, tools, memory, RAG, structured output — behind one readable builder chain, and runs every request through an **advisor chain** (see 3.13). It's the `ChatClient` equivalent of `WebClient`.

**How does it work?** Inject the auto-configured `ChatClient.Builder`, call `.build()`, then chain: `.prompt()` → set messages/options → `.call()` (blocking) or `.stream()` (reactive) → extract the result (`.content()`, `.entity()`, `.chatResponse()`).

**Java example — a complete REST endpoint:**

```java
@RestController
class ChatController {

    private final ChatClient chatClient;

    // The builder is auto-configured. Configure defaults here if you want.
    ChatController(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("You are a helpful assistant. Be concise.")
            .build();
    }

    @GetMapping("/ask")
    String ask(@RequestParam String q) {
        return chatClient.prompt()   // start building a request
            .user(q)                 // the user message
            .call()                  // send it (blocking)
            .content();              // get the text answer
    }
}
```

**Project example:** This *is* Project 1 (Simple AI Chat). Everything else in Spring AI is adding links to this chain.

---

## 3.3 Prompt & Messages

**What is it?** A `Prompt` is the full input object: a list of `Message`s (`SystemMessage`, `UserMessage`, `AssistantMessage`) plus options. With `ChatClient` you rarely build `Prompt` by hand — `.system()` and `.user()` build the messages for you.

**Why do we need it?** Because chat models take *roles*, not a blob (recall section 1). The system message sets behavior; user messages carry the human input; assistant messages carry history.

**How does it work?** `.system(...)` adds/overrides the system message, `.user(...)` adds a user message. You can also use **prompt templates** with `{placeholders}`.

**Java example — templated prompt:**

```java
String answer = chatClient.prompt()
    .system("You are a translator. Only output the translation, nothing else.")
    .user(u -> u
        .text("Translate to {language}: {text}")
        .param("language", "French")
        .param("text", "Good morning, how are you?"))
    .call()
    .content();
```

> Under the hood the `{placeholders}` are filled by the `StTemplateRenderer` before the HTTP call.

---

## 3.4 Chat options

**What is it?** Per-request settings: model, temperature, max tokens, top-p, etc.

**Why do we need it?** You often want defaults in config but **override per call** — e.g., temperature 0 for extraction, 0.8 for a "make it catchy" endpoint — without new beans.

**How does it work?** Pass an options object to `.options()`. In Spring AI 2.0, options are **immutable and built with builders**.

**Java example:**

```java
String creative = chatClient.prompt()
    .user("Write a fun tagline for a coffee shop.")
    .options(OpenAiChatOptions.builder()
        .model("gpt-4o")       // override the default model just for this call
        .temperature(0.9)
        .maxTokens(60)
        .build())
    .call()
    .content();
```

---

## 3.5 Streaming

**What is it?** Getting the answer **token-by-token as it's generated**, instead of waiting for the whole thing.

**Why do we need it?** UX. A chatbot that types the answer live feels instant; a 6-second blank wait feels broken. Same total time, far better perceived latency.

**How does it work?** Use `.stream()` instead of `.call()`. You get a Project Reactor `Flux<String>`. Serve it as SSE from a Spring WebFlux/MVC endpoint.

**Java example — streaming SSE endpoint:**

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
Flux<String> stream(@RequestParam String q) {
    return chatClient.prompt()
        .user(q)
        .stream()      // <-- streaming instead of .call()
        .content();    // Flux<String>: each element is a chunk of the answer
}
```

---

## 3.6 Structured output

**What is it?** Getting a **typed Java object** back (a record/POJO or `List<T>`), not a raw `String` you have to parse.

**Why do we need it?** In backend code you want `Invoice`, not free text. Structured output makes the LLM behave like a **typed function**: text in, object out. Huge for extraction, classification, and building further logic on the result.

**How does it work?** Call `.entity(YourType.class)`. Spring AI injects format instructions (a JSON schema) into the prompt, then deserializes the JSON response into your type. Spring AI 2.0 adds a `StructuredOutputValidationAdvisor` that **self-corrects** if the JSON fails validation, and providers with native structured-output support can be used via `.useProviderStructuredOutput()`.

**Java example:**

```java
record MovieReview(String title, int year, String sentiment, List<String> themes) {}

MovieReview review = chatClient.prompt()
    .user("Analyze this review: 'Inception (2010) blew my mind — loved the layered dreams and heist tension.'")
    .call()
    .entity(MovieReview.class);   // returns a populated MovieReview record

// review.sentiment() -> "positive", review.themes() -> ["dreams", "heist"], etc.
```

**Project example:** Project 2 (Text Summarizer) returns a structured summary object with title, bullet points, and word count.

---

## 3.7 Tool calling

**What is it?** Letting the model call your Java methods (see section 6 for the deep dive).

**Why do we need it?** So the model can fetch live data and take actions instead of guessing.

**How does it work? (Spring AI 2.0 change!)** Annotate a method with `@Tool`, pass the containing object to `.tools(...)`. In 2.0, a `ToolCallingAdvisor` is **auto-registered** in the advisor chain and runs the entire call → execute → respond loop for you.

**Java example:**

```java
class WeatherTools {
    @Tool(description = "Get the current weather for a city")
    String getWeather(@ToolParam(description = "City name") String city) {
        // In real life: call a weather API. Stubbed here.
        return "Weather in " + city + ": 24°C, sunny";
    }
}

String answer = chatClient.prompt()
    .user("What should I wear in Tashkent today?")
    .tools(new WeatherTools())   // model may call getWeather("Tashkent"); loop is automatic
    .call()
    .content();
```

**Project example:** Project 4 (Tool-Calling Assistant).

---

## 3.8 RAG (in Spring AI)

**What is it?** Retrieval-Augmented Generation — retrieve relevant chunks from a vector store, inject into the prompt.

**Why do we need it?** So the model answers from *your* documents.

**How does it work?** The `QuestionAnswerAdvisor` does it automatically: on each call it embeds the user question, searches the `VectorStore`, and prepends the top chunks to the prompt.

**Java example (wiring):**

```java
String answer = chatClient.prompt()
    .advisors(QuestionAnswerAdvisor.builder(vectorStore).build())
    .user("What is our refund policy?")
    .call()
    .content();
```

Full build-out in [section 5](#5-rag) and Project 5.

---

## 3.9 Embeddings (in Spring AI)

**What is it?** The `EmbeddingModel` bean — text → vector.

**Why do we need it?** It's the engine behind RAG and any semantic search. Usually you don't call it directly; the `VectorStore` uses it for you. But you can.

**How does it work?** Add an embedding-capable starter (the OpenAI starter includes it), then inject `EmbeddingModel`.

**Java example:**

```java
@Component
class EmbeddingExample {
    private final EmbeddingModel embeddingModel;
    EmbeddingExample(EmbeddingModel m) { this.embeddingModel = m; }

    float[] vectorFor(String text) {
        return embeddingModel.embed(text);  // e.g. a 1536-float array
    }
}
```

Config for the embedding model (2.0 keys):

```properties
spring.ai.openai.embedding.model=text-embedding-3-small
```

---

## 3.10 Vector stores (in Spring AI)

**What is it?** The `VectorStore` interface — a uniform API over vector databases (pgvector, Chroma, Redis, Qdrant…). Two core operations: `add(documents)` and `similaritySearch(query)`.

**Why do we need it?** It's where RAG knowledge lives, and it keeps your code independent of the specific vector DB.

**How does it work?** Add a vector-store starter (e.g. `spring-ai-starter-vector-store-pgvector`), configure it, inject `VectorStore`. When you `add`, Spring AI embeds the text and stores vector + text + metadata. When you `similaritySearch`, it embeds the query and returns the nearest documents.

**Java example:**

```java
@Component
class KnowledgeBase {
    private final VectorStore vectorStore;
    KnowledgeBase(VectorStore vs) { this.vectorStore = vs; }

    void ingest() {
        vectorStore.add(List.of(
            new Document("Refunds are available within 30 days of purchase."),
            new Document("Support hours are 9am–6pm, Monday to Friday.")
        ));
    }

    List<Document> search(String query) {
        return vectorStore.similaritySearch(
            SearchRequest.builder().query(query).topK(3).build());
    }
}
```

pgvector config (2.0):

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/appdb
spring.datasource.username=postgres
spring.datasource.password=postgres

spring.ai.vectorstore.pgvector.initialize-schema=true
spring.ai.vectorstore.pgvector.index-type=HNSW
spring.ai.vectorstore.pgvector.distance-type=COSINE_DISTANCE
spring.ai.vectorstore.pgvector.dimensions=1536
```

---

## 3.11 Memory (in Spring AI)

**What is it?** `ChatMemory` — stores conversation history and replays it so the model "remembers."

**Why do we need it?** Because the model is stateless (section 1). Without it, every message is a fresh start.

**How does it work?** Add a `MessageChatMemoryAdvisor` to the chain and pass a **conversation id** per call so different users/chats stay separate.

**Java example:**

```java
ChatClient client = builder
    .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
    .build();

String reply = client.prompt()
    .user("What did I just ask you?")
    .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, "user-42"))
    .call()
    .content();
```

Full treatment in [section 7](#7-memory) and Project 3.

---

## 3.12 Advisors (enough to use them)

**What is it?** An **interceptor** in the `ChatClient` pipeline. Every request flows through an ordered chain of advisors before hitting the model, and every response flows back through them. Think **Servlet filters / Spring interceptors, but for AI calls.**

**Why do we need it?** They're how Spring AI composes cross-cutting behavior — memory, RAG, logging, tool calling — without you wiring it manually. **In Spring AI 2.0 this is central:** even tool calling is now an advisor (`ToolCallingAdvisor`), so a tool call can loop back through the chain.

**How does it work?** You add advisors with `.advisors(...)` (per call) or `.defaultAdvisors(...)` (on the builder). Each can modify the request going out and the response coming back. Order matters and is controlled by precedence.

The mental model:

```text
                 ┌──────────────────── ChatClient advisor chain ───────────────────┐
your request ──► │ MemoryAdvisor ─► RAG(QuestionAnswer)Advisor ─► ToolCallingAdvisor │ ──► LLM
your answer  ◄── │        (each advisor sees request out AND response back)          │ ◄── LLM
                 └──────────────────────────────────────────────────────────────────┘
```

**Built-in advisors you'll actually use:**

| Advisor | Purpose |
|---|---|
| `MessageChatMemoryAdvisor` | Adds conversation memory |
| `QuestionAnswerAdvisor` | Does RAG (retrieve + inject) |
| `ToolCallingAdvisor` | Runs the tool loop (auto-registered in 2.0) |
| `SimpleLoggerAdvisor` | Logs requests/responses (great for debugging) |

**Java example — stack several:**

```java
ChatResponse response = chatClient.prompt()
    .advisors(
        MessageChatMemoryAdvisor.builder(chatMemory).build(),   // memory
        QuestionAnswerAdvisor.builder(vectorStore).build(),     // RAG
        new SimpleLoggerAdvisor())                              // logging
    .user("What's our refund policy, and did I ask about this before?")
    .call()
    .chatResponse();
```

You now know enough Spring AI to build real apps. Let's meet the other framework.

---

# 4. LangChain4j

LangChain4j is an independent, **Java-first** framework for LLM apps (inspired by Python's LangChain, but idiomatic Java). It works with plain Java, Quarkus, *and* Spring Boot. Its signature feature — **AI Services** — is one of the most elegant abstractions in the whole space, so we'll spend extra time there.

### Setup

`pom.xml`:

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>dev.langchain4j</groupId>
      <artifactId>langchain4j-bom</artifactId>
      <version>1.19.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j</artifactId>
  </dependency>
  <dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai</artifactId>
  </dependency>

  <!-- OPTIONAL: Spring Boot starter that auto-wires everything from properties -->
  <dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai-spring-boot-starter</artifactId>
  </dependency>
</dependencies>
```

With the Spring Boot starter, config lives in `application.properties`:

```properties
langchain4j.open-ai.chat-model.api-key=${OPENAI_API_KEY}
langchain4j.open-ai.chat-model.model-name=gpt-4o-mini
langchain4j.open-ai.chat-model.temperature=0.2
```

---

## 4.1 ChatModel (LangChain4j)

**What is it?** The low-level model interface — LangChain4j's equivalent of Spring AI's `ChatModel`. `model.chat("...")` in, response out.

**Why do we need it?** Same reason: the portable, low-level building block. In plain Java you build it explicitly with a builder; with the Spring starter it's auto-configured.

**How does it work?** Build it with the provider's builder, or inject it.

**Java example:**

```java
ChatModel model = OpenAiChatModel.builder()
    .apiKey(System.getenv("OPENAI_API_KEY"))
    .modelName("gpt-4o-mini")
    .temperature(0.2)
    .build();

String answer = model.chat("What is dependency injection, in one sentence?");
```

You *can* use it directly, but the idiomatic LangChain4j way is AI Services…

---

## 4.2 AI Services ⭐ (the heart of LangChain4j)

**What is it?** You declare a **plain Java interface** describing what you want; LangChain4j generates a **dynamic proxy** that implements it by talking to the LLM. If you've used **Spring Data JPA repositories** or **OpenFeign/Retrofit clients**, you already know the pattern: *you write the interface, the framework writes the implementation.*

```java
interface Assistant {
    String chat(String message);
}
```

**Why do we need it?** It removes essentially all boilerplate. No manual message building, no response parsing, no glue. Your AI capability becomes **just an interface you inject and call** — which is exactly how the rest of your Java code already works.

**How does it work? (this is the important part)**

When you call `assistant.chat("Hello")`, the proxy:

```text
assistant.chat("Hello")
        ↓
1. Reads your interface + annotations (@SystemMessage, @UserMessage, params)
        ↓
2. Builds the message list:
     - @SystemMessage text        → SystemMessage
     - method argument / template → UserMessage
     - (if memory configured)     → prepends stored history
        ↓
3. (if tools configured)       → advertises your @Tool methods to the model
   (if retriever configured)   → runs RAG and injects retrieved chunks
        ↓
4. Calls the underlying ChatModel (the real HTTP call)
        ↓
5. Takes the model's reply and CONVERTS it to your method's return type:
     - String   → raw text
     - POJO     → parsed from JSON (structured output)
     - enum     → parsed as classification
     - Result<T>→ content + metadata (tokens, sources)
        ↓
returns to your code
```

The interface is a **typed façade over a message-list-manipulation-plus-HTTP-call.** Everything you learned in section 1 is happening — the proxy just does it for you based on the interface shape.

**Java example — building one manually (plain Java):**

```java
interface Assistant {
    @SystemMessage("You are a witty assistant. Keep answers under 20 words.")
    String chat(String userMessage);
}

ChatModel model = OpenAiChatModel.builder()
    .apiKey(System.getenv("OPENAI_API_KEY"))
    .modelName("gpt-4o-mini")
    .build();

Assistant assistant = AiServices.create(Assistant.class, model);
String reply = assistant.chat("Why is the sky blue?");
```

**Java example — the Spring Boot way (`@AiService`):**

```java
@AiService   // LangChain4j Spring starter creates & registers the bean automatically
interface Assistant {
    @SystemMessage("You are a witty assistant. Keep answers under 20 words.")
    String chat(String userMessage);
}

@RestController
class ChatController {
    private final Assistant assistant;   // injected like any Spring bean
    ChatController(Assistant assistant) { this.assistant = assistant; }

    @GetMapping("/ask")
    String ask(@RequestParam String q) { return assistant.chat(q); }
}
```

**Templated user messages & typed returns:**

```java
@AiService
interface SupportAgent {

    @SystemMessage("You are a support agent for AcmeBank. Be polite and concise.")
    @UserMessage("Customer said: {{message}}. Classify the urgency.")
    Urgency classifyUrgency(@V("message") String message);   // enum return = classification

    enum Urgency { LOW, MEDIUM, HIGH, CRITICAL }
}
```

`SupportAgent.classifyUrgency("My card was stolen!")` → `Urgency.CRITICAL`. The proxy handled the prompt, the call, and the enum parsing.

**Why this matters:** In LangChain4j, **AI Services are how you'll build almost everything** — chatbots, extractors, classifiers, RAG assistants, agents. Master this one concept and the rest is configuration.

---

## 4.3 Prompts & Messages (LangChain4j)

**What is it?** Same role concept as everywhere: `SystemMessage`, `UserMessage`, `AiMessage`. In AI Services you rarely construct them — annotations do it.

**Why & How:** `@SystemMessage` sets the persona; `@UserMessage` templates the user input with `{{placeholders}}` bound by `@V("name")` (or `{{it}}` for a single argument).

**Java example:**

```java
interface Translator {
    @UserMessage("Translate the following text to {{lang}}: {{text}}")
    String translate(@V("lang") String language, @V("text") String text);
}
```

If you need full control, you can still build messages manually and call `model.chat(messages)`.

---

## 4.4 Tools (LangChain4j)

**What is it?** Java methods the model can call — annotate with `@Tool`.

**Why do we need it?** Same as always: let the LLM fetch data / take actions.

**How does it work?** Put `@Tool` on methods, register the containing object via `.tools(...)` (or a `@Component` picked up by the Spring starter). LangChain4j advertises them to the model and runs the call loop.

**Java example:**

```java
class BankingTools {
    @Tool("Get the current account balance for a customer id")
    double getBalance(String customerId) {
        return 1234.56;   // real impl: query DB
    }
}

Assistant assistant = AiServices.builder(Assistant.class)
    .chatModel(model)
    .tools(new BankingTools())
    .build();

assistant.chat("What's the balance for customer C-100?");
// model calls getBalance("C-100"), then answers with the number
```

---

## 4.5 Memory (LangChain4j)

**What is it?** `ChatMemory` — stores and replays conversation history.

**Why do we need it?** Stateless model; you supply the memory.

**How does it work?** Attach a `ChatMemory` (e.g. `MessageWindowChatMemory` keeps the last N messages). For multi-user apps, use a `chatMemoryProvider` keyed by `@MemoryId`.

**Java example — per-user memory:**

```java
interface Assistant {
    String chat(@MemoryId String userId, @UserMessage String message);
}

Assistant assistant = AiServices.builder(Assistant.class)
    .chatModel(model)
    .chatMemoryProvider(userId -> MessageWindowChatMemory.withMaxMessages(20))
    .build();

assistant.chat("alice", "My name is Alice.");
assistant.chat("alice", "What's my name?");   // -> "Alice" (separate memory per userId)
```

---

## 4.6 Embeddings & Vector Stores (LangChain4j)

**What is it?** `EmbeddingModel` (text → vector) and `EmbeddingStore<TextSegment>` (stores + searches vectors). The `InMemoryEmbeddingStore` is perfect for demos; `PgVectorEmbeddingStore` (from `langchain4j-pgvector`) for production.

**Why do we need it?** Foundation of RAG (next section).

**How does it work?** Embed text, `add` to the store; at query time embed the question and `search` for nearest segments.

**Java example:**

```java
EmbeddingModel embeddingModel = OpenAiEmbeddingModel.builder()
    .apiKey(System.getenv("OPENAI_API_KEY"))
    .modelName("text-embedding-3-small")
    .build();

EmbeddingStore<TextSegment> store = new InMemoryEmbeddingStore<>();

TextSegment segment = TextSegment.from("Refunds are available within 30 days.");
store.add(embeddingModel.embed(segment).content(), segment);
```

---

## 4.7 RAG (LangChain4j)

**What is it?** Retrieval wired into an AI Service via a `ContentRetriever`.

**Why do we need it?** Answer from your documents.

**How does it work?** Build an `EmbeddingStoreContentRetriever` over your store + embedding model, then pass it to `.contentRetriever(...)` on the AI Service. Before each call, LangChain4j retrieves relevant segments and injects them into the prompt automatically.

**Java example:**

```java
ContentRetriever retriever = EmbeddingStoreContentRetriever.builder()
    .embeddingStore(store)
    .embeddingModel(embeddingModel)
    .maxResults(5)
    .minScore(0.75)
    .build();

Assistant assistant = AiServices.builder(Assistant.class)
    .chatModel(model)
    .contentRetriever(retriever)   // RAG is now automatic on every call
    .build();

assistant.chat("What is the refund window?");   // answers "30 days" from your docs
```

Full RAG walkthrough next. Project 6 rebuilds Project 5 with exactly this.

---

## 4.8 Agents (LangChain4j)

**What is it?** An AI Service with tools *is already a mini-agent* — the model loops, calling tools until it can answer. LangChain4j also provides higher-level agentic building blocks, but for most needs "AI Service + tools + memory" is your agent.

We cover agents properly in [section 8](#8-agents).

---

# 5. RAG

RAG deserves its own section because it's the single most valuable technique for business AI apps — it's how you make an LLM answer questions about **your** data.

### The problem RAG solves

The model doesn't know your internal wiki, your product catalog, or yesterday's tickets. Two bad alternatives:
- **Fine-tuning** — expensive, slow, and stale the moment data changes.
- **Stuff everything into the prompt** — impossible; it won't fit the context window and costs a fortune.

**RAG** = fetch only the *relevant* pieces at query time and give them to the model. Cheap, always current, and grounded in real text (less hallucination).

### The RAG flow

```text
        INDEXING (do this once, ahead of time)
        ─────────────────────────────────────
Documents (PDFs, wiki, tickets...)
    ↓  split
Chunks (small overlapping pieces of text)
    ↓  embed  (each chunk → vector)
Vectors
    ↓  store
Vector database (pgvector)


        QUERYING (every time a user asks)
        ─────────────────────────────────
User question
    ↓  embed  (question → vector)
Query vector
    ↓  similarity search (find nearest chunk vectors)
Top-K most relevant chunks
    ↓  build prompt: "Answer using this context: <chunks>. Question: <question>"
LLM
    ↓
Answer grounded in your documents
```

### The five terms, precisely

- **Chunk** — a small slice of a document (e.g. ~500–1000 tokens). We split because (a) whole documents don't fit context windows, and (b) smaller chunks give *precise* matches. **Overlap** (repeating a bit between chunks) avoids cutting an idea in half.
- **Embedding** — each chunk turned into a vector (meaning as numbers).
- **Vector database** — stores those vectors + the original text, and searches them fast.
- **Similarity search** — given the question's vector, find the closest chunk vectors (usually cosine distance). "Closest" ≈ "most semantically related."
- **Retrieval** — the act of pulling the top-K chunks to feed the model.

> **Why chunking + embeddings beat keyword search:** a user asking "how do I get my money back?" will match a chunk titled "Refund Policy" even with **zero shared keywords**, because their *meanings* are close in vector space. That's the superpower.

### Small RAG application — Spring Boot + Spring AI + PostgreSQL + pgvector

This is the core of Project 5. Here's the essential wiring.

**Dependencies** (add to the Spring AI setup from section 3):

```xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
</dependency>
```

**Postgres with pgvector** (Docker, for local dev):

```bash
docker run -d --name pgvector -p 5432:5432 \
  -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=appdb \
  pgvector/pgvector:pg17
```

**Config** (`application.properties`):

```properties
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.model=gpt-4o-mini
spring.ai.openai.embedding.model=text-embedding-3-small

spring.datasource.url=jdbc:postgresql://localhost:5432/appdb
spring.datasource.username=postgres
spring.datasource.password=postgres

spring.ai.vectorstore.pgvector.initialize-schema=true
spring.ai.vectorstore.pgvector.dimensions=1536
spring.ai.vectorstore.pgvector.distance-type=COSINE_DISTANCE
spring.ai.vectorstore.pgvector.index-type=HNSW
```

**Indexing service** — split documents into chunks and store them:

```java
@Service
class IngestionService {

    private final VectorStore vectorStore;
    IngestionService(VectorStore vectorStore) { this.vectorStore = vectorStore; }

    /** Read a text document, split into chunks, embed + store. */
    void ingest(String rawText, String sourceName) {
        Document doc = new Document(rawText, Map.of("source", sourceName));

        // TokenTextSplitter chunks the text (with sensible defaults) so pieces fit + match well.
        List<Document> chunks = new TokenTextSplitter().split(doc);

        // vectorStore.add() embeds each chunk and stores vector + text + metadata in pgvector.
        vectorStore.add(chunks);
    }
}
```

**Query service** — RAG via the `QuestionAnswerAdvisor`:

```java
@Service
class RagService {

    private final ChatClient chatClient;

    RagService(ChatClient.Builder builder, VectorStore vectorStore) {
        this.chatClient = builder
            // QuestionAnswerAdvisor: on each call it embeds the question,
            // searches the vector store, and injects the top chunks into the prompt.
            .defaultAdvisors(QuestionAnswerAdvisor.builder(vectorStore)
                .searchRequest(SearchRequest.builder().topK(4).similarityThreshold(0.7).build())
                .build())
            .build();
    }

    String ask(String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .content();   // grounded in retrieved chunks
    }
}
```

**Controller:**

```java
@RestController
@RequestMapping("/kb")
class KnowledgeController {
    private final IngestionService ingestion;
    private final RagService rag;

    KnowledgeController(IngestionService i, RagService r) { this.ingestion = i; this.rag = r; }

    @PostMapping("/documents")
    void upload(@RequestBody String text, @RequestParam String name) {
        ingestion.ingest(text, name);
    }

    @GetMapping("/ask")
    String ask(@RequestParam String q) { return rag.ask(q); }
}
```

That's a complete RAG app. Upload docs to `/kb/documents`, ask at `/kb/ask?q=...`, and answers come from *your* content. Project 5 fleshes this out with file uploads; Project 6 rebuilds it in LangChain4j.

---

# 6. Tool Calling

### What it really is

Tool calling (a.k.a. function calling) lets the LLM **request that your code run a function**, then use the result to finish its answer. Critically: **the LLM never runs your code.** It only *asks*; your Java executes and returns the result. You stay in control.

### The flow

```text
User asks:
"What is the weather in Tashkent?"
        ↓
LLM sees available tools: [ getWeather(city) ]
        ↓
LLM decides: "I need getWeather with city=Tashkent"
   (returns a structured tool-call request — NOT a final answer)
        ↓
Your Java application executes:  getWeather("Tashkent")  →  "24°C, sunny"
        ↓
Tool result is sent back to the LLM (as a special tool message)
        ↓
LLM writes the final answer: "It's 24°C and sunny in Tashkent — a light jacket is plenty."
        ↓
User sees the answer
```

This is a **loop**, and in Spring AI 2.0 the `ToolCallingAdvisor` runs it for you automatically (the model may even call several tools before answering).

### Why we need it

An LLM alone is frozen in its training data and can't act. Tools connect it to the live world: databases, REST APIs, calculations, sending email, creating orders. Tools are the difference between a chatbot that *talks* and an assistant that *does*.

### Small Spring Boot app with tool calling

Tools are just annotated Java methods. Keep them small and well-described — the **description is what the model reads** to decide when to call them.

```java
@Component
class AssistantTools {

    private final CustomerRepository customers;   // your normal Spring bean
    AssistantTools(CustomerRepository customers) { this.customers = customers; }

    @Tool(description = "Add two numbers together")
    int add(@ToolParam(description = "first number") int a,
            @ToolParam(description = "second number") int b) {
        return a + b;
    }

    @Tool(description = "Look up a customer's details by their ID")
    Customer getCustomer(@ToolParam(description = "the customer ID, e.g. C-100") String id) {
        return customers.findById(id).orElseThrow();
    }

    @Tool(description = "Get today's date")
    String today() {
        return LocalDate.now().toString();
    }
}
```

Wire them into a `ChatClient`:

```java
@RestController
class AssistantController {

    private final ChatClient chatClient;
    private final AssistantTools tools;

    AssistantController(ChatClient.Builder builder, AssistantTools tools) {
        this.chatClient = builder.build();
        this.tools = tools;
    }

    @GetMapping("/assistant")
    String ask(@RequestParam String q) {
        return chatClient.prompt()
            .user(q)
            .tools(tools)        // expose all @Tool methods on this bean
            .call()              // ToolCallingAdvisor runs the whole loop automatically
            .content();
    }
}
```

Now `/assistant?q=What's 15 plus 27, and what is customer C-100's name?` triggers **two** tool calls (`add` and `getCustomer`) and a single combined answer. You wrote plain methods; the framework did the orchestration. This is Project 4.

> **Good tool design (matters a lot):** clear names, precise `description`s, small focused methods, and validated inputs. The model chooses tools *entirely* from their descriptions — vague descriptions = wrong or missed calls.

---

# 7. Memory

### What's actually happening

The goal:

```text
User: My name is John.
User: What is my name?
AI:   Your name is John.
```

For this to work, the model must "remember" the first message. But remember section 1: **the model is stateless — it remembers nothing between calls.** So how?

**The trick: you resend the history every time.** "Memory" is not a model feature — it's you including the previous messages in the next request.

Behind the scenes, call #2 doesn't send just "What is my name?" It sends the whole conversation:

```text
Request actually sent on the 2nd turn:
[
  { system:    "You are a helpful assistant." },
  { user:      "My name is John." },
  { assistant: "Nice to meet you, John!" },   ← stored from turn 1
  { user:      "What is my name?" }            ← the new message
]
```

The model reads the whole list and answers "John." That's the entire secret. Frameworks just **store and replay** this list for you.

```text
                 ┌─────────── Memory store (per conversation id) ───────────┐
turn 1:  user ──►│ append user msg ─► call LLM ─► append assistant reply     │──► reply
turn 2:  user ──►│ append user msg ─► send FULL history to LLM ─► append reply│──► reply
                 └──────────────────────────────────────────────────────────┘
```

> **Practical caveat:** history grows every turn, and it all counts against the context window and your token bill. So real memory implementations **trim** — keep the last N messages (window), or **summarize** older turns into a short recap. That's why `MessageWindowChatMemory.withMaxMessages(20)` exists.

### Spring AI approach

Use `MessageChatMemoryAdvisor` and pass a **conversation id** so each user/chat is isolated.

```java
@RestController
class ChatbotController {

    private final ChatClient chatClient;

    ChatbotController(ChatClient.Builder builder) {
        // In-memory store (last 20 messages). Swap for JDBC/Redis-backed in production.
        ChatMemory memory = MessageWindowChatMemory.builder()
            .maxMessages(20)
            .build();

        this.chatClient = builder
            .defaultAdvisors(MessageChatMemoryAdvisor.builder(memory).build())
            .build();
    }

    @GetMapping("/chat")
    String chat(@RequestParam String conversationId, @RequestParam String message) {
        return chatClient.prompt()
            .user(message)
            // the id keeps each conversation's history separate
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
            .call()
            .content();
    }
}
```

`/chat?conversationId=john&message=My name is John` then `/chat?conversationId=john&message=What is my name?` → "Your name is John." Different `conversationId` = different memory. This is Project 3.

> **Production note:** the default store is in-memory (lost on restart, not shared across instances). Spring AI offers JDBC/Redis-backed memory repositories; Spring AI 2.0 also introduces the community **`spring-ai-session`** module for event-sourced conversation memory.

### LangChain4j approach (briefly)

Same idea, done through AI Services with `@MemoryId`:

```java
interface Chatbot {
    String chat(@MemoryId String conversationId, @UserMessage String message);
}

Chatbot bot = AiServices.builder(Chatbot.class)
    .chatModel(model)
    .chatMemoryProvider(id -> MessageWindowChatMemory.withMaxMessages(20))
    .build();

bot.chat("john", "My name is John.");
bot.chat("john", "What is my name?");   // -> "Your name is John."
```

Same concept (store + replay history, isolated by id), just expressed as an interface. Notice how both frameworks converge on the identical mental model — only the syntax differs.

---

# 8. Agents

### What an agent is — without the hype

Strip away the buzzwords:

> **An agent is an LLM running in a loop, deciding the next action (usually a tool call) at each step, until the task is done.**

A normal LLM call is **one shot**: question → answer. An agent is **multi-step**: it can call a tool, look at the result, decide it needs *another* tool, call that, and only then answer. The LLM itself chooses the path — you don't hard-code the steps.

```text
User gives a task:  "Email me a summary of customer C-100's recent orders."
      ↓
LLM decides:  "First I need the customer's orders."   → calls getOrders("C-100")
      ↓
Gets result: [order1, order2, order3]
      ↓
LLM decides:  "Now summarize them."                    → (reasons internally)
      ↓
LLM decides:  "Now send the email."                    → calls sendEmail(...)
      ↓
Gets result: "sent"
      ↓
Final answer: "Done — I emailed you a summary of C-100's 3 recent orders."
```

Notice: **agent = tool calling + a loop + the model deciding when it's finished.** You already know all the pieces. That's why there's no magic — an agent is the tool-calling loop from section 6, allowed to run multiple iterations.

### Why we need it

Some tasks can't be done in one call because later steps depend on earlier results ("look up the order *then* decide whether to refund"). Agents let the LLM plan and adapt at runtime instead of you writing rigid `if/else` orchestration for every possible path.

> **Reality check (no hype):** agents are powerful but non-deterministic and can loop or make poor choices. Keep tools narrow, keep the task scoped, cap the number of steps, and validate tool inputs. Don't build an "autonomous do-anything platform" — build a focused assistant with a handful of good tools. That's what actually works in production.

### A small Spring Boot agent

The beauty: with tools attached, `ChatClient` already loops. To make it a purposeful *agent*, we give it a goal (system prompt) and a few tools, and let it work.

```java
@Component
class OrderTools {

    @Tool(description = "Get all recent orders for a customer by ID")
    List<Order> getOrders(@ToolParam(description = "customer ID") String customerId) {
        return List.of(
            new Order("O-1", "Laptop",  1200.00),
            new Order("O-2", "Mouse",     25.00),
            new Order("O-3", "Keyboard",  75.00));
    }

    @Tool(description = "Calculate the total value of a list of order amounts")
    double total(@ToolParam(description = "amounts to sum") List<Double> amounts) {
        return amounts.stream().mapToDouble(Double::doubleValue).sum();
    }

    @Tool(description = "Send an email to a recipient")
    String sendEmail(@ToolParam(description = "recipient email") String to,
                     @ToolParam(description = "email body") String body) {
        // real impl: JavaMailSender
        return "Email sent to " + to;
    }
}

@RestController
class AgentController {

    private final ChatClient agent;
    private final OrderTools tools;

    AgentController(ChatClient.Builder builder, OrderTools tools) {
        this.tools = tools;
        this.agent = builder
            .defaultSystem("""
                You are an operations assistant. To complete a task you may call tools
                multiple times. Think step by step: gather data, compute, then act.
                When the task is done, reply with a short confirmation.
                """)
            .build();
    }

    @PostMapping("/agent")
    String run(@RequestBody String task) {
        return agent.prompt()
            .user(task)
            .tools(tools)     // the model loops over these tools until the task is complete
            .call()
            .content();
    }
}
```

POST `"Get customer C-100's orders, calculate the total, and email a summary to boss@acme.com"` and the agent will call `getOrders` → `total` → `sendEmail`, then confirm — deciding each step itself. This is Project 7.

> LangChain4j does the same with an AI Service that has multiple `@Tool`s attached — an AI Service with tools *is* an agent. It also offers higher-level agentic APIs, but the same "tools + loop" foundation applies.

---

# 9. Spring AI vs LangChain4j

Both do the same job — help you build LLM apps in Java — and both wrap the same underlying HTTP calls. The difference is **philosophy and feel.**

```text
Spring AI
    ↓
Feels native inside Spring Boot — auto-config, starters, properties, DI.
The "Spring way" applied to AI. Fluent ChatClient + advisor chain.

LangChain4j
    ↓
A Java-focused AI framework that runs anywhere (plain Java, Quarkus, Spring).
Signature style: declarative AI Services (you define an interface).
```

### What they have in common

- Unified API over many providers (OpenAI, Anthropic, Google, Bedrock, Ollama, …).
- All the same concepts: chat, streaming, structured output, tools, memory, embeddings, vector stores, RAG, MCP.
- Provider-swappable by changing a dependency + config, not your logic.
- Both are production-ready and actively developed.

### How they differ

| Aspect | Spring AI | LangChain4j |
|---|---|---|
| **Origin** | Official Spring project | Independent, community-driven |
| **Best fit** | Spring Boot apps | Any JVM app (plain Java, Quarkus, Spring) |
| **Core API** | Fluent `ChatClient` + **advisors** | Declarative **AI Services** (interfaces) |
| **Feel** | Imperative, builder-style calls | Declarative, "define the interface" |
| **Config** | `application.properties`, auto-config | Builders, or Spring/Quarkus starters |
| **Extensibility model** | Advisor chain (interceptors) | AI Service config + retrievers/tools |
| **Cross-cutting concerns** | Advisors (memory, RAG, tools, logging) | Wired into the AI Service builder |

### Same concepts, side by side

| Concept | Spring AI | LangChain4j |
|---|---|---|
| Low-level model | `ChatModel` | `ChatModel` |
| Main API | `ChatClient` (fluent) | `AiServices` (interface proxy) |
| Structured output | `.entity(Type.class)` | POJO return type on the method |
| Tools | `@Tool` + `.tools(...)` | `@Tool` + `.tools(...)` |
| Memory | `MessageChatMemoryAdvisor` + conversation id | `ChatMemory` + `@MemoryId` |
| RAG | `QuestionAnswerAdvisor` + `VectorStore` | `ContentRetriever` + `EmbeddingStore` |
| Streaming | `.stream()` → `Flux` | `TokenStream` return type |

### The same call in both

**Spring AI (imperative):**

```java
String answer = chatClient.prompt()
    .system("You are helpful.")
    .user("Hello")
    .call()
    .content();
```

**LangChain4j (declarative):**

```java
@AiService
interface Assistant {
    @SystemMessage("You are helpful.")
    String chat(String message);
}
// ... assistant.chat("Hello");
```

### When to choose which

- **Choose Spring AI if:** you're already all-in on Spring Boot and want the idiomatic Spring experience — auto-config, DI, properties, and the advisor chain for cross-cutting concerns. It'll feel like the rest of your app.
- **Choose LangChain4j if:** you want the elegant declarative AI-Service style, you're on Quarkus or plain Java, or you love expressing AI features as typed interfaces. Its AI Services are hard to beat for clean code.
- **Honestly:** for a Spring Boot shop, either works great. Many teams start with Spring AI for cohesion. Pick one per project; both cover everything in this guide. Understanding one makes the other easy — the concepts are identical.

---

# 10. Practical Projects

Seven projects, each adding one capability. All use the setup from sections 3–4. I'll show the essential code for each; earlier sections already gave you the detailed pieces.

---

## Project 1 — Simple AI Chat

```text
Spring Boot → Spring AI → LLM
```

A REST API that takes a question and returns an AI answer.

```java
@RestController
class ChatController {
    private final ChatClient chatClient;
    ChatController(ChatClient.Builder builder) { this.chatClient = builder.build(); }

    @GetMapping("/chat")
    String chat(@RequestParam String q) {
        return chatClient.prompt().user(q).call().content();
    }
}
```

**What you learned:** the fundamental `prompt → call → content` loop. Everything else builds on this.

---

## Project 2 — AI Text Summarizer

Receives text, returns a **structured** summary. Teaches prompts + structured output.

```java
record Summary(String title, List<String> keyPoints, String oneLineSummary, int originalWordCount) {}

@RestController
class SummarizerController {
    private final ChatClient chatClient;
    SummarizerController(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("You are an expert editor. Summarize text accurately and concisely.")
            .build();
    }

    @PostMapping("/summarize")
    Summary summarize(@RequestBody String text) {
        return chatClient.prompt()
            .user(u -> u.text("Summarize the following text:\n\n{text}").param("text", text))
            .options(OpenAiChatOptions.builder().temperature(0.2).build())  // low temp = faithful
            .call()
            .entity(Summary.class);   // structured output → typed object
    }
}
```

**What you learned:** structured output (`.entity`), prompt templating, and using low temperature for factual tasks. You get a clean `Summary` object, not text to parse.

---

## Project 3 — AI Chatbot With Memory

Conversational chatbot that remembers earlier turns.

```java
@RestController
class MemoryChatController {
    private final ChatClient chatClient;

    MemoryChatController(ChatClient.Builder builder) {
        ChatMemory memory = MessageWindowChatMemory.builder().maxMessages(20).build();
        this.chatClient = builder
            .defaultAdvisors(MessageChatMemoryAdvisor.builder(memory).build())
            .build();
    }

    @GetMapping("/chat")
    String chat(@RequestParam String conversationId, @RequestParam String message) {
        return chatClient.prompt()
            .user(message)
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
            .call()
            .content();
    }
}
```

**Test it:** two calls with the same `conversationId` — tell it your name, then ask for it. **What you learned:** memory = replaying history, isolated per conversation id (section 7).

---

## Project 4 — Tool-Calling Assistant

An assistant that calls Java methods: calculate, look up a customer, call a REST API.

```java
@Component
class AssistantTools {
    private final RestClient http = RestClient.create();

    @Tool(description = "Calculate the result of a basic arithmetic expression a <op> b")
    double calculate(@ToolParam String op, @ToolParam double a, @ToolParam double b) {
        return switch (op) {
            case "+" -> a + b; case "-" -> a - b;
            case "*" -> a * b; case "/" -> a / b;
            default -> throw new IllegalArgumentException("Unknown op: " + op);
        };
    }

    @Tool(description = "Retrieve customer information by customer ID")
    Map<String, Object> getCustomer(@ToolParam String customerId) {
        return Map.of("id", customerId, "name", "Acme Corp", "tier", "gold");  // real: DB call
    }

    @Tool(description = "Get the current price of a cryptocurrency in USD")
    String cryptoPrice(@ToolParam(description = "e.g. bitcoin") String coin) {
        return http.get()
            .uri("https://api.coingecko.com/api/v3/simple/price?ids={c}&vs_currencies=usd", coin)
            .retrieve().body(String.class);
    }
}

@RestController
class AssistantController {
    private final ChatClient chatClient;
    private final AssistantTools tools;
    AssistantController(ChatClient.Builder b, AssistantTools t) { this.chatClient = b.build(); this.tools = t; }

    @GetMapping("/assistant")
    String ask(@RequestParam String q) {
        return chatClient.prompt().user(q).tools(tools).call().content();
    }
}
```

**What you learned:** exposing methods as tools, and how the model picks and chains them (section 6). Tools connect the LLM to calculations, your database, and external APIs.

---

## Project 5 — RAG Application (Company Knowledge Assistant)

Spring Boot + Spring AI + PostgreSQL + pgvector. Upload company documents, ask questions about them.

Use the dependencies, Docker Postgres, and config from [section 5](#5-rag). Full app:

```java
@Service
class KnowledgeService {

    private final VectorStore vectorStore;
    private final ChatClient chatClient;

    KnowledgeService(VectorStore vectorStore, ChatClient.Builder builder) {
        this.vectorStore = vectorStore;
        this.chatClient = builder
            .defaultAdvisors(QuestionAnswerAdvisor.builder(vectorStore)
                .searchRequest(SearchRequest.builder().topK(4).similarityThreshold(0.7).build())
                .build())
            .defaultSystem("""
                You are the company knowledge assistant. Answer ONLY from the provided context.
                If the answer isn't in the context, say you don't have that information.
                """)
            .build();
    }

    /** Ingest an uploaded document: split → embed → store. */
    void addDocument(String content, String filename) {
        Document doc = new Document(content, Map.of("source", filename));
        List<Document> chunks = new TokenTextSplitter().split(doc);
        vectorStore.add(chunks);
    }

    /** Answer a question grounded in stored documents (RAG). */
    String ask(String question) {
        return chatClient.prompt().user(question).call().content();
    }
}

@RestController
@RequestMapping("/kb")
class KnowledgeController {
    private final KnowledgeService kb;
    KnowledgeController(KnowledgeService kb) { this.kb = kb; }

    @PostMapping("/upload")
    String upload(@RequestParam MultipartFile file) throws IOException {
        String text = new String(file.getBytes(), StandardCharsets.UTF_8);
        kb.addDocument(text, file.getOriginalFilename());
        return "Ingested: " + file.getOriginalFilename();
    }

    @GetMapping("/ask")
    String ask(@RequestParam String q) { return kb.ask(q); }
}
```

**Try it:** upload an employee-handbook `.txt`, then ask "How many vacation days do I get?" — the answer comes from your file. **What you learned:** the full RAG pipeline end-to-end (section 5). The `defaultSystem` guardrail ("answer only from context") is what keeps it from hallucinating.

---

## Project 6 — Same Application With LangChain4j

Rebuild Project 5 with LangChain4j so you can *feel* the difference on the same problem.

**Dependencies:** `langchain4j`, `langchain4j-open-ai(-spring-boot-starter)`, and `langchain4j-pgvector`.

```java
@Configuration
class RagConfig {

    @Bean
    EmbeddingModel embeddingModel() {
        return OpenAiEmbeddingModel.builder()
            .apiKey(System.getenv("OPENAI_API_KEY"))
            .modelName("text-embedding-3-small")
            .build();
    }

    @Bean
    EmbeddingStore<TextSegment> embeddingStore() {
        return PgVectorEmbeddingStore.builder()
            .host("localhost").port(5432)
            .database("appdb").user("postgres").password("postgres")
            .table("kb_embeddings").dimension(1536)
            .build();
    }

    @Bean
    ContentRetriever contentRetriever(EmbeddingStore<TextSegment> store, EmbeddingModel model) {
        return EmbeddingStoreContentRetriever.builder()
            .embeddingStore(store).embeddingModel(model)
            .maxResults(4).minScore(0.7)
            .build();
    }
}

// The AI Service — RAG is attached declaratively via the retriever bean.
@AiService(wiringMode = AiServiceWiringMode.EXPLICIT,
           chatModel = "openAiChatModel",
           contentRetriever = "contentRetriever")
interface KnowledgeAssistant {
    @SystemMessage("Answer ONLY from the provided context. If unknown, say so.")
    String ask(String question);
}

@Service
class IngestionService {
    private final EmbeddingStore<TextSegment> store;
    private final EmbeddingModel embeddingModel;
    IngestionService(EmbeddingStore<TextSegment> s, EmbeddingModel m) { this.store = s; this.embeddingModel = m; }

    void addDocument(String content, String filename) {
        Document doc = Document.from(content, Metadata.from("source", filename));
        EmbeddingStoreIngestor.builder()
            .documentSplitter(DocumentSplitters.recursive(1000, 200))  // chunk size, overlap
            .embeddingModel(embeddingModel)
            .embeddingStore(store)
            .build()
            .ingest(doc);
    }
}
```

**What changed vs. Project 5:**

| | Spring AI (P5) | LangChain4j (P6) |
|---|---|---|
| Query API | `ChatClient` + `QuestionAnswerAdvisor` | `@AiService` interface + `ContentRetriever` |
| Vector store | `VectorStore` (auto-config) | `PgVectorEmbeddingStore` (explicit bean) |
| Ingestion | `TokenTextSplitter` + `vectorStore.add` | `EmbeddingStoreIngestor` + `DocumentSplitters` |
| Style | Imperative call chain | Declarative interface |

**What stayed the same (the important part):** the *concepts* are identical — split → embed → store → retrieve → inject → answer. Same PostgreSQL, same pgvector, same embeddings, same RAG idea. **Only the syntax differs.** That's the whole lesson of this project: frameworks are interchangeable vocabulary for the same underlying model.

---

## Project 7 — Simple AI Agent

A small agent that uses several tools to complete a multi-step task. This is the agent from [section 8](#8-agents) — `OrderTools` (get orders, sum totals, send email) attached to a `ChatClient` with a goal-oriented system prompt. Re-read section 8 for the full code.

**What you learned:** an agent = tools + loop + the model deciding when it's done. Keep the scope tight and the tools focused.

---

# 11. MCP — Simple Introduction

### What MCP is

**MCP (Model Context Protocol)** is an open standard for connecting AI applications to external tools and data through a **uniform protocol**. Think of it as **"USB-C for AI tools"** — one standard plug so any MCP-compatible AI app can use any MCP-compatible tool server.

### Why it exists

Without MCP, every integration is bespoke: you write custom tool code for Slack, then again for GitHub, then again for your database, and each AI app re-implements them. MCP standardizes this. A tool provider writes **one MCP server**, and **any** MCP client (Claude Desktop, your Spring AI app, an IDE) can use it — no custom glue. It turns tools/data sources into reusable, shareable plugins.

### The pieces

```text
   ┌─────────────────┐         MCP protocol          ┌─────────────────┐
   │   MCP Client    │◄────────(JSON-RPC over────────►│   MCP Server    │
   │  (your AI app)  │      stdio or HTTP stream)      │ (tool provider) │
   └─────────────────┘                                 └────────┬────────┘
        the LLM can now                                          │ exposes
        use the server's tools                          ┌────────┴────────┐
                                                         │  Tools  │ Resources
                                                         └─────────┴────────┘
```

- **MCP client** — lives in your AI application; connects to servers and makes their capabilities available to the LLM (as tools).
- **MCP server** — a separate process/service that exposes capabilities. Could be official (GitHub, filesystem, Postgres) or one you write.
- **Tools** — actions the server offers (functions the LLM can call), e.g. `create_issue`, `run_query`.
- **Resources** — data the server exposes for reading, e.g. a file's contents, a DB record. (There are also **prompts** — reusable prompt templates.)

The payoff: **tool calling (section 6), but the tools live in an external, reusable server** instead of inside your app.

### Tiny example — Spring AI as an MCP client

Spring AI has MCP support. Add the client starter and point it at a server; the server's tools become available to your `ChatClient` just like local `@Tool`s.

```xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-starter-mcp-client</artifactId>
</dependency>
```

```properties
# Connect to an MCP server (Streamable HTTP is the default transport in Spring AI 2.0)
spring.ai.mcp.client.streamable-http.connections.github.url=http://localhost:8080
```

```java
@RestController
class McpController {
    private final ChatClient chatClient;

    // Spring AI collects tools from connected MCP servers into a ToolCallbackProvider.
    McpController(ChatClient.Builder builder, ToolCallbackProvider mcpTools) {
        this.chatClient = builder.defaultToolCallbacks(mcpTools).build();
    }

    @GetMapping("/mcp-ask")
    String ask(@RequestParam String q) {
        // The LLM can now call tools exposed by the MCP server, transparently.
        return chatClient.prompt().user(q).call().content();
    }
}
```

### Tiny example — writing an MCP server (Spring AI 2.0)

Spring AI 2.0 added annotations. Expose a method as an MCP tool other AI apps can use:

```java
@Component
class WeatherMcpServer {

    @McpTool(description = "Get the current weather for a city")
    String getWeather(@McpToolParam(description = "city name") String city) {
        return "Weather in " + city + ": 24°C, sunny";
    }
}
```

Add `spring-ai-starter-mcp-server`, and this tool is now callable by any MCP client. **That's the essence** — one standard so tools are written once and reused everywhere. (We're deliberately not going deeper into the protocol; you rarely need to.)

---

# 12. Final Project — AI Customer Support Assistant

The capstone that ties **everything** together: Spring Boot + Spring AI + LLM + conversation memory + RAG + PostgreSQL/pgvector + tool calling.

```text
                 Customer
                    ↓
              REST API
                    ↓
              Spring Boot
                    ↓
              Spring AI  (ChatClient + advisor chain)
              /    |    \
             /     |     \
           LLM    RAG    Tools
                   |       |
             PostgreSQL   APIs / DB
              pgvector
```

**What it does:** a customer asks a support question. The assistant:
1. **Remembers** the conversation (memory advisor).
2. **Retrieves** relevant policy/FAQ text from company docs (RAG advisor over pgvector).
3. **Acts** when needed — look up the customer's orders, check order status, create a support ticket (tools).
4. Produces a grounded, context-aware answer.

This is just the pieces you've already built, **composed in one `ChatClient`** via the advisor chain plus tools. That composition is the entire point — you're not learning anything new here, you're assembling.

### The tools

```java
@Component
class SupportTools {
    private final OrderRepository orders;
    private final TicketRepository tickets;
    SupportTools(OrderRepository o, TicketRepository t) { this.orders = o; this.tickets = t; }

    @Tool(description = "Get a customer's recent orders by their customer ID")
    List<Order> getOrders(@ToolParam(description = "customer ID") String customerId) {
        return orders.findByCustomerId(customerId);
    }

    @Tool(description = "Get the delivery status of a specific order by order ID")
    String orderStatus(@ToolParam(description = "order ID") String orderId) {
        return orders.findById(orderId).map(Order::status).orElse("Order not found");
    }

    @Tool(description = "Create a support ticket for a customer issue")
    String createTicket(@ToolParam String customerId, @ToolParam String issue) {
        Ticket t = tickets.save(new Ticket(customerId, issue, "OPEN"));
        return "Created ticket " + t.getId();
    }
}
```

### The assistant service (memory + RAG + tools together)

```java
@Service
class SupportAssistant {

    private final ChatClient chatClient;
    private final SupportTools tools;

    SupportAssistant(ChatClient.Builder builder,
                     VectorStore vectorStore,   // company docs live here (pgvector)
                     ChatMemory chatMemory,
                     SupportTools tools) {
        this.tools = tools;
        this.chatClient = builder
            .defaultSystem("""
                You are AcmeShop's customer support assistant.
                - Use retrieved policy documents to answer policy questions accurately.
                - Use tools to look up real order data and to create tickets.
                - If you cannot resolve an issue, create a support ticket.
                - Be warm, concise, and never invent policies or order details.
                """)
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory).build(),          // MEMORY
                QuestionAnswerAdvisor.builder(vectorStore)                     // RAG
                    .searchRequest(SearchRequest.builder().topK(4).similarityThreshold(0.7).build())
                    .build()
            )
            .build();
    }

    String handle(String conversationId, String message) {
        return chatClient.prompt()
            .user(message)
            .tools(tools)                                                      // TOOLS
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
            .call()
            .content();
    }
}
```

### The controller

```java
@RestController
@RequestMapping("/support")
class SupportController {
    private final SupportAssistant assistant;
    SupportController(SupportAssistant a) { this.assistant = a; }

    @PostMapping("/chat")
    ChatResponse chat(@RequestBody ChatRequest req) {
        String reply = assistant.handle(req.conversationId(), req.message());
        return new ChatResponse(reply);
    }

    record ChatRequest(String conversationId, String message) {}
    record ChatResponse(String reply) {}
}
```

### How it all fits — a walkthrough

Customer (conversation `cust-42`) says: *"My order O-500 hasn't arrived — what's your delivery policy and where is it?"*

1. **Memory advisor** loads any prior turns for `cust-42` and prepends them.
2. **RAG advisor** embeds the question, searches pgvector, and injects the *delivery policy* chunk into the prompt.
3. The **LLM** reads system prompt + history + policy context + question, and decides it needs live data → calls the **tool** `orderStatus("O-500")`.
4. The **ToolCallingAdvisor** runs it against PostgreSQL → `"In transit, arriving tomorrow"`, feeds the result back.
5. The **LLM** composes the final answer: explains the delivery policy (from RAG) *and* the live status (from the tool), in a friendly tone.
6. **Memory advisor** stores this turn so the next message has context.

Every concept from this guide — messages, tokens/context, embeddings, vector DB, RAG, tools, memory, the advisor chain — collaborating in one request. **That's a real AI application.** And notice: you built it out of small, familiar pieces. There was never any magic.

> **Production hardening (beyond scope, but know they exist):** rate limiting, cost/token budgets, prompt-injection defenses (especially with tools + RAG), PII handling, evaluation/testing of answers, observability (Spring AI integrates with Micrometer), and human-in-the-loop for sensitive tool actions like refunds.

---

# 13. Learning Style Recap

The method that makes any new AI concept click — apply it to anything you meet next:

### What is it?
Plain definition, no jargon.

### Why do we need it?
The concrete problem it solves (usually an LLM limitation: stateless, no private knowledge, can't act, costs tokens).

### How does it work?
The internal flow — almost always *"manipulate the message list, then make the HTTP call."*

### Java example
The smallest runnable snippet.

### Project example
Where it lives in a real app.

Avoid: unnecessary theory, giant API lists, AI math, and enterprise-architecture rabbit holes. The goal is to **understand and build**, not memorize docs.

---

# 14. Final Summary — Mental Map

Everything branches from the LLM:

```text
LLM  (a stateless, non-deterministic text function you call over HTTP)
 |
 +-- Prompt              (what you send: system + user + assistant messages)
 |
 +-- Chat                (request/response; streaming for live UX)
 |
 +-- Structured Output   (get a typed Java object, not a String)
 |
 +-- Embeddings          (text → vector; meaning as numbers)
 |      |
 |   Vector DB           (store + similarity-search vectors — pgvector)
 |      |
 |     RAG               (retrieve your data, inject into prompt → grounded answers)
 |
 +-- Tools               (LLM asks; your Java runs functions → it can ACT)
 |
 +-- Memory              (resend history → conversations that remember)
 |
 +-- Agents              (tools + a loop; the LLM decides the next step)
 |
 +-- MCP                 (standard protocol so tools/data are reusable across apps)
```

And where the two frameworks fit:

```text
                AI Application
                      |
             +--------+--------+
             |                 |
        Spring AI         LangChain4j
      (ChatClient +      (AI Services /
        advisors)          interfaces)
             |                 |
             +--------+--------+
                      |
                     LLM
                      |
             OpenAI / Gemini /
             Anthropic / Bedrock / Ollama
```

### The one thing to remember

> Every AI feature in this guide is **the same HTTP-call-to-a-text-function**, dressed up:
> **memory** resends the message list, **RAG** injects retrieved text into it, **tools** let the model request a function and read its result, **agents** loop that, and **MCP** shares tools across apps.
> Spring AI and LangChain4j are two idiomatic Java ways to write that plumbing. Learn the concepts once; the frameworks are just vocabulary.

**You now understand the core AI application concepts and can build practical AI applications with Java, Spring Boot, Spring AI, and LangChain4j.** Start with Project 1, work through to the capstone, and you'll have hands-on command of all of it.

---

## Sources (official docs, verified September 2026)

- [Spring AI 2.0.0 GA announcement](https://spring.io/blog/2026/06/12/spring-ai-2-0-0-GA-available-now/)
- [Spring AI Reference — ChatClient API](https://docs.spring.io/spring-ai/reference/api/chatclient.html)
- [Spring AI Reference — Upgrade Notes (2.0 changes)](https://docs.spring.io/spring-ai/reference/upgrade-notes.html)
- [Spring AI Reference — pgvector](https://docs.spring.io/spring-ai/reference/api/vectordbs/pgvector.html)
- [Spring AI — Getting Started](https://docs.spring.io/spring-ai/reference/getting-started.html)
- [LangChain4j — Get Started](https://docs.langchain4j.dev/get-started/)
- [LangChain4j — AI Services](https://docs.langchain4j.dev/tutorials/ai-services/)
- [LangChain4j — RAG](https://docs.langchain4j.dev/tutorials/rag/)
- [LangChain4j GitHub](https://github.com/langchain4j/langchain4j)
