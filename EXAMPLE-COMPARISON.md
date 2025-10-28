# Example: DevUI README Comparison

This document shows a concrete example of differences between English and Chinese documentation files.

## File Comparison: devui/README.zh-tw.md

### Current Status
- **English source:** `python/samples/getting_started/devui/README.md` (160 lines)
- **Chinese translation:** `python/samples/getting_started/devui/README.zh-tw.md` (119 lines)
- **Gap:** 41 lines missing (25% of content)

### What's Missing in Chinese Version?

#### 1. Workflow Examples Table (Lines 62-67 in English)

**English has 3 workflows:**
```markdown
| Sample                                       | Description                                                       | Features                                                                                                                    | Required Environment Variables                                                        |
| -------------------------------------------- | ----------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| [**workflow_agents/**](workflow_agents/)     | Content review workflow with agents as executors                  | Agents as workflow nodes, conditional routing based on structured outputs, quality-based paths (Writer → Reviewer → Editor/Publisher) | `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_CHAT_DEPLOYMENT_NAME`, `AZURE_OPENAI_ENDPOINT` |
| [**spam_workflow/**](spam_workflow/)         | 5-step email spam detection workflow with branching logic         | Sequential execution, conditional branching (spam vs. legitimate), multiple executors, mock spam detection                  | None - uses mock data                                                                 |
| [**fanout_workflow/**](fanout_workflow/)     | Advanced data processing workflow with parallel execution         | Fan-out/fan-in patterns, complex state management, multi-stage processing (validation → transformation → quality assurance) | None - uses mock data                                                                 |
```

**Chinese currently has only 1 workflow:**
```markdown
| 範例 | 描述 | 功能 | 所需環境變數 |
| ---------------------------------------------------- | ---------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| [**basic_workflow/**](basic_workflow/) | 使用 Azure OpenAI 的簡單寫作者-編輯者工作流程 | 工作流程編排、代理鏈接、Azure OpenAI 整合 | `AZURE_OPENAI_ENDPOINT`、`AZURE_OPENAI_CHAT_DEPLOYMENT_NAME` |
```

**Action needed:** Add the 2 missing workflow examples: spam_workflow and fanout_workflow.

---

#### 2. API Usage Section (Lines 119-137 in English)

**English includes API examples:**
```markdown
## API Usage

DevUI exposes OpenAI-compatible endpoints:

\`\`\`bash
curl -X POST http://localhost:8080/v1/responses \
  -H "Content-Type: application/json" \
  -d '{
    "model": "agent-framework",
    "input": "What is the weather in Seattle?",
    "extra_body": {"entity_id": "agent_directory_weather-agent_<uuid>"}
  }'
\`\`\`

List available entities:

\`\`\`bash
curl http://localhost:8080/v1/entities
\`\`\`
```

**Chinese version:** This entire section is missing.

**Action needed:** Translate and add the API Usage section with both curl examples.

---

#### 3. Agent Examples Table (Line 60 in English)

**English has 3 agents:**
- weather_agent_azure
- foundry_agent  
- **openai_agent** ← Missing in Chinese

**Chinese has 3 agents:**
- weather_agent_azure
- foundry_agent
- openai_agent ← Actually present!

**Status:** ✅ This is up to date after checking

---

#### 4. "Using DevUI with Your Own Agents" Section (Lines 90-117)

**English includes detailed example:**
```markdown
## Using DevUI with Your Own Agents

To make your agent discoverable by DevUI:

1. Create a folder for your agent
2. Add an `__init__.py` that exports `agent` or `workflow`
3. (Optional) Add a `.env` file for environment variables

Example:

\`\`\`python
# my_agent/__init__.py
from agent_framework import ChatAgent
from agent_framework.openai import OpenAIChatClient

agent = ChatAgent(
    name="MyAgent",
    description="My custom agent",
    chat_client=OpenAIChatClient(),
    # ... your configuration
)
\`\`\`

Then run:

\`\`\`bash
devui /path/to/my/agents/folder
\`\`\`
```

**Chinese has a condensed version** but missing the Python code example.

**Action needed:** Add the Python code example for creating custom agents.

---

#### 5. Learn More Links (Lines 139-143)

**English has:**
```markdown
## Learn More

- [DevUI Documentation](../../../packages/devui/README.md)
- [Agent Framework Documentation](https://docs.microsoft.com/agent-framework)
- [Sample Guidelines](../../SAMPLE_GUIDELINES.md)
```

**Chinese has:**
```markdown
## 參考資料

- [Agent Framework 文件](../../../README.md)
- [DevUI 套件文件](../../../../packages/devui/README.md)
```

**Difference:** Missing the "Sample Guidelines" link and has different link for main documentation.

**Action needed:** Update links to match English version.

---

## Summary of Required Changes

1. ✅ Add 2 missing workflows to the table
2. ⚠️ Add entire "API Usage" section (high priority)
3. ✅ Agent table is already complete
4. ⚠️ Add Python code example for custom agents
5. ✅ Update "Learn More" links

**Estimated time:** 2 hours

## How This Was Discovered

Using the analysis script `/tmp/advanced_doc_check.py`, we compared:
- Heading counts
- Code block counts
- Link counts
- Line counts
- Content structure

This revealed:
```
English: 17 headings, 10 code blocks, 9 links, 160 lines
Chinese: 16 headings, 5 code blocks, 6 links, 119 lines
```

The difference of 5 code blocks and 3 links immediately flagged this file for manual review.

---

*This example demonstrates the type of analysis performed on all 24 Traditional Chinese documentation files.*
