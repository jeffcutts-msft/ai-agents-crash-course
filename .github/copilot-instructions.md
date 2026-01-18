# AI Agents Crash Course - Copilot Instructions

## Project Overview
This is a Udemy course codebase teaching AI agent development with OpenAI's Agents SDK. The project builds a **nutrition assistant chatbot** that demonstrates agent patterns from basic to production-ready, including tool calling, RAG, multi-agent orchestration, guardrails, and authentication.

## Architecture & Structure

### Key Components
- **`chatbot/`** - Starter templates for students (incomplete)
- **`chatbot_complete/`** - Reference implementations progressing from simple to production-ready:
  - `1_simple_chatbot.py` - Basic Chainlit integration
  - `2_agentic_chatbot.py` - Tool calling with streaming
  - `3_memory.py` - Conversation persistence with SQLiteSession
  - `4_authentication.py` - Password-based auth
- **`multi_agent_chatbot/`** - Production system with multi-agent handoffs, MCP integration, and guardrails
- **`notebooks/`** & **`notebooks_complete/`** - Jupyter tutorials (student vs reference)
- **`chatkit/`** - Next.js frontend using OpenAI ChatKit web component for Agent Builder workflows
- **`data/`** - Calorie database for RAG (`calories.csv`, `calorie_database.txt`)
- **`chroma/`** - ChromaDB persistent storage for nutrition embeddings

### Agent Patterns
All agents use the **OpenAI Agents SDK** (not LangChain):
```python
from agents import Agent, Runner, function_tool

nutrition_agent = Agent(
    name="Nutrition Assistant",
    instructions="...",  # System prompt
    tools=[calorie_lookup_tool],
    mcp_servers=[exa_search_mcp],  # Model Context Protocol
    input_guardrails=[food_topic_guardrail]
)

# Execute with streaming
result = Runner.run_streamed(nutrition_agent, user_input, session=session)
```

### Multi-Agent Architecture
See [`multi_agent_chatbot/nutrition_agent.py`](multi_agent_chatbot/nutrition_agent.py) for complete example:
1. **Agent-as-tool pattern**: Convert agents to tools with `agent.as_tool()`
2. **Handoffs**: Use `handoffs=[agent]` for sequential delegation
3. **Guardrails**: Async `@input_guardrail` decorators validate inputs before execution

## Development Workflows

### Environment Setup
```bash
# Create .env from template (required for all operations)
cp .env.template .env
# Set: OPENAI_API_KEY, OPENAI_DEFAULT_MODEL, EXA_API_KEY, CHAINLIT_AUTH_SECRET
```

### Running Chatbots
```bash
# Development (auto-reload)
chainlit run chatbot_complete/4_authentication.py --port 10000 --host 0.0.0.0

# Production deployment (from COURSE_RESOURCES.md)
chainlit run chatbot/4_authentication.py --port 10000 --host 0.0.0.0
```

### Working with Notebooks
- Notebooks use **async/await** patterns - cells must be run sequentially
- Start with [`notebooks_complete/simplest_agent.ipynb`](notebooks_complete/simplest_agent.ipynb) to verify setup
- RAG setup: Run [`rag_setup/rag_setup.ipynb`](rag_setup/rag_setup.ipynb) to create ChromaDB embeddings

### ChatKit Frontend
```bash
cd chatkit
npm install
# Configure .env.local with OPENAI_API_KEY and NEXT_PUBLIC_CHATKIT_WORKFLOW_ID from Agent Builder
npm run dev  # Runs on localhost:3000
```

## Project-Specific Conventions

### Chainlit Integration
- Use `@cl.on_chat_start` to initialize sessions: `cl.user_session.set("agent_session", SQLiteSession(...))`
- Stream responses with `msg.stream_token()` on `ResponseTextDeltaEvent`
- Visualize tool calls with `cl.Step(name=tool_name, type="tool")`
- Authentication via `@cl.password_auth_callback` (username/password from `.env`)

### Memory & Sessions
- **Short-term memory**: Use `SQLiteSession("conversation_history")` for stateful conversations
- Pass session to runner: `Runner.run_streamed(agent, input, session=session)`
- Sessions persist across messages within a Chainlit user session

### RAG Implementation
- ChromaDB at [`chroma/`](chroma/) with persistent client:
  ```python
  chroma_client = chromadb.PersistentClient(path=str(chroma_path))
  nutrition_db = chroma_client.get_collection(name="nutrition_db")
  ```
- Tool pattern: `@function_tool` decorator with docstrings for agent context
- Query with metadata: `results["metadatas"][0][i]["food_item"]`

### MCP (Model Context Protocol)
- Connect external tools (Exa Search): `MCPServerStreamableHttp` with 90s timeout
- Connect in `@cl.on_chat_start`: `await exa_search_mcp.connect()`
- Add to agents: `mcp_servers=[exa_search_mcp]`

### Guardrails Pattern
```python
@input_guardrail
async def food_topic_guardrail(ctx, agent, input):
    result = await Runner.run(guardrail_agent, input, context=ctx.context)
    return GuardrailFunctionOutput(
        tripwire_triggered=(not result.final_output.only_about_food)
    )
```
- Use structured outputs (`output_type=BaseModel`) in guardrail agents
- Catch `InputGuardrailTripwireTriggered` exceptions to handle blocked inputs

## Common Pitfalls
- **Async execution**: Always use `await` with `Runner.run()` and `Runner.run_streamed()`
- **Event streaming**: Filter events by type - `event.type == "raw_response_event"` and check `isinstance(event.data, ResponseTextDeltaEvent)`
- **MCP timeout**: Exa Search can be slow under load - use 90s timeout instead of default 30s
- **Environment variables**: `dotenv.load_dotenv()` must be called before accessing `os.environ`
- **ChromaDB paths**: Use absolute paths via `Path(__file__).parent.parent / "chroma"`

## Key Files Reference
- Agent definitions: [`chatbot_complete/nutrition_agent.py`](chatbot_complete/nutrition_agent.py), [`multi_agent_chatbot/nutrition_agent.py`](multi_agent_chatbot/nutrition_agent.py)
- Chainlit entry points: [`chatbot_complete/2_agentic_chatbot.py`](chatbot_complete/2_agentic_chatbot.py), [`multi_agent_chatbot/agentic_chatbot.py`](multi_agent_chatbot/agentic_chatbot.py)
- RAG setup: [`rag_setup/rag_setup.ipynb`](rag_setup/rag_setup.ipynb)
- Course reference: [`COURSE_RESOURCES.md`](COURSE_RESOURCES.md)
