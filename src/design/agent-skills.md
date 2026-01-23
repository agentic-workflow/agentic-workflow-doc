# Agent Skills

In an enterprise Java environment, **Option 2 (Database storage) is significantly better**, but with a specific implementation pattern: **The "Library Reference" pattern.**

While Markdown is the best format for the LLM to *read*, JSON is the best format for your Java engine to *orchestrate*. You should store the Markdown content in the database and reference it in your JSON workflow.

Here is the breakdown of why Option 2 wins and how to implement the hierarchy.

---

### Why Option 2 (Database) is better than Option 1 (Inline JSON)

#### 1. Avoids "Escaping Hell"
Markdown relies heavily on newlines, quotes, and special characters. In Option 1, you would have to escape all of these inside a JSON string:
*   **JSON:** `"instructions": "### Skill Name\n1. Do this\n2. \"Don't\" do that..."`
This is unreadable and very prone to syntax errors. Storing the raw Markdown in a DB `CLOB` or `TEXT` field preserves the formatting perfectly.

#### 2. Skill Reusability (The "DRY" Principle)
In an enterprise, you will have "Common Skills" (e.g., *Corporate Tone*, *Security Guidelines*, *Logging Standards*) that every agent must have.
*   **Option 1:** You have to copy-paste the JSON into 50 different workflow files.
*   **Option 2:** You create one "Security Skill" in the DB and 50 workflows simply link to its ID. If you update the security policy once, all 50 agents are updated instantly.

#### 3. Hierarchical Management
Since you mentioned skills are hierarchical, a Database handles this naturally via a `parent_id` relationship. A single JSON file becomes incredibly "deep" and hard to parse if it tries to represent a multi-level hierarchy of text blocks.

---

### The Recommended Hybrid Implementation

To keep your `workflow.json` spec clean, don't store the *text* of the skill in the JSON. Store the **Reference**.

#### Step 1: Update the Schema (`$defs/agentDefinition`)
Modify your schema so that an Agent can "inherit" skills.

```json
"agentDefinition": {
  "type": "object",
  "properties": {
    "skills": {
      "type": "array",
      "items": {
        "oneOf": [
          { "type": "string", "description": "The ID or Name of the skill in the DB" },
          { 
            "type": "object", 
            "properties": {
              "id": { "type": "string" },
              "version": { "type": "string" }
            }
          }
        ]
      }
    }
  }
}
```

#### Step 2: The Java Loading Logic
When your Java Generic Agent starts up:
1.  It reads the `workflow.json`.
2.  It sees the list of skill IDs: `["legal-compliance", "customer-support-tone", "product-knowledge-v2"]`.
3.  The Java engine queries the Database: `SELECT content FROM skills WHERE id IN (...)`.
4.  The Java engine **concatenates** the Markdown files into a single System Prompt.

#### Step 3: Managing the Hierarchy in Java
Since skills are hierarchical, your Java code should perform a **Recursive Fetch**:
*   If the workflow asks for "Product Knowledge," your Java code checks the DB and sees "Product Knowledge" has a parent "Base Research Skill."
*   The engine fetches **both** and stacks them:
    1.  *# Base Research Rules* (Parent)
    2.  *## Product Specific Knowledge* (Child)

---

### How this looks in the Database

| Skill ID | Parent ID | Content (Markdown) |
| :--- | :--- | :--- |
| `enterprise-base` | NULL | `# Core Rules \n - No PII \n - Be polite` |
| `tech-support` | `enterprise-base` | `## Tech Support \n - Use code blocks \n - Ask for logs` |
| `java-specialist` | `tech-support` | `### Java Skill \n - Focus on JDK 21 \n - Check pom.xml` |

### Summary of Benefits
1.  **Markdown for the LLM:** The LLM gets a beautifully structured Markdown document (which is what it wants).
2.  **Database for the Admin:** Your Java backend provides a UI for experts to edit Markdown skills.
3.  **JSON for the Orchestrator:** Your `workflow.json` remains a slim, high-level "map" that just points to the right skills.

**Final Tip:** In your Java code, implement a **Cache**. You don't want to query the DB for the same Markdown text 1,000 times a minute. Load the skills into a `HashMap<String, String>` (Cache) when the first agent call comes in.

