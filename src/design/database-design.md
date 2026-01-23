# Database Design

To integrate the **Agentic Workflow** into the existing database, we need to separate **Design-Time** tables (the blueprints) from your existing **Run-Time** tables (the instances).

### 1. The Design-Time Tables (The Blueprints)

These tables store the "What" and "How" of your agents and workflows.

```sql
-- 1. Workflow Definitions: Stores the Agentic Workflow JSON
CREATE TABLE wf_definition_t (
    wf_def_id           UUID NOT NULL,
    namespace           VARCHAR(126) NOT NULL,
    name                VARCHAR(126) NOT NULL,
    version             VARCHAR(20) NOT NULL,
    definition_json     TEXT NOT NULL, -- The Agentic Workflow DSL
    active              BOOLEAN DEFAULT TRUE,
    update_ts           TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    update_user         VARCHAR(126) DEFAULT SESSION_USER,
    PRIMARY KEY(wf_def_id),
    UNIQUE(namespace, name, version)
);

-- 2. Agent Definitions: Stores the "Brain" configuration
CREATE TABLE agent_definition_t (
    agent_def_id        UUID NOT NULL,
    agent_name          VARCHAR(126) NOT NULL,
    model_provider      VARCHAR(64) NOT NULL,  -- 'openai', 'anthropic', etc.
    model_name          VARCHAR(126) NOT NULL, -- 'gpt-4o', 'claude-3-5-sonnet'
    api_key_ref         VARCHAR(126),          -- Reference to Secret Manager key
    temperature         NUMERIC(3,2) DEFAULT 0.7,
    max_tokens          INTEGER,               -- max number of tokens can be used
    PRIMARY KEY(agent_def_id),
    UNIQUE(agent_name)
);

-- 3. Agent Skills: Stores Hierarchical Markdown
CREATE TABLE agent_skill_t (
    skill_id            UUID NOT NULL,
    parent_skill_id     UUID,                  -- Self-reference for Hierarchy
    name                VARCHAR(126) NOT NULL,
    description         VARCHAR(500),
    content_markdown    TEXT NOT NULL,         -- The actual skill instructions
    version             VARCHAR(20) DEFAULT '1.0.0',
    PRIMARY KEY(skill_id),
    FOREIGN KEY(parent_skill_id) REFERENCES agent_skill_t(skill_id)
);

-- 4. Agent-Skill Mapping: Links Agents to their Skills
CREATE TABLE agent_skill_map_t (
    agent_def_id        UUID NOT NULL,
    skill_id            UUID NOT NULL,
    sequence_id         INTEGER DEFAULT 0,     -- Order in which skills are concatenated
    PRIMARY KEY(agent_def_id, skill_id),
    FOREIGN KEY(agent_def_id) REFERENCES agent_definition_t(agent_def_id) ON DELETE CASCADE,
    FOREIGN KEY(skill_id) REFERENCES agent_skill_t(skill_id) ON DELETE CASCADE
);
```

### 2. The Run-Time Extensions (The Instances)

To support Agent-as-a-Service, we need to track the **LLM Conversation History** (Memory) and link your existing processes to specific workflow versions.

```sql
-- 5. Agent Memory: Stores the conversation thread for a specific process/task
-- This is critical for maintaining context in the cloud.
CREATE TABLE agent_session_history_t (
    session_history_id  UUID NOT NULL,
    process_id          UUID NOT NULL,         -- Link to your process_info_t
    task_id             UUID,                  -- Link to task_info_t (optional)
    role                VARCHAR(20) NOT NULL,  -- 'system', 'user', 'assistant', 'tool'
    content             TEXT NOT NULL,         -- The message text
    tool_call_id        VARCHAR(126),          -- If the message is a tool call
    created_ts          TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY(session_history_id),
    FOREIGN KEY(process_id) REFERENCES process_info_t(process_id) ON DELETE CASCADE
);

-- Index for fast retrieval of chat history in chronological order
CREATE INDEX agent_session_idx1 ON agent_session_history_t (process_id, created_ts);
```

### 3. Modifications to your Existing Tables

You should link your existing `process_info_t` to the definition so the engine knows which logic to execute.

```sql
-- Link the Process Instance to the Blueprint version
ALTER TABLE process_info_t 
ADD COLUMN wf_def_id UUID REFERENCES wf_definition_t(wf_def_id);

-- Optional: Track which Agent is currently working on a specific Task
ALTER TABLE task_info_t 
ADD COLUMN agent_def_id UUID REFERENCES agent_definition_t(agent_def_id);
```

### How the Java Generic Agent uses this:

1.  **Workflow Start:** When a process starts, it writes to `process_info_t` and stores the `wf_def_id`.
2.  **Skill Loading:** The Java Engine sees that `Task A` requires `Agent X`. It queries:
    ```sql
    -- Fetch the merged skill hierarchy for the agent
    SELECT content_markdown FROM agent_skill_t s
    JOIN agent_skill_map_t m ON s.skill_id = m.skill_id
    WHERE m.agent_def_id = ?
    ORDER BY m.sequence_id ASC;
    ```
3.  **Memory Management:** Before calling the LLM, the Java Engine fetches previous turns:
    ```sql
    SELECT role, content FROM agent_session_history_t 
    WHERE process_id = ? 
    ORDER BY created_ts ASC;
    ```
4.  **Audit:** Every step is already tracked by your `audit_log_t`, and every assignment by your `task_asst_t`.

### Why this structure works for Enterprise:
*   **Version Control:** `wf_definition_t` allows you to have "V1" and "V2" of a workflow running simultaneously.
*   **Hierarchy:** `agent_skill_t.parent_skill_id` allows you to build a tree of knowledge that the Java code can traverse.
*   **Auditability:** By linking `agent_session_history_t` to `process_id`, you can reconstruct exactly what the AI said and why it made a certain decision during a business process.
*   **Security:** `api_key_ref` keeps your keys out of the database (pointing instead to your Vault).

