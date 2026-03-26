# Block Mini Specs

This file is the behavioral shorthand for the new repo.

Use the old `sdg_hub` repo only to answer narrow questions. Do not port its structure.

## Global Rules

- Every block runner is a plain function.
- Signature:

```python
def run_block(df: pd.DataFrame, block: BlockSpec, ctx: ExecutionContext) -> pd.DataFrame:
    ...
```

- `input_cols` and `output_cols` should be normalized inside the runner to the shape the block expects.
- Blocks return a new `DataFrame`.
- Blocks should fail loudly on missing required input columns.
- If a block creates new output columns, those target columns should normally not already exist.
- Prefer direct pandas code over helper layers.
- If a behavior is weird but flows depend on it, preserve the flow-facing behavior.

## Row Count Categories

- `same`: row count unchanged
- `expand`: one input row can become many output rows
- `filter`: row count can shrink
- `reshape`: row count changes due to wide/long transform

## Wave 1 Blocks

### PromptBuilderBlock

- Purpose: render a YAML prompt template into chat messages.
- Inputs:
  - `input_cols`: `list[str]` or `dict[str, str]`
  - `output_cols`: exactly one column
  - `prompt_config_path`: path relative to the flow file
  - `format_as_messages`: default `true`
- Output shape:
  - default: `list[dict[str, str]]` with `role` and `content`
  - optional fallback: concatenated string if `format_as_messages=false`
- Row count: `same`
- Required prompt shape:
  - YAML list of message objects
  - each object has `role` and `content`
  - at least one `user` message
  - last message should be `user`
- Variable mapping:
  - if `input_cols` is a list, column names are template variable names
  - if `input_cols` is a dict, map `dataset_col -> template_var`
- Empty rendered messages should be skipped.
- If rendering fails for a row, fail or produce empty output only if we explicitly choose that behavior. Prefer fail-fast in the new repo.

### LLMChatBlock

- Purpose: call the configured chat client for each row of rendered messages.
- Inputs:
  - exactly one input column containing `list[dict]` messages
  - one output column
  - model params such as `model`, `temperature`, `max_tokens`, `n`
  - optional extra provider params passed via `extra`
- Output shape:
  - always write `list[dict]` to the output column
  - even when `n=1`, keep the value as a one-element list
- Response dict shape should be normalized to:

```python
{
    "content": str | None,
    "reasoning_content": str | None,
    "tool_calls": list[dict] | None,
    "raw": Any,
}
```

- Row count: `same`
- Validation:
  - input cell must be a non-empty list
  - each message must be a dict with `role` and `content`
- `async_mode` exists in old YAMLs. In v1 we can parse it but ignore it.
- `ctx.chat_client` is required.

### LLMResponseExtractorBlock

- Purpose: extract normalized fields from the `LLMChatBlock` output.
- Inputs:
  - exactly one input column
  - accepted cell shapes: `dict` or `list[dict]`
  - booleans:
    - `extract_content`
    - `extract_reasoning_content`
    - `extract_tool_calls`
  - `expand_lists`: default `true`
  - `field_prefix`: default `""`
- Output columns:
  - if `field_prefix` is empty, default prefix is `block_name + "_"`
  - generated names are:
    - `<prefix>content`
    - `<prefix>reasoning_content`
    - `<prefix>tool_calls`
- Row count:
  - single dict input -> `same`
  - list input + `expand_lists=true` -> `expand`
  - list input + `expand_lists=false` -> `same`, but extracted values are lists
- Missing extracted fields:
  - if requested field is missing in a response, skip that field for that response
  - if no requested fields are found for a response, that response is invalid
- Invalid non-dict items inside a list should be skipped.
- If an input row contains no valid responses, fail.

### TagParserBlock

- Purpose: extract tagged text spans into output columns.
- Inputs:
  - exactly one input column
  - `start_tags: list[str]`
  - `end_tags: list[str]`
  - `output_cols` count must match tag pair count
  - optional `parser_cleanup_tags`
- Extraction rules:
  - if `start_tag == ""` and `end_tag == ""`, return the whole trimmed text
  - otherwise extract all regex matches between start and end tags
  - cleanup tags are removed from extracted values after parsing
- Input cell shapes:
  - `str`
  - `list[str]`
- Row count:
  - string input with multiple matches -> `expand`
  - list input -> `same`, output columns contain lists
  - no matches -> input row is dropped
- Multi-column behavior:
  - values are zipped across output columns
  - tag pair ordering matters

### DuplicateColumnsBlock

- Purpose: duplicate existing columns into new columns.
- Inputs:
  - `input_cols` is a dict of `source_col -> target_col`
- Output:
  - copied columns added to the dataframe
- Row count: `same`
- Fail if any source column is missing.
- Fail if target behavior would be ambiguous; keep implementation explicit.

### RenameColumnsBlock

- Purpose: rename columns.
- Inputs:
  - `input_cols` is a dict of `old_col -> new_col`
- Output:
  - dataframe with renamed columns
- Row count: `same`
- Fail if any source column is missing.
- Fail if any target column already exists.
- Chained / circular rename behavior should not be supported.

### ColumnValueFilterBlock

- Purpose: keep rows whose first input column satisfies a comparison.
- Inputs:
  - first input column is the column to test
  - `filter_value`: scalar or list
  - `operation`: one of
    - `eq`, `ne`, `lt`, `le`, `gt`, `ge`, `contains`, `in`
  - optional `convert_dtype`: `int` or `float`
