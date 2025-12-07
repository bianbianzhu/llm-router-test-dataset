# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **Note**: For comprehensive user documentation, see [README.md](./README.md).

## Project Overview

This is a **self-evolving** intent classification test dataset and evaluation framework for LLM-based intent routing systems. It uses LangChain, LangSmith, and OpenAI to classify user queries into predefined intent categories, with automated optimization until benchmark exceeds 95%.

## Common Commands

```bash
# Install dependencies
pnpm install

# Run tests
pnpm test              # Interactive mode
pnpm test:run          # Single run
pnpm test:watch        # Watch mode

# Create dataset in LangSmith
pnpm create:dataset <datasetName> [--fileName <prefix>] [--type test|loose]
# Example: pnpm create:dataset my-dataset -f faq --type test

# Run evaluation and generate report
pnpm eval:report <datasetName> [--prefix <experimentPrefix>] [--concurrency none|low|high]
# Example: pnpm eval:report my-dataset -p "experiment_1" -c high

# List datasets
pnpm langsmith:basic --operation list-datasets

# Show the constructed system prompt
pnpm show:prompt
```

## Workflow Files (AI Agent Prompts)

These files are designed for AI agents (Claude, Gemini CLI, etc.) to execute complete workflows:

| File | Purpose |
|------|---------|
| `toml_prompt_v1.1.md` | Intent creation & data generation workflow (Steps 0-7) |
| `toml_prompt_eval_v1.md` | Evaluation & self-evolution workflow (Steps 1-3) |
| `template.md` | Intent definition template structure |

**Self-Evolution Loop**: Run evaluation → Analyze failures → Optimize intent definitions → Re-evaluate → Repeat until benchmark > 95%.

## Architecture

### Core Components

- **Intent Classifier** (`src/intent_classifier/`): LangChain-based classifier using GPT-4o-mini with structured output
  - `index.ts`: Main classifier pipeline using `ChatPromptTemplate` and `withStructuredOutput`
  - `schema.ts`: Zod schema defining valid intents (coupon, FAQ, wine_recommendation, membership, delivery status, event)

- **System Prompt Construction** (`prompts/system_prompt.ts`): Dynamically builds 8-part prompts from markdown intent definitions. Validates KV cache hit threshold (1024 tokens).

- **Evaluation Framework** (`evaluation/`):
  - `scripts/eval.ts`: Runs evaluation using LangSmith's `evaluate()` function
  - `scripts/evaluator.ts`: Contains `strictEqualEvaluator` for exact intent matching
  - `scripts/eval_report.ts`: Generates reports with benchmark and failure analysis
  - `scripts/create_dataset.ts`: Uploads test data to LangSmith datasets
  - `scripts/data_formatter.ts`: Transforms JSON test data to LangSmith format

- **Context Compression** (`compression/context_compression.ts`): Chat history compression using Gemini for long conversations. Uses 70% token threshold for triggering compression, preserves 30% of recent history.

### Data Structure

- **Intent Definitions** (`intents/*.md`): Markdown files defining each intent category
  - Strict: `<intent>.md` - Used in production prompts
  - Loose: `<intent>_loose.md` - Used for robustness testing (NEVER modify during optimization)

- **Test Data** (`test_data/`): JSON files with format:
  - `{intent}_development.json`: Development set (25 examples)
  - `{intent}_test.json`: Standard test set (75 examples)
  - `{intent}_test_loose.json`: Looser test variations (75 examples)

Test data format:
```json
{
  "id": "uuid",
  "input": "user query",
  "intent": "expected_intent"
}
```

### Key Dependencies

- `@langchain/openai`, `@langchain/core`: LLM interaction
- `langsmith`: Evaluation infrastructure
- `openevals`: LLM-as-judge evaluator
- `tiktoken`: Token counting for prompt optimization
- `@google/genai`: Gemini for context compression
- `zod`: Schema validation

## Adding New Intents

1. Create `intents/<name>.md` following `template.md` structure
2. Create `intents/<name>_loose.md` for broader variations
3. Update `src/intent_classifier/schema.ts` to add the new intent to the enum
4. Generate test data (development, test, test_loose)
5. Upload to LangSmith and run evaluation

## Environment Variables

Required in `.env`:
- `OPENAI_API_KEY`
- `LANGCHAIN_API_KEY` (for LangSmith)
- `GEMINI_API_KEY` (for compression features)
