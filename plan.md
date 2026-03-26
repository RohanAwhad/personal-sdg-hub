# Fresh SDG Repo Plan

## Goal

Build a fresh SDG repo that keeps the real value from the old project:

- all `flow.yaml` assets
- all prompt YAML assets
- the behavior those assets rely on

Everything else gets rebuilt from scratch.

## Hard Decisions

- Keep `pandas` as the core data container.
- Drop `datasets` from the runtime core.
- Reimplement every block from scratch.
- Do not depend on LiteLLM.
- Do not recreate the old class hierarchy or helper maze.
- Do not use the old repo as architecture inspiration.
- Use the old repo only to answer narrow semantic questions about a block.

## Product Thesis

- The gold is in `flows/` and `prompts/`.
- The engine around them should be tiny.
- Compatibility should be asset-level, not framework-level.
- Blocks should be short, obvious, and boring.
- A 100-line function is fine.
- A pile of indirection is not.

## Design Principles

- Functional core.
- Explicit I/O.
- Minimal abstraction.
- No import-side-effect magic.
- No runtime mutation of loaded flow specs.
- Fail fast.
- Prefer direct code over reusable cleverness.
- Write to a small, boring subset of the pandas API.

## V1 Scope

V1 should support:

- loading all copied flows
- loading all prompt YAMLs
- sequential flow execution
- runtime block overrides by block name
- normalized chat client protocol
- non-agent flows first

V1 should not support:

- FastAPI UI
- MCP runtime
- agent runtime
- checkpointing
- Rich display / metrics dashboards
- source YAML rewriting
- `datasets.Dataset` as a first-class runtime type

## Core Architectural Shape

The new system should be built from five small pieces:

1. flow loader
2. block registry
3. sequential executor
4. chat protocol + adapters
5. block functions

That is the whole engine.

## Repo Layout

```text
new_sdg/
  plan.md
  devlogs.md
  pyproject.toml
  README.md
  src/sdg/
    __init__.py
    flows/
      ... copied asset tree ...
    core/
      types.py
      loader.py
      registry.py
      executor.py
      validation.py
    chat/
      types.py
      protocol.py
      openai_compatible.py
      fake.py
    blocks/
      llm.py
      parsing.py
      transform.py
      filtering.py
      agent.py
    logging.py
  tests/
    test_loader.py
    test_executor.py
    test_chat_protocol.py
    test_blocks_llm.py
    test_blocks_parsing.py
    test_blocks_transform.py
    test_blocks_filtering.py
    flows/
```

## Data Model

### FlowSpec

```python
@dataclass(frozen=True)
class FlowSpec:
    metadata: FlowMetadata
    blocks: list[BlockSpec]
    source_path: Path
```

### FlowMetadata

```python
@dataclass(frozen=True)
class FlowMetadata:
    name: str
    id: str | None
    description: str
    version: str
    author: str
    license: str
    tags: list[str]
    recommended_models: RecommendedModels | None
    dataset_requirements: DatasetRequirements | None
    extra: dict[str, Any]
```

Notes:

- preserve unknown metadata fields in `extra`
- never rewrite source YAML to inject ids
- if `id` is missing, derive it in memory only

### BlockSpec

```python
@dataclass(frozen=True)
class BlockSpec:
    block_type: str
    block_name: str
    input_cols: str | list[str] | dict[str, Any] | None
    output_cols: str | list[str] | dict[str, Any] | None
    params: dict[str, Any]
```

### ExecutionContext

```python
@dataclass(frozen=True)
class ExecutionContext:
    flow_path: Path
    chat_client: ChatClient | None
    overrides: dict[str, dict[str, Any]]
    logger: Any
```

## Runtime Contract

We should not build a `BaseBlock` hierarchy.

The runtime contract should be plain functions.

```python
BlockRunner = Callable[[pd.DataFrame, BlockSpec, ExecutionContext], pd.DataFrame]
```

Registry:

```python
BLOCK_RUNNERS: dict[str, BlockRunner]
```

Execution:

```python
def run_flow(
    flow: FlowSpec,
    df: pd.DataFrame,
    chat_client: ChatClient | None = None,
    overrides: dict[str, dict[str, Any]] | None = None,
) -> pd.DataFrame:
    ...
```

Override merge:

```python
effective_params = {
    **block.params,
    **overrides.get(block.block_name, {}),
}
```

No mutation of:

- `FlowSpec`
- `BlockSpec`
- original block config dicts

## Pandas Policy

Keep the runtime pandas-only.

But use a small subset of pandas so the code stays portable and easy to replace later.

Allowed / encouraged operations:

- `df.copy()`
- column reads and writes
- `df.rename(...)`
- `df.melt(...)`
- `df.to_dict("records")`
- `pd.DataFrame(records)`
- boolean masks
- `pd.concat(...)`
- simple `Series.apply(...)`

Avoid unless clearly necessary:

- complex indexing tricks
- MultiIndex
- dtype cleverness
- custom extension arrays
- pandas-heavy abstraction wrappers

## Chat Interface

This is the main external boundary.

### ChatMessage

```python
@dataclass(frozen=True)
class ChatMessage:
    role: Literal["system", "user", "assistant", "tool"]
    content: str
```

### ChatRequest

```python
@dataclass(frozen=True)
class ChatRequest:
    messages: list[ChatMessage]
    model: str
    n: int = 1
    temperature: float | None = None
    max_tokens: int | None = None
    extra: dict[str, Any] = field(default_factory=dict)
```

### ChatResponse

```python
@dataclass(frozen=True)
class ChatResponse:
    content: str | None
    reasoning_content: str | None = None
    tool_calls: list[dict[str, Any]] | None = None
    raw: Any = None
```

### ChatClient Protocol

```python
class ChatClient(Protocol):
    def complete(self, request: ChatRequest) -> list[ChatResponse]:
        ...
```

Rules:

- every adapter must conform to this protocol
- blocks depend on `ChatClient`, not provider SDKs
- provider-specific normalization happens inside adapters
- the rest of the repo should never care where responses came from

## First Adapter

First real adapter should be `OpenAICompatibleChatClient` using `httpx`.

Why:

- small API surface
- covers many hosted backends
- works for local vLLM-style servers
- no LiteLLM dependency

Possible future adapters:

- `AnthropicChatClient`
- `GeminiChatClient`
- optional extras packages later

## Loader Design

Responsibilities:

- read `flow.yaml`
- validate root structure
- parse metadata
- parse ordered block list
- preserve relative prompt paths
- keep unknown metadata fields
- never modify source files

Key behavior to preserve:

- block order is the program
- prompt paths are resolved relative to the flow file
- block names in YAML are the public compatibility surface

## Block Implementation Rules

This part matters most.

Each block should be implemented as:

- one top-level runner function
- maybe one or two local helpers if they remove obvious repetition
- direct, readable pandas code
- no inheritance
- no decorator magic beyond registry setup if needed

Strong preference:

- 50-150 line functions are okay
- fewer files is okay
- local repetition is okay if it keeps behavior obvious

Anti-patterns to avoid:

- helper chains across many files
- a framework for block execution
- abstract base classes for everything
- “smart” shared mixins
- magic config mutation

## Old Repo Usage Policy

We should not port the old structure.

Use the old repo only to answer questions like:

- what columns does this block require?
- what columns does it produce?
- does it expand rows or keep row count stable?
- what happens on empty lists / invalid values?
- what shape does this flow expect at the next block boundary?

For each block, we should extract a tiny note and stop reading.

## Migration Order

### Phase 0: Bootstrap

- create repo skeleton
- add `devlogs.md`
- add `pyproject.toml`
- copy `flows/` tree unchanged

Deliverable:

- fresh repo with assets present

### Phase 1: Core Kernel

