# AI Tasks - GitHub Copilot Instructions

## Project Overview
AI Tasks is a Liferay-based platform for building AI-assisted, agentic workflows without extensive programming. It integrates LangChain4J for LLM orchestration and supports Model Context Protocol (MCP) for standardized service integrations. The system exposes visual workflows as REST endpoints, leveraging Liferay's permission/auditing capabilities.

## Architecture

### Core Module Structure (Liferay Workspace Pattern)
- **`ai-tasks-api`**: Service builder generated API interfaces
- **`ai-tasks-service`**: Business logic, persistence, and LangChain4J integrations (service.xml)
- **`ai-tasks-spi`**: Extension point interfaces (`AITaskNode`, `AITaskTool`, `AITaskContextContributor`)
- **`ai-tasks-rest-api`/`rest-impl`**: REST endpoints generated from OpenAPI spec (`rest-openapi.yaml`)
- **`ai-tasks-rest-client`**: Auto-generated REST client from OpenAPI
- **Client Extensions**: React-based admin UI (`ai-tasks-admin-web`), chat widgets

### Key Data Flow
1. REST request → `AITaskResponseResourceImpl.generate()` (external reference code based routing)
2. Task configuration (JSON) → Node graph execution
3. Node types implement `AITaskNode.execute()` returning `AITaskNodeResponse`
4. Chat nodes use LangChain4J's `AiServices` pattern with memory/tools
5. Response includes execution trace, token usage, timing

### Node Type Architecture
- **Base classes**: `BaseChatModelAITaskNode`, `BaseStreamingChatModelAITaskNode`
- **Implementations**: Provider-specific nodes (OpenAI, Ollama, Gemini, Anthropic, etc.) registered via `@Component(property = "ai.task.node.type=<type>")`
- **Node execution**: Configured via JSON parameters, supports prompt templates with `{{input.text}}` syntax
- **Memory**: Optional chat memory via LangChain4J's `MessageWindowChatMemory` (configurable max messages)

## Development Workflows

### Building & Deployment
```bash
# Initialize Liferay bundle (first time)
cd liferay-workspace && gw initBundle

# Deploy all modules
cd modules && gw deploy

# Deploy client extensions
cd client-extensions && gw deploy

# Start DXP locally
blade server run

# Docker-based development
gw createDockerContainer && gw startDockerContainer
```

### Service Builder Workflow
1. Edit `modules/ai-tasks-service/service.xml`
2. Run `gw buildService` - generates API/implementation stubs
3. Implement service methods in `*LocalServiceImpl.java`
4. Run `gw deploy` - deploys both API and service JARs

### REST API Development
1. Edit `modules/ai-tasks-rest-impl/rest-openapi.yaml`
2. Run `gw buildREST` - generates resource interfaces/implementations
3. Implement endpoints in `*ResourceImpl.java` extending `Base*ResourceImpl`
4. Configuration lives in `rest-config.yaml`

### Client Extension Development (React)
- Located in `client-extensions/ai-tasks-admin-web/`
- Uses React 18, @xyflow/react for visual workflow editor
- Build: `yarn build` → outputs to `build/static/`
- Configuration: `client-extension.yaml` defines custom element registration
- Dev mode: `yarn start` (connects to DXP via `client-extension.dev.yaml`)

## Project-Specific Conventions

### Environment Variables
Prefix sensitive values with `env:` in task configurations:
```json
{"apiKey": "env:OPENAI_API_KEY"}
```

### External Reference Codes
Primary task identifier for REST endpoints:
```
POST /o/ai-tasks/v1.0/generate/{externalReferenceCode}
```
Must be unique per company, set via `externalReferenceCode` field.

### Task Configuration Schema
- Version: `schemaVersion: "1.0"`
- Structure: `configuration.nodes[]` (nodes) + `configuration.edges[]` (connections)
- Node IDs: Auto-generated (e.g., `blushCoolCoyote`) via `unique-names-generator`
- Parameters: Node-specific, validated in backend implementations

### Debugging/Logging
- Set `trace: true` in task configuration for execution details in response
- Set `logRequests: true`/`logResponses: true` on individual nodes for LLM traffic logs
- Use `debug: true` for debug output in HTTP responses

### Extension Points (SPI Module)
```java
// Add custom node types
@Component(property = "ai.task.node.type=myCustomNode", service = AITaskNode.class)
public class MyCustomNode implements AITaskNode {
  AITaskNodeResponse execute(AITaskContext ctx, JSONObject config, String nodeId, boolean trace)
}

// Add context contributors (inject data into AI context)
@Component(service = AITaskContextContributor.class)
public class MyContextContributor implements AITaskContextContributor {
  void contribute(AITaskContext ctx, HttpServletRequest req, Locale locale, User user)
}
```

### LangChain4J Integration Patterns
- **Chat models**: Built via provider-specific builders (e.g., `OpenAiChatModel.builder()`)
- **AI Services**: Created in `AIServiceHelperImpl.createAssistant()` using `AiServices.builder(ChatAssistant.class)`
- **Tool providers**: MCP configured via `toolProvider.mcp.clients[]` with stdio/HTTP transports
- **Memory**: `MessageWindowChatMemory` with configurable window size (`memoryMaxMessages`)

## Critical Dependencies
- Liferay DXP 7.4 (U128+) required for semantic search providers
- Java 17
- LangChain4J 1.0.0-beta2 (all provider modules included)
- Node.js/Yarn for client extensions
- Gradle wrapper (`gw` or `./gradlew`)

## Common Patterns

### Adding a New LLM Provider
1. Create node class in `modules/ai-tasks-service/src/.../internal/task/node/type/<provider>/`
2. Extend `BaseChatModelAITaskNode` or `BaseStreamingChatModelAITaskNode`
3. Implement `getChatLanguageModel()` with provider-specific builder
4. Register via `@Component(property = "ai.task.node.type=<type>", service = AITaskNode.class)`
5. Add frontend constants in `client-extensions/ai-tasks-admin-web/src/constants/AITasksNodeTypesConstants.js`
6. Add default parameters in `utils/nodeUtils.js`

### Sample Task Structure
See `samples/*/` for examples. Key elements:
- Input trigger node (entry point)
- Chat/image model nodes (LLM operations)
- Webhook nodes (external integrations)
- Condition nodes (workflow branching)
- Edges define execution flow

## Testing
- Local testing: AI Tasks Admin preview chat or Liferay API explorer (`/o/api`)
- Integration tests: `modules/ai-tasks-rest-test/` using Liferay test framework
- Client extension: Test chatbot widget via fragment on portal page

## Known Limitations
- LLM calls are blocking (performance issue tracked in GitHub #2)
- LangChain4J AI Services lack multimodal input support (upstream issue)
- Vertex AI auth requires gcloud CLI setup before DXP startup
