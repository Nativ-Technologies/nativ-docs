# CrewAI — Nativ Localization Tools

Use Nativ inside any CrewAI crew. Give your agents the ability to translate, search translation memory, and leverage brand voice — all through natural language.

## How it works

```
CrewAI Agent receives task
    ↓
Agent decides to call a Nativ tool (translate, search TM, etc.)
    ↓
Tool calls the Nativ API via the Python SDK
    ↓
Result flows back into the agent's reasoning
    ↓
Agent continues with localized content
```

## Prerequisites

- Python 3.9+
- A Nativ API key (`nativ_xxx...` from dashboard.usenativ.com)
- CrewAI installed (`pip install crewai`)

## Quick start

### 1. Install dependencies

```bash
pip install crewai nativ
```

### 2. Define Nativ tools for CrewAI

CrewAI uses its own `BaseTool` class. Here's how to wrap the Nativ Python SDK:

```python
import os
from crewai.tools import BaseTool
from nativ import Nativ

NATIV_API_KEY = os.environ.get("NATIV_API_KEY", "nativ_your_key_here")


class TranslateTool(BaseTool):
    name: str = "Translate Text"
    description: str = (
        "Translate text to another language using Nativ's AI localization engine. "
        "Uses the team's translation memory, brand voice, and style guides. "
        "Args: text (str), target_language (str, e.g. 'French'), "
        "context (str, optional, e.g. 'marketing headline')."
    )

    def _run(self, text: str, target_language: str, context: str = "") -> str:
        with Nativ(api_key=NATIV_API_KEY) as client:
            result = client.translate(
                text, target_language, context=context or None
            )
        out = result.translated_text
        if result.rationale:
            out += f"\nRationale: {result.rationale}"
        return out


class TranslateBatchTool(BaseTool):
    name: str = "Translate Batch"
    description: str = (
        "Translate multiple texts to the same target language in one call. "
        "Args: texts (list[str]), target_language (str)."
    )

    def _run(self, texts: list, target_language: str) -> str:
        with Nativ(api_key=NATIV_API_KEY) as client:
            results = client.translate_batch(texts, target_language)
        return "\n".join(
            f"{i+1}. {r.translated_text}" for i, r in enumerate(results)
        )


class SearchTranslationMemoryTool(BaseTool):
    name: str = "Search Translation Memory"
    description: str = (
        "Search the translation memory for existing translations. "
        "Args: query (str), target_language_code (str, optional, e.g. 'fr')."
    )

    def _run(self, query: str, target_language_code: str = "") -> str:
        with Nativ(api_key=NATIV_API_KEY) as client:
            matches = client.search_tm(
                query,
                target_language_code=target_language_code or None,
            )
        if not matches:
            return "No matches found."
        lines = [f"Found {len(matches)} match(es):"]
        for m in matches:
            lines.append(
                f'- [{m.score:.0f}% {m.match_type}] '
                f'"{m.source_text}" -> "{m.target_text}"'
            )
        return "\n".join(lines)


class GetBrandVoiceTool(BaseTool):
    name: str = "Get Brand Voice"
    description: str = (
        "Get the brand voice prompt that shapes all translations. "
        "No arguments needed."
    )

    def _run(self) -> str:
        with Nativ(api_key=NATIV_API_KEY) as client:
            bv = client.get_brand_voice()
        if not bv.exists or not bv.prompt:
            return "No brand voice configured."
        return f"Brand voice:\n{bv.prompt}"


class GetLanguagesTool(BaseTool):
    name: str = "Get Languages"
    description: str = (
        "List all target languages configured in the Nativ workspace. "
        "No arguments needed."
    )

    def _run(self) -> str:
        with Nativ(api_key=NATIV_API_KEY) as client:
            langs = client.get_languages()
        if not langs:
            return "No languages configured."
        return "\n".join(
            f"- {lang.language} ({lang.language_code})" for lang in langs
        )
```

