# OpenAI Assistants — Nativ Function Calling

Use Nativ as a tool inside any OpenAI Assistant. The assistant decides when to translate, your code executes the Nativ API call, and the result flows back into the conversation.

## How it works

```
User message
    ↓
OpenAI Assistant (decides to call translate_text)
    ↓
Your code receives the function call + arguments
    ↓
Your code calls the Nativ API (via SDK or HTTP)
    ↓
Result returned to the assistant
    ↓
Assistant formats the response for the user
```

**Key difference from Custom GPT:** In Custom GPT Actions, ChatGPT calls the Nativ API directly. With the Assistants API, *your code* is the middleware — you receive function calls, execute them, and return results. This gives you full control over auth, error handling, and logging.

## Prerequisites

- An OpenAI API key with Assistants API access
- A Nativ API key (`nativ_xxx...` from dashboard.usenativ.com)
- Python 3.9+ (examples use the Nativ Python SDK)

## Quick start

### 1. Install dependencies

```bash
pip install openai nativ
```

### 2. Create the assistant (one-time)

```python
import json
from openai import OpenAI

client = OpenAI()

with open("tools.json") as f:
    tools = json.load(f)

assistant = client.beta.assistants.create(
    name="Nativ Localization",
    instructions=(
        "You are a localization assistant powered by Nativ. "
        "You help users translate and culturally adapt text using "
        "translation memory, brand voice, glossaries, and style guides.\n\n"
        "When translating to multiple languages, make separate function "
        "calls for each language. Always pass context when the user "
        "provides it (e.g. 'luxury fashion headline', 'mobile app button'). "
        "For important translations, enable include_tm_info and backtranslate "
        "so the user can verify quality.\n\n"
        "Be concise and professional. Format translations clearly with "
        "the language name and result."
    ),
    tools=tools,
    model="gpt-4o",
)

print(f"Assistant ID: {assistant.id}")
# Save this — you reuse the same assistant for all conversations.
```

### 3. Run a conversation with function calling

```python
import json
import base64
from openai import OpenAI
from nativ import NativClient

openai_client = OpenAI()
nativ = NativClient(api_key="nativ_your_key_here")

ASSISTANT_ID = "asst_xxxxxxxxxxxx"  # from step 2


def handle_function_call(name, arguments):
    """Execute a Nativ API call and return the result as a string."""

    if name == "translate_text":
        params = {k: v for k, v in arguments.items() if v is not None}
        params["tool"] = "api"
        result = nativ.translate(**params)
        return json.dumps({
            "translated_text": result.translated_text,
            "metadata": {"word_count": result.metadata.word_count, "cost": result.metadata.cost}
            if result.metadata else None,
        })

    if name == "extract_text_from_image":
        image_bytes = base64.b64decode(arguments["image_base64"])
        result = nativ.extract_text(image_bytes, arguments["filename"])
        return json.dumps({"extracted_text": result.extracted_text})

    if name == "generate_styled_text_image":
        image_bytes = base64.b64decode(arguments["image_base64"])
        params = {k: v for k, v in arguments.items()
                  if k not in ("image_base64", "filename") and v is not None}
        result = nativ.culturalize_image(image_bytes, arguments["filename"], **params)
        return json.dumps({
            "images": [{"image_base64": img.image_base64} for img in result.images],
            "metadata": {"cost": result.metadata.cost, "num_images": result.metadata.num_images}
            if result.metadata else None,
        })

    if name == "inspect_cultural_sensitivity":
        image_bytes = base64.b64decode(arguments["image_base64"])
        countries = arguments.get("countries")
        result = nativ.inspect_image(image_bytes, arguments["filename"],
                                     countries=countries)
        return json.dumps({
            "verdict": result.verdict,
            "affected_countries": [
                {"country": c.country, "issue": c.issue, "suggestion": c.suggestion}
                for c in (result.affected_countries or [])
            ],
        })

    return json.dumps({"error": f"Unknown function: {name}"})


def chat(user_message):
    """Send a message and handle the full function-calling loop."""

    thread = openai_client.beta.threads.create()
    openai_client.beta.threads.messages.create(
        thread_id=thread.id, role="user", content=user_message
    )

    run = openai_client.beta.threads.runs.create_and_poll(
        thread_id=thread.id, assistant_id=ASSISTANT_ID
    )

    while run.status == "requires_action":
        tool_outputs = []
        for call in run.required_action.submit_tool_outputs.tool_calls:
            args = json.loads(call.function.arguments)
            result = handle_function_call(call.function.name, args)
            tool_outputs.append({"tool_call_id": call.id, "output": result})

        run = openai_client.beta.threads.runs.submit_tool_outputs_and_poll(
            thread_id=thread.id, run_id=run.id, tool_outputs=tool_outputs
        )

    messages = openai_client.beta.threads.messages.list(thread_id=thread.id)
    return messages.data[0].content[0].text.value


# Try it
print(chat("Translate 'Shop the collection' to French and Japanese"))
```

