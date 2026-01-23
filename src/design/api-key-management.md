# API Key Management

In an enterprise cloud environment, API keys and session management need to be decoupled. 

Here is the breakdown of how to handle Context, Sessions, and API keys in an "Agent-as-a-Service" architecture.

---

### 1. API Key vs. Session Identity
*   **The API Key** is for **Authentication & Billing**: It tells the LLM provider (like OpenAI or Anthropic) which *company* is calling and who to bill. You typically use one (or a few) "Service Account" keys for the entire cloud application.
*   **The Session/Thread ID** is for **Context**: LLMs are "stateless." They don't remember you from the last call unless you provide a **Thread ID** (if using a provider's built-in memory like OpenAI Assistants API) or send the **Chat History** back with every new request.

### 2. How to Manage Context in the Cloud
Since you are designing an enterprise service, you have three professional ways to manage context across the lifecycle of a business process:

#### A. The "Stateless" Managed History (Recommended for Privacy)
In this model, your "Agent Service" stores the conversation history in your own enterprise database (e.g., Redis, PostgreSQL, or CosmosDB).
1.  Agent Instance starts.
2.  It fetches the `session_id` history from **your** DB.
3.  It sends the history + the new prompt to the LLM using the **Global API Key**.
4.  It saves the new response back to **your** DB.
*   **Benefit:** You own the data. If you switch from OpenAI to Anthropic, you still have your history.

#### B. The "Provider-Side" Threading (OpenAI Assistants API)
1.  Your service creates a `thread_id` on OpenAI’s servers.
2.  Each time the agent instance runs, it just sends the `thread_id`.
3.  OpenAI manages the context internally.
*   **Benefit:** Simpler to code.
*   **Drawback:** You are "locked in" to that provider, and your enterprise data stays on their servers longer.

---

### 3. Managing API Keys in Enterprise Cloud
You should never bake API keys into your agent code or pass them around as variables. Use these three enterprise patterns:

#### Pattern 1: Secret Management (The Gold Standard)
Use a service like **AWS Secrets Manager**, **Azure Key Vault**, or **HashiCorp Vault**.
*   The Agent Service is assigned an **IAM Role** (Identity).
*   When an instance starts, it asks the Vault for the key: *"I am Agent-Service-Production, give me the OpenAI-Key."*
*   The key only exists in the instance's **memory**, never in the logs or the code.

#### Pattern 2: The "AI Gateway" Proxy (Best for Governance)
In large enterprises, you don't let agents call OpenAI directly. You build a **Gateway Service**.
*   Agent Instance calls `https://internal-ai-gateway.enterprise.com`.
*   The API key is pushed/pulled to the target server from Secrets Manager.
*   The Gateway adds the API key to the header.
*   **Why?** You can perform **Rate Limiting** (stop an agent from spending $10k in an hour), **PII Filtering** (stop the agent from sending customer Social Security numbers to the cloud), and **Logging**.

#### Pattern 3: Key Rotation
By using a Secret Manager or Gateway, you can change the API key every 30 days without ever restarting or redeploying your agent code.

---

### 4. How this fits into your `workflow.json`
Since you are designing a spec, you can handle this by defining the "Authentication Policy" in the `use` section of your workflow. 

Your spec already has an `authenticationPolicy`. In the cloud, you would configure it to point to a **secret reference** rather than a literal string:

```json
"use": {
  "authentications": {
    "openAiAuth": {
      "bearer": {
        "token": "${ secrets.OPENAI_API_KEY }" 
      }
    }
  },
  "agents": {
    "researchAgent": {
      "model": {
        "provider": "openai",
        "name": "gpt-4o",
        "authentication": { "use": "openAiAuth" }
      },
      "memory": {
        "type": "thread",
        "id": "${ context.sessionId }" 
      }
    }
  }
}
```

### Summary for your Architecture:
1.  **API Keys:** Use **one** key per environment, stored in a **Secret Vault**.
2.  **Context:** Use a **Session ID** or **Thread ID** passed in the workflow context.
3.  **Deployment:** Each "Agent-as-a-Service" call receives a `sessionId`. The service uses that ID to look up the "state" of the business process, regardless of which cloud container is actually running the code.