### 3. Create a crew with Nativ tools

```python
from crewai import Agent, Task, Crew

tools = [
    TranslateTool(),
    TranslateBatchTool(),
    SearchTranslationMemoryTool(),
    GetBrandVoiceTool(),
    GetLanguagesTool(),
]

translator = Agent(
    role="Localization Specialist",
    goal="Translate and culturally adapt content while maintaining brand voice",
    backstory=(
        "You are an expert localization specialist who uses Nativ's AI engine "
        "to produce on-brand translations. You always check the brand voice "
        "first, search the translation memory for consistency, and provide "
        "context when translating."
    ),
    tools=tools,
    verbose=True,
)

task = Task(
    description=(
        "Translate the following marketing copy to French and Japanese:\n\n"
        "'Shop the collection - timeless pieces for the modern wardrobe.'\n\n"
        "Check the brand voice first, then translate with appropriate context."
    ),
    expected_output="Translations in French and Japanese with rationale.",
    agent=translator,
)

crew = Crew(agents=[translator], tasks=[task], verbose=True)
result = crew.kickoff()
print(result)
```

## Alternative: Using langchain-nativ

If you already use `langchain-nativ`, you can wrap those tools for CrewAI:

```bash
pip install crewai langchain-nativ
```

```python
from crewai.tools import BaseTool
from langchain_nativ import NativToolkit

lc_tools = NativToolkit(api_key="nativ_your_key_here").get_tools()


class LangChainToolWrapper(BaseTool):
    """Wrap a LangChain tool for CrewAI."""
    lc_tool: object = None

    def __init__(self, lc_tool):
        super().__init__(
            name=lc_tool.name,
            description=lc_tool.description,
            lc_tool=lc_tool,
        )

    def _run(self, **kwargs) -> str:
        return self.lc_tool.run(kwargs)


crewai_tools = [LangChainToolWrapper(t) for t in lc_tools]
# Pass crewai_tools to your agent's tools= parameter
```

## Multi-agent example

CrewAI shines with multiple specialized agents. Here is a localization pipeline:

```python
from crewai import Agent, Task, Crew, Process

reviewer = Agent(
    role="Quality Reviewer",
    goal="Review translations for accuracy and brand consistency",
    backstory=(
        "You review translations by checking them against the brand voice "
        "and translation memory. You flag inconsistencies and suggest fixes."
    ),
    tools=[SearchTranslationMemoryTool(), GetBrandVoiceTool()],
    verbose=True,
)

translate_task = Task(
    description="Translate the product descriptions to German.",
    expected_output="German translations of all product descriptions.",
    agent=translator,
)

review_task = Task(
    description="Review the German translations for brand voice consistency.",
    expected_output="Review report with any issues flagged.",
    agent=reviewer,
)

crew = Crew(
    agents=[translator, reviewer],
    tasks=[translate_task, review_task],
    process=Process.sequential,
    verbose=True,
)

result = crew.kickoff()
```

## Available tools

| Tool | Nativ endpoint | Description |
|---|---|---|
| `TranslateTool` | `POST /text/culturalize` | Translate text with TM, brand voice, and glossary |
| `TranslateBatchTool` | `POST /text/culturalize/batch` | Translate multiple texts at once |
| `SearchTranslationMemoryTool` | `GET /tm/search` | Search TM for existing translations |
| `GetBrandVoiceTool` | `GET /brand-voice` | Get the brand voice prompt |
| `GetLanguagesTool` | `GET /languages` | List configured target languages |

## Tips

- **Always check brand voice first** — instruct your agent to read the brand voice before translating
- **Use context** — pass context like "luxury fashion headline" or "mobile app button" for better results
- **Search TM before translating** — this ensures consistency with previous translations
- **Multi-agent crews** — use one agent to translate and another to review for quality assurance
- **Sequential process** — use `Process.sequential` when review depends on translation output