## What each tool does

| Tool | Nativ endpoint | Description |
|---|---|---|
| `translate_text` | `POST /text/culturalize` | Translate text with TM, brand voice, glossary, formality, and backtranslation |
| `extract_text_from_image` | `POST /text/extract` | OCR — extract visible text from an image |
| `generate_styled_text_image` | `POST /image/culturalize` | Generate a new image with translated text in the same visual style |
| `inspect_cultural_sensitivity` | `POST /image/inspect` | Check an image for cultural sensitivity issues across countries |

## Using the tools.json file

The `tools.json` file contains all four function definitions in the format expected by `client.beta.assistants.create(tools=...)`. You can:

- Use all four tools (recommended)
- Use only `translate_text` for a text-only assistant
- Add your own custom tools alongside the Nativ ones

## Authentication

Each API call uses **your** Nativ API key. This means:

- All translations bill to your account
- The assistant inherits your TM, brand voice, glossaries, and style guides
- If you're building a multi-tenant app, use a different Nativ API key per customer

## Cost

- Text translation: 100 credits per call (~$0.10 USD)
- Image culturalization: 500 credits per call (~$0.50 USD)
- OCR / cultural sensitivity check: 100 credits per call (~$0.10 USD)
- Plus standard OpenAI Assistants API costs

## Tips

- **Always pass `context`** when the user provides it — it significantly improves translation quality
- **Use `max_characters`** for UI strings (buttons, labels, menu items) to ensure they fit
- **Enable `backtranslate` + `include_rationale`** for high-stakes translations so users can verify quality
- **Reuse the assistant** — create it once, then use the same `ASSISTANT_ID` for all threads
- **Thread = conversation** — create a new thread for each independent conversation

## Streaming (optional)

For a real-time UX, use streaming with event handlers:

```python
from openai import AssistantEventHandler

class NativHandler(AssistantEventHandler):
    def on_text_delta(self, delta, snapshot):
        print(delta.value, end="", flush=True)

    def on_tool_call_created(self, tool_call):
        print(f"\n[Calling {tool_call.function.name}...]")

with openai_client.beta.threads.runs.stream(
    thread_id=thread.id,
    assistant_id=ASSISTANT_ID,
    event_handler=NativHandler(),
) as stream:
    stream.until_done()
```

## Comparison: Custom GPT vs Assistants API

| | Custom GPT | Assistants API |
|---|---|---|
| **Who calls the API** | ChatGPT calls Nativ directly | Your code calls Nativ |
| **Auth** | API Key or OAuth in GPT Actions | Your Nativ API key in your code |
| **UI** | ChatGPT web/mobile | Your own app, CLI, or service |
| **Customization** | Limited to GPT editor | Full control — add any logic |
| **Best for** | End users in ChatGPT | Developers building products |
