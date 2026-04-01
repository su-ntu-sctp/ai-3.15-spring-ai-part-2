# Lesson: Coaching: Spring AI Part 2 — Structured Output and Conversation Memory

## Lesson Overview

This is the second Spring AI coaching session. We continue building on the `spring-ai-demo` project from the previous session. In this lesson we go beyond basic chat endpoints and explore two powerful features: getting the AI to return structured Java objects, and giving the AI a memory so it can remember previous messages in a conversation — just like ChatGPT does.

**Prerequisites:** Spring AI basics (Lesson 3.12) — project setup, `ChatClient`, basic `/chat` endpoint, system prompts

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

### Creating a Response Class

Open your `spring-ai-demo` project. Let's create a scenario: we want the AI to generate a customer profile based on a job title. The response should be a structured object, not a paragraph of text.

First, create a `CustomerProfile.java` record. A Java `record` is a concise way to define a data class — it automatically generates the constructor, getters, `equals`, `hashCode`, and `toString` for us.

```java
public record CustomerProfile(
    String firstName,
    String lastName,
    String email,
    String jobTitle,
    String company
) {}
```

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

Let's break down what is new here:

- `.user(u -> u.text("...").param("jobTitle", jobTitle))` — this is a **prompt template** with a `{jobTitle}` placeholder that gets replaced with the actual value at runtime. This is cleaner than string concatenation.
- `.entity(CustomerProfile.class)` — instead of `.content()` which returns a `String`, `.entity()` tells Spring AI to map the response directly to our `CustomerProfile` class. Under the hood, Spring AI instructs the model to return valid JSON matching the structure of our record, then deserialises it automatically.

Run the application and test:

```
localhost:8080/generate-customer?jobTitle=Software Engineer
```

You should see a proper JSON response — not a paragraph of text — that maps to your `CustomerProfile` fields. Spring Boot automatically serialises the `CustomerProfile` object to JSON for the HTTP response.

Try different job titles and observe how the AI generates different but consistently structured responses every time.

### Why This is Powerful

Without structured output, if you asked the AI "generate a customer profile" you would get a paragraph like:
> *"Here is a customer profile: John Smith is a Software Engineer at TechCorp. His email is john.smith@techcorp.com..."*

With structured output, you get a proper Java object that you can immediately use in your application — pass to a service, save to a database, or return as a clean API response. This is how real AI-powered applications work.

---

### 🧑‍💻 Activity 1 **(15 minutes)**

Create a new record called `ProductRecommendation` with the following fields:

```java
public record ProductRecommendation(
    String productName,
    String category,
    String targetAudience,
    double estimatedPrice
) {}
```

Add a new endpoint `/recommend-product` that accepts a `@RequestParam String interest` and asks the AI to recommend a product for someone with that interest. Return the result as a `ProductRecommendation` object.

Test with a few different interests (e.g. `photography`, `gaming`, `cooking`) and observe the structured responses.

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

### How Chat Memory Works

Spring AI's Chat Memory works by storing the conversation history and automatically injecting it into every new request as context. This is done through a concept called an **Advisor** — a component that intercepts the request before it reaches the model and enriches it with the conversation history.

Spring AI auto-configures a `ChatMemory` bean for us by default, so we don't need to add any extra dependencies.

### Adding Memory to Our Chat Endpoint

Create a new controller `MemoryChatController.java`.

```java
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

Let's break this down:

- `ChatMemory chatMemory` — Spring AI auto-configures this bean. It stores conversation history in memory (lost when the app restarts, which is fine for now).
- `MessageChatMemoryAdvisor.builder(chatMemory).build()` — this is the advisor that automatically retrieves conversation history and injects it into every request before it reaches the model.
- `.defaultAdvisors(...)` — registers the memory advisor as a default for all calls made by this `ChatClient`.
- `ChatMemory.CONVERSATION_ID` — each conversation needs a unique ID so the memory knows which history to retrieve. We pass this as a `sessionId` query parameter so we can test multiple separate conversations.

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

### 🧑‍💻 Activity 2 **(15 minutes)**

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

- **Structured Output** — use `.entity(MyClass.class)` to get the AI to return a proper Java object instead of plain text. Combine with prompt templates using `.param()` for cleaner, dynamic prompts.
- **Conversation Memory** — use `MessageChatMemoryAdvisor` with Spring AI's auto-configured `ChatMemory` bean to give the AI a persistent conversation history. Pass a `CONVERSATION_ID` to manage separate conversations independently.

These two features are the building blocks of real-world AI-powered applications. In the next Spring AI session we will explore **RAG (Retrieval Augmented Generation)** — teaching the AI to answer questions using your own documents and data.

---

END