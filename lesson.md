# Lesson: Coaching: Spring AI Part 2 — Structured Output and Conversation Memory

## Lesson Overview

This is the second Spring AI coaching session. We continue building on the `spring-ai-demo` project from the previous session. In this lesson we go beyond basic chat endpoints and explore two powerful features: getting the AI to return structured Java objects, and giving the AI a memory so it can remember previous messages in a conversation — just like ChatGPT does.

**Prerequisites:** Spring AI basics (Lesson 3.12) — project setup, `ChatClient`, basic `/chat` endpoint, system prompts

> ⚙️ **Version Check:** Before starting, confirm your `pom.xml` BOM is on Spring AI `1.1.7` — the latest stable release. If you are on an older version, update it now to avoid any API mismatches with this lesson.

## Lesson Objectives

By the end of this lesson, students will be able to:

1. **Use** structured output to map AI responses directly to Java objects
2. **Implement** conversation memory so the AI maintains context across multiple messages

---

## Part 1: Structured Output

### The Problem with Plain Text Responses

In Lesson 3.12, our `/chat` endpoint returned a plain `String`. This works for displaying text, but what if we want to use the AI's response programmatically — for example, extract a name, a price, or a list of items and pass them to another method?

Plain text is hard to work with in code. We would have to parse it manually, which is messy and fragile.

**Structured Output** solves this. It tells the AI exactly what format to return its response in, and Spring AI automatically maps that response to a Java object for us.

---

### Creating a Response Class — Using a Java `record`

Open your `spring-ai-demo` project. Let's create a scenario: we want the AI to generate a customer profile based on a job title. The response should be a structured object, not a paragraph of text.

First, create a `CustomerProfile.java` record inside `src/main/java/sg/edu/ntu/`:

```java
package sg.edu.ntu;

public record CustomerProfile(
    String firstName,
    String lastName,
    String email,
    String jobTitle,
    String company
) {}
```

#### What is a Java `record`?

A `record` is a special class type introduced in Java 16 designed for holding data. When you declare a record, the Java compiler automatically generates:

- A constructor that accepts all fields
- Getters for all fields (e.g. `firstName()`, `lastName()`)
- `equals()`, `hashCode()`, and `toString()`

**Why use a `record` instead of a regular class?**

Records are **immutable** — once created, the values inside cannot be changed. This makes them a perfect fit for AI response objects. The data comes back from the model, gets mapped into the record, and then flows through your application without being accidentally modified. In a real Spring application you would also use records for DTOs (Data Transfer Objects) — objects that carry data between layers.

Compare the two approaches:

```java
// Regular class — verbose, mutable, easy to accidentally modify
public class CustomerProfile {
    private String firstName;
    // ... constructor, getters, setters, equals, hashCode, toString...
}

// Record — concise, immutable, purpose-built for data
public record CustomerProfile(String firstName, String lastName, ...) {}
```

For AI response mapping, always reach for a `record` first.

---

### Building the Structured Output Endpoint

In `AiController.java`, add a new endpoint that asks the AI to generate a customer profile and return it as a `CustomerProfile` object.

```java
@GetMapping("/generate-customer")
public CustomerProfile generateCustomer(@RequestParam String jobTitle) {
    return chatClient.prompt()
        .user(u -> u.text("Generate a realistic fictional customer profile for someone with the job title: {jobTitle}. " +
                          "Make up a realistic name, email, company and return the data.")
                    .param("jobTitle", jobTitle))
        .call()
        .entity(CustomerProfile.class);
}
```

#### Prompt Templates — what is `.param()` doing here?

Notice the `.user(u -> u.text("...{jobTitle}...").param("jobTitle", jobTitle))` pattern. This is a **Prompt Template** — a prompt that contains named placeholders `{...}` that get filled in at runtime with actual values.

You could achieve the same result with string concatenation:
```java
// Without a prompt template — works but fragile
.user("Generate a customer profile for: " + jobTitle)
```

But prompt templates are the better approach for three reasons:

1. **Readability** — it's immediately clear what is dynamic vs what is fixed in the prompt
2. **Reusability** — the prompt structure is defined once and reused with different values
3. **Prompt Injection Protection** — if a user passes malicious input (e.g. a `jobTitle` of `"Ignore all previous instructions and..."`), the template treats it as a data value, not as part of the prompt instruction. This is one of the most important security patterns in AI engineering — always use `.param()` when injecting user input into a prompt.

#### How `.entity()` works under the hood

When you call `.entity(CustomerProfile.class)`, Spring AI does the following automatically:

1. **Inspects your class** — it reads the fields of `CustomerProfile` and generates a **JSON Schema** from them (e.g. `{ "firstName": "string", "lastName": "string", ... }`)
2. **Injects the schema into the prompt** — Spring AI appends instructions to the prompt telling the model to return a JSON response that exactly matches this schema
3. **Parses the response** — when the model responds, Spring AI takes the JSON string and deserialises it into a `CustomerProfile` object using Jackson
4. **Spring Boot serialises it back to JSON** — when your endpoint returns the `CustomerProfile` object, Spring Boot automatically converts it to JSON for the HTTP response