- implement dataclasses and protocol types
- implement loader
- implement registry
- implement executor
- implement dataset requirement validation

Deliverable:

- all copied flows load successfully

### Phase 2: Wave 1 Blocks

Implement the core set first:

- `PromptBuilderBlock`
- `LLMChatBlock`
- `LLMResponseExtractorBlock`
- `TagParserBlock`
- `DuplicateColumnsBlock`
- `RenameColumnsBlock`
- `ColumnValueFilterBlock`
- `RegexParserBlock`
- `MeltColumnsBlock`

Deliverable:

- knowledge and evaluation flows run with fake chat client

### Phase 3: First Real Chat Adapter

- implement `OpenAICompatibleChatClient`
- add adapter contract tests
- run representative flows against a real endpoint

Deliverable:

- end-to-end flow execution without LiteLLM

### Phase 4: Wave 2 Blocks

- `SamplerBlock`
- `RowMultiplierBlock`
- `JSONParserBlock`
- `JSONStructureBlock`

Deliverable:

- red-team and text-analysis flows run

### Phase 5: Optional Layer

- minimal CLI if needed
- loguru logging cleanup
- optional concurrency tuning
- optional docs for custom adapters and custom blocks

### Phase 6: Separate Plugins

- `AgentBlock`
- `AgentResponseExtractorBlock`
- MCP-specific pieces

Deliverable:

- advanced features isolated away from the core engine

## Flow Priority

Start with representative high-value flows:

1. `knowledge_infusion/enhanced_multi_summary_qa/detailed_summary`
2. `knowledge_infusion/enhanced_multi_summary_qa/key_facts`
3. `evaluation/rag_evaluation`
4. `red_team/prompt_generation`
5. `text_analysis/structured_insights`

Leave `agentic/mcp_distillation` for later.

## Testing Strategy

### Unit Tests

For each block:

- happy path
- missing input columns
- output columns created correctly
- row-count behavior
- empty input behavior where relevant
- edge cases for lists / parsing / filtering

### Loader Tests

- all copied flows parse
- prompt paths resolve
- metadata extras are preserved
- source YAMLs are never modified

### Chat Contract Tests

- `FakeChatClient` satisfies protocol
- every real adapter satisfies protocol
- adapter outputs normalize into `ChatResponse`

### Flow Regression Tests

Run representative flows with `FakeChatClient` and assert:

- output columns
- row counts
- parser behavior
- filter behavior
- `n > 1` expansion behavior

## Logging Plan

Keep logging small.

- use `loguru`
- read level from `LOGGING_LEVEL`
- write to `logs/`
- no Rich dependency
- no provider-specific logging hacks in core

Only log:

- flow start / finish
- block start / finish
- adapter request metadata without secrets
- major failures with block names and flow names

## Non-Goals

- perfect abstraction
- framework-like extensibility on day one
- provider-agnostic everything from the start
- high-performance optimization before parity
- reproducing every old repo feature before core flows work

## Biggest Risks

1. hidden row-expansion semantics in parser/extractor blocks
2. response shape differences across chat adapters
3. overengineering the new runtime too early
4. accidentally rebuilding the old abstraction stack in a new folder

## Risk Mitigation

- keep a block mini-spec for each migrated block
- implement blocks directly from the mini-spec
- use fake chat client heavily in tests
- prioritize one representative flow at a time
- reject abstractions unless they remove obvious repeated logic

## Success Criteria

We are done with the first solid version when:

- all copied flows load cleanly
- key non-agent flows run on the new runtime
- the repo has no LiteLLM dependency in core
- the engine is understandable by reading a handful of files
- each block implementation is short and direct
- adding a new adapter only requires conforming to `ChatClient`

## Final Recommendation

Do not refactor the old repo into shape.

Build a new repo with:

- copied assets
- a tiny pandas-based execution kernel
- dataclasses for control-plane types
- plain functions for blocks
- one strict chat protocol boundary
- no abstraction games