- Row count: `filter`
- Behavior:
  - if `convert_dtype` is set, conversion happens before filtering
  - conversion failures become null-like and are filtered out
  - if `filter_value` is scalar, normalize it to a list internally
  - keep row if any filter value matches

### RegexParserBlock

- Purpose: parse text using a regex with capture groups.
- Inputs:
  - exactly one input column
  - `parsing_pattern`
  - optional `parser_cleanup_tags`
- Output behavior:
  - one capture group -> first output column
  - tuple of capture groups -> map in order across `output_cols`
- Input cell shapes:
  - `str`
  - `list[str]`
- Row count:
  - string input with multiple regex matches -> `expand`
  - list input -> `same`, output columns contain lists
  - no matches -> input row is dropped

### MeltColumnsBlock

- Purpose: convert selected columns from wide to long form.
- Inputs:
  - `input_cols`: columns to melt
  - `output_cols`: exactly two columns in this order:
    - `[value_column, variable_column]`
- Output behavior:
  - all non-melted columns are `id_vars`
  - melted value goes to `value_column`
  - original column name goes to `variable_column`
- Row count: `reshape`
- Fail if any input column is missing.

## Wave 2 Blocks

### SamplerBlock

- Purpose: sample values from per-row collections.
- Inputs:
  - exactly one input column
  - exactly one output column
  - `num_samples >= 1`
  - optional `random_seed`
  - `return_scalar`
- Accepted input cell types:
  - `list`
  - `set`
  - array-like iterable
  - `dict[item, weight]` for weighted sampling
- Sampling rules:
  - sample without replacement
  - use `min(num_samples, len(values))`
  - weighted dicts ignore zero-weight items
  - weighted dicts require finite non-negative weights
- Output shape:
  - usually `list[Any]`
  - if `return_scalar=true` and `num_samples=1`, unwrap to scalar or `None`
- Row count: `same`

### RowMultiplierBlock

- Purpose: duplicate each row `num_samples` times.
- Inputs:
  - `num_samples >= 1`
  - optional `shuffle`
  - optional `random_seed`
- Output behavior:
  - repeat each row `num_samples` times
  - optionally shuffle final order
- Row count: `expand`

### JSONParserBlock

- Purpose: parse JSON from text and expand fields into columns.
- Inputs:
  - exactly one input column
  - optional `output_cols`
  - `field_prefix`
  - `fix_trailing_commas`
  - `extract_embedded`
  - `drop_input`
- Parsing behavior:
  - if `extract_embedded=true`, try to find JSON object or array inside surrounding text
  - fix common trailing comma issues if enabled
  - parsed dict -> expand keys to columns
  - parsed list -> wrap as `items`
  - parsed scalar -> wrap as `value`
- Output behavior:
  - if `output_cols` is empty, keep all parsed fields
  - if `output_cols` is provided, keep requested columns that actually exist
  - if none of the requested columns exist, keep all parsed fields
  - apply `field_prefix` after field selection
  - drop input column if requested
- Row count: `same`

### JSONStructureBlock

- Purpose: combine multiple columns into one JSON string column.
- Inputs:
  - `input_cols`: list of columns
  - `output_cols`: exactly one column
  - `ensure_json_serializable`
  - `pretty_print`
- Behavior:
  - create dict whose keys match `input_cols`
  - values come directly from the row
  - if `ensure_json_serializable=true`, recursively convert unknown values to strings
  - serialize to JSON text
- Output shape:
  - output column contains JSON string, not dict
- Row count: `same`

## Plugin / Later Blocks

### AgentBlock

- Purpose: send row-level messages to an external agent runtime.
- Status: plugin / later
- Inputs:
  - `agent_framework`
  - `agent_url`
  - optional `agent_api_key`
  - `input_cols` pointing at a messages-like column or plain text column
  - one output column
- Message normalization:
  - list -> use as-is
  - dict -> wrap in a one-element list
  - scalar -> wrap as one `user` message
- Output shape:
  - one response dict per row
- Row count: `same`
- Old repo behavior is tied to connectors. New repo should treat this as a separate plugin boundary.

### AgentResponseExtractorBlock

- Purpose: extract text, session id, and tool trace from agent responses.
- Status: plugin / later
- Inputs:
  - exactly one input column
  - accepted cell shapes: `dict` or `list[dict]`
  - `agent_framework`
  - booleans:
    - `extract_text`
    - `extract_session_id`
    - `extract_tool_trace`
  - `expand_lists`
  - `field_prefix`
- Current old behavior is Langflow-specific.
- Output columns:
  - default prefix is `block_name + "_"`
  - generated names:
    - `<prefix>text`
    - `<prefix>session_id`
    - `<prefix>tool_trace`
- Row count behavior mirrors `LLMResponseExtractorBlock`.

## Coverage Map

### Knowledge / Evaluation Flows

Need:

- `PromptBuilderBlock`
- `LLMChatBlock`
- `LLMResponseExtractorBlock`
- `TagParserBlock`
- `DuplicateColumnsBlock`
- `RenameColumnsBlock`
- `ColumnValueFilterBlock`
- `RegexParserBlock`
- `MeltColumnsBlock`

### Red Team / Structured Insights

Need in addition:

- `SamplerBlock`
- `RowMultiplierBlock`
- `JSONParserBlock`
- `JSONStructureBlock`

### Agentic MCP Distillation

Needs later plugin work:

- `AgentBlock`
- `AgentResponseExtractorBlock`

## Implementation Notes

- Start from the block spec, not the old codebase.
- If a block can be implemented in one clear function, do that.
- If a helper makes the code harder to read, inline it.
- Preserve weird behavior only when a real flow depends on it.