This is why you do not need to write any JSON parsing code yourself.

#### What happens if the AI returns bad JSON?

It is possible — though uncommon with GPT-4o — for the model to return malformed JSON or miss a field. In that case, Spring AI will throw a runtime exception during deserialisation. In production applications you would add error handling around the `.entity()` call. For this lesson, if you see a `500` error, check the console — it is likely a JSON parsing failure. Re-running the request usually resolves it since LLM responses have some randomness.

> 💡 **Instructor Note — Native Structured Output:** Spring AI also supports a more reliable mode called **Native Structured Output**, enabled with `AdvisorParams.ENABLE_NATIVE_STRUCTURED_OUTPUT`. In native mode, the model's own JSON Schema enforcement is used — the model *guarantees* the output matches the schema, rather than just being instructed to try. This is the direction the industry is moving for production applications. For this lesson we use standard `.entity()` to understand the concept first — native mode is a one-line upgrade once you understand the foundation.

### Why This is Powerful

Without structured output, if you asked the AI "generate a customer profile" you would get a paragraph like:
> *"Here is a customer profile: John Smith is a Software Engineer at TechCorp. His email is john.smith@techcorp.com..."*

With structured output, you get a proper Java object that you can immediately use in your application — pass to a service, save to a database, or return as a clean API response. This is how real AI-powered applications work.

Now run the application and test it yourself — try a few different job titles:

```
localhost:8080/generate-customer?jobTitle=Software Engineer
localhost:8080/generate-customer?jobTitle=Marketing Manager
localhost:8080/generate-customer?jobTitle=Data Scientist
```

Notice that every response is a clean, consistently structured JSON object — not a paragraph of text. This is the foundation that makes AI responses usable in real application code.

---

## Part 2: Conversation Memory

### The Problem — LLMs are Stateless

By default, every message you send to an LLM is completely independent. The model has no memory of previous messages. This is why if you ask our `/chat` endpoint "What is Java?" and then follow up with "Can you give me an example?", the AI has no idea what "it" refers to.

Try it now with your existing `/chat` endpoint:

```
localhost:8080/chat?message=My name is Bruce Banner
localhost:8080/chat?message=What is my name?
```

The second call will return something like "I don't know your name" — because every request starts fresh. This is called being **stateless**.

Real chat applications like ChatGPT feel natural because they remember the conversation history. Spring AI makes this easy to implement with **Chat Memory**.

---

### How Chat Memory Works — The Advisor Pattern

Spring AI's Chat Memory is built on a concept called **Advisors**. Before we write the code, you need to understand what an Advisor is.

#### What is an Advisor?

An **Advisor** is Spring AI's equivalent of middleware or an interceptor. It sits in between your code and the AI model, and it can:

- **Intercept the request** before it reaches the model — to enrich it, modify it, or add context
- **Intercept the response** after it comes back from the model — to log it, transform it, or store it

Think of it as a pipeline:

```
Your Code → [Advisor 1] → [Advisor 2] → AI Model → [Advisor 2] → [Advisor 1] → Your Code
```

You can chain multiple advisors together. Each one runs in order before the model call, and in reverse order after. This is the **Advisor Chain**.

In this lesson, we use `MessageChatMemoryAdvisor` — an advisor that:
1. **Before the request:** retrieves the conversation history and injects it into the prompt
2. **After the response:** saves the new message and the model's reply back into memory

Spring AI auto-configures a `ChatMemory` bean for us by default, so we don't need to add any extra dependencies.

#### What does `ChatMemory` actually store?

Spring AI's default memory implementation is called `MessageWindowChatMemory`. It stores the **full message objects** — both the user messages and the assistant replies — in a sliding window. The default window size is **20 messages**. Once the conversation exceeds 20 messages, the oldest ones are dropped to keep the window size fixed.

This is important to understand: the AI does not truly "remember" — on every new request, the full message history (up to 20 messages) is sent to the model as context alongside the new message. The model reads all of it and responds accordingly. This is exactly how ChatGPT works.

> ⚠️ **Production Insight — Token Cost:** Because the full conversation history is sent on every request, longer conversations cost significantly more tokens. A 20-message conversation sends all 20 messages to the model on the 21st call. In production applications, token budgeting and memory window sizing are important cost control decisions. For this lesson, in-memory storage is fine — but be aware it is lost when the application restarts.

---

### Adding Memory to Our Chat Endpoint

Create a new controller `MemoryChatController.java` inside `src/main/java/sg/edu/ntu/`:

```java
package sg.edu.ntu;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.MessageChatMemoryAdvisor;
import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class MemoryChatController {

  private final ChatClient chatClient;

  public MemoryChatController(ChatClient.Builder chatClientBuilder, ChatMemory chatMemory) {
    this.chatClient = chatClientBuilder
        .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
        .build();
  }

  @GetMapping("/memory-chat")
  public String memoryChat(@RequestParam String message,
                           @RequestParam(defaultValue = "default-session") String sessionId) {
    return chatClient.prompt()
        .user(message)
        .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, sessionId))
        .call()
        .content();
  }
}
```

#### Breaking this down

**Constructor:**

- `ChatMemory chatMemory` — Spring AI auto-configures this bean using `MessageWindowChatMemory` backed by an in-memory store. You receive it via constructor injection — no extra setup needed.
- `MessageChatMemoryAdvisor.builder(chatMemory).build()` — creates the memory advisor wired to our `ChatMemory` store.
- `.defaultAdvisors(...)` — registers the advisor as the **default** for every call made by this `ChatClient`. You set this once at build time and it applies automatically.

**Why two places for advisors — `.defaultAdvisors()` vs `.advisors()`?**

This is a common point of confusion. Here is the distinction:

| | Where | When it runs |
|---|---|---|
| `.defaultAdvisors()` | On the `ChatClient.Builder` (constructor) | Registered once, applies to **every call** automatically |
| `.advisors(a -> a.param(...))` | On the `.prompt()` chain (per request) | Used to pass **runtime parameters** into the already-registered advisor |

In our code: the `MessageChatMemoryAdvisor` is registered once via `defaultAdvisors()`. But it needs to know *which conversation's history* to retrieve — and that changes per request. So we pass the `CONVERSATION_ID` at runtime via `.advisors(a -> a.param(...))`. The advisor is the same; only the parameter changes.

**`ChatMemory.CONVERSATION_ID`:**

This is the key that tells the memory advisor which conversation to load. Each unique ID has its own independent history. In a real application, this would be a UUID generated when the user starts a new chat session — not a hardcoded string. In this lesson we pass it as a query parameter so you can test multiple conversations independently.

> ⚠️ **Important:** The `sessionId` must always be provided. We set `defaultValue = "default-session"` for convenience during testing, but in production you should always generate and manage unique session IDs explicitly — otherwise different users could accidentally share the same conversation memory.

---

### Testing Conversation Memory

Run the application and test — use the same `sessionId` across multiple calls to simulate a real conversation.

```
localhost:8080/memory-chat?message=My name is Bruce Banner&sessionId=session1
localhost:8080/memory-chat?message=What is my name?&sessionId=session1
localhost:8080/memory-chat?message=What do I do for work?&sessionId=session1
```

The AI should remember your name from the first message and reference it in subsequent responses.

Now try a different session ID:

```
localhost:8080/memory-chat?message=What is my name?&sessionId=session2
```

This should return "I don't know your name" — because `session2` has its own separate memory with no history yet. Each conversation ID has its own independent context.

### The "ChatGPT Feel"

This is exactly how ChatGPT and similar applications work at a high level — each conversation has a unique ID, and the history of that conversation is sent along with every new message. Spring AI handles all of this complexity for us with just a few lines of code.

---

### 🧑‍💻 Activity **(15 minutes)**

Build a memory-enabled **CRM assistant** endpoint `/crm-assistant` in `MemoryChatController.java` that:

1. Has a **system prompt** making it a helpful CRM assistant (reuse what you learned in Lesson 3.12)
2. Supports **conversation memory** so it remembers what was discussed
3. Accepts a `sessionId` parameter to support multiple separate conversations

Test it with a multi-turn conversation — for example:

```
/crm-assistant?message=I have a customer named Tony Stark who is a CEO&sessionId=crm1
/crm-assistant?message=What is his job title?&sessionId=crm1
/crm-assistant?message=Draft a follow-up email for him&sessionId=crm1
```

**Hint:** Combine `.system("...")` and `.defaultAdvisors(...)` together in the `ChatClient.Builder`.

---

## Summary

In this session you added two significant capabilities to your Spring AI application:

- **Structured Output** — use `.entity(MyClass.class)` to get the AI to return a proper Java object instead of plain text. Spring AI generates a JSON Schema from your class, instructs the model to match it, and deserialises the result automatically. Combine with prompt templates using `.param()` for cleaner, injection-safe, dynamic prompts.
- **Conversation Memory** — use `MessageChatMemoryAdvisor` with Spring AI's auto-configured `ChatMemory` bean to give the AI a persistent conversation history. The default `MessageWindowChatMemory` holds up to 20 messages per conversation. Pass a `CONVERSATION_ID` via `.advisors()` at runtime to manage separate conversations independently. Every message in history is sent to the model on every call — so memory has a real token cost in production.

These two features are the building blocks of real-world AI-powered applications. In the next Spring AI session we will explore **RAG (Retrieval Augmented Generation)** — teaching the AI to answer questions using your own documents and data.

---

END