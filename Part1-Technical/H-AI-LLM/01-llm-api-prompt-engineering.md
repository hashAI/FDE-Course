# 01 — LLM API & Prompt Engineering
## FDE Course: AI/LLM Fundamentals for Logistics/Manufacturing/Supply Chain

**Prerequisites:** Python intermediate, basic HTTP/REST  
**Related files:**  
- [02-tool-calling.md](./02-tool-calling.md) — structured execution via tools  
- [03-rag.md](./03-rag.md) — grounding LLM responses in customer data  
- [07-guardrails-cost-tradeoffs.md](./07-guardrails-cost-tradeoffs.md) — cost estimation, prompt caching

---

## Why This File Exists

As an FDE, you are the technical interface between Anthropic's capabilities and a customer's messy real-world problem. You'll sit in a room with a Head of Logistics at a 3PL (third-party logistics provider) who says: "We receive 4,000 carrier invoices per month as PDFs. We manually key in 12 fields from each one. Can your AI fix that?"

Your job is to:
1. Know immediately that this is an **extraction task** — structured output from unstructured document
2. Know which model to use (Haiku for speed/cost, Sonnet if accuracy matters on complex invoices)
3. Know how to write the prompt (system prompt + few-shot examples + XML-wrapped document)
4. Know how to validate the output (Pydantic)
5. Know how to handle failures (retry with correction prompt)
6. Know how to estimate the cost (input tokens × price + output tokens × price)

This file gives you all of that.

---

## Part 1: LLM API Fundamentals

### 1.1 The Anthropic Claude API — Core Concepts

The Anthropic API is a **stateless HTTP API**. Every call is independent — there is no server-side session. You pass the entire conversation history in each request. This is important for understanding cost (you pay for all tokens every time) and context management (you have a fixed window to work with).

**The Messages API** (`POST /v1/messages`) is the main endpoint for Claude. It takes:
- A **model** identifier
- A list of **messages** (conversation turns)
- An optional **system** prompt
- Generation parameters (max_tokens, temperature, etc.)

And returns a **Message** object with:
- **content**: list of content blocks (text, tool_use)
- **stop_reason**: why generation stopped
- **usage**: token counts

### 1.2 Installing and Configuring the SDK

```python
# Install
# pip install anthropic

import anthropic
import os

# The SDK reads ANTHROPIC_API_KEY from environment automatically
# Or pass explicitly:
client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

# Async client for production use
async_client = anthropic.AsyncAnthropic(api_key=os.environ["ANTHROPIC_API_KEY"])
```

**FDE context:** In customer demos, always use environment variables. Never hardcode keys. If you're building a demo on a customer's laptop, show them how to set the env var. This builds trust that you understand security basics.

### 1.3 Your First API Call

```python
import anthropic

client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": "What is the HS code for laptop computers?"
        }
    ]
)

print(message.content[0].text)
# "The HS code for laptop computers is 8471.30 under the Harmonized System..."
```

### 1.4 The Message Object — Complete Structure

Understanding the response object is critical. Here's the full breakdown:

```python
import anthropic
from anthropic.types import Message, TextBlock

client = anthropic.Anthropic()

response: Message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system="You are a logistics data extraction assistant.",
    messages=[
        {"role": "user", "content": "Extract the tracking number from: FEDEX-TRACK-789012"}
    ]
)

# Response structure:
print(type(response))                    # <class 'anthropic.types.message.Message'>
print(response.id)                       # 'msg_01XFDUDYJgAACzvnptvVoYEL'
print(response.type)                     # 'message'
print(response.role)                     # 'assistant'
print(response.model)                    # 'claude-sonnet-4-6'

# Content blocks — always a list, even for simple text
print(response.content)                  # [TextBlock(text='The tracking number is...', type='text')]
print(len(response.content))             # 1 (just text in this case)
print(response.content[0].type)          # 'text'
print(response.content[0].text)          # 'The tracking number is FEDEX-TRACK-789012.'

# Stop reason — CRITICAL for agentic loops
print(response.stop_reason)             # 'end_turn' (normal completion)
# Other values: 'max_tokens', 'stop_sequence', 'tool_use'

# Usage — for cost tracking
print(response.usage.input_tokens)      # 45
print(response.usage.output_tokens)     # 23
print(response.usage.cache_creation_input_tokens)  # 0 (no caching used)
print(response.usage.cache_read_input_tokens)      # 0

# Safe text extraction helper
def get_text(response: Message) -> str:
    """Extract text from a Message, handling edge cases."""
    for block in response.content:
        if hasattr(block, 'text'):
            return block.text
    return ""

text = get_text(response)
```

### 1.5 Roles and Conversation Structure

Claude uses a **turn-based conversation model** with three roles:

**`system`**: Sets the LLM's persona, constraints, and behavioral rules. Not a "turn" — it's a persistent instruction. Passed as a top-level parameter, not inside the messages array.

**`user`**: Input from the human (or your application pretending to be the human). Always the odd-numbered turns (1st, 3rd, 5th...).

**`assistant`**: Claude's response. Always even-numbered turns. You can also **pre-fill** the assistant turn to force a specific response format (more on this below).

**Rules that will bite you if you ignore them:**
- Messages must alternate: user, assistant, user, assistant...
- You CANNOT have two consecutive user or assistant messages
- The first message MUST be a user message
- The system prompt is NOT a message — it's separate

```python
# Multi-turn conversation example
messages = [
    {
        "role": "user",
        "content": "I have a shipment from Shanghai to LA. What documents do I need?"
    },
    {
        "role": "assistant", 
        "content": "For a shipment from Shanghai to Los Angeles, you'll need: 1) Commercial Invoice..."
    },
    {
        "role": "user",
        "content": "The goods are electronics worth $50,000. Does that change anything?"
    }
    # Next call will add another assistant response
]

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system="You are an expert in international trade compliance.",
    messages=messages
)
```

### 1.6 Content Blocks

The `content` field in a message can be a string OR a list of content blocks. Content blocks allow mixed media:

```python
# Simple string (equivalent to a single text block)
messages = [{"role": "user", "content": "What is this?"}]

# Content blocks — more explicit, required for images/tool results
messages = [
    {
        "role": "user",
        "content": [
            {
                "type": "text",
                "text": "Extract the data from this invoice image:"
            },
            {
                "type": "image",
                "source": {
                    "type": "base64",
                    "media_type": "image/jpeg",
                    "data": "<base64_encoded_image_data>"
                }
            }
        ]
    }
]

# For tool results (see 02-tool-calling.md)
# For document reading (PDF, text files)
messages = [
    {
        "role": "user",
        "content": [
            {
                "type": "document",
                "source": {
                    "type": "text",
                    "media_type": "text/plain",
                    "data": "INVOICE #12345\nShipper: Acme Corp..."
                }
            },
            {
                "type": "text",
                "text": "Extract all line items from this invoice as JSON."
            }
        ]
    }
]
```

### 1.7 Generation Parameters — What They Actually Do

**`model`** (required): Which Claude model to use. See Section 1.9 for selection guide.

**`max_tokens`** (required): The MAXIMUM number of output tokens. Claude will stop when it reaches this limit OR when it's naturally done (whichever comes first). Setting this too low truncates responses. Setting it too high doesn't cost you — you only pay for tokens actually generated.

```python
# WRONG: max_tokens too low for a complex extraction task
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=100,  # Will truncate JSON mid-output!
    messages=[{"role": "user", "content": "Extract all 20 fields from this invoice..."}]
)

# RIGHT: set high enough to never truncate
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=4096,  # Safe upper bound for most extraction tasks
    messages=[{"role": "user", "content": "Extract all 20 fields from this invoice..."}]
)
```

**`temperature`** (0.0 to 1.0): Controls randomness.
- `0.0`: Deterministic, always picks highest probability token. Use for extraction, classification, structured output.
- `1.0`: Maximum randomness. Use for creative writing, brainstorming.
- For logistics/manufacturing FDE work: **use 0.0 for data extraction, 0.3-0.5 for analysis/summaries, 0.7+ for creative content** (rare).

```python
# Extraction task — deterministic
response = client.messages.create(
    model="claude-haiku-4-5-20251001",
    max_tokens=1024,
    temperature=0.0,  # Deterministic — same invoice always gives same extraction
    messages=[{"role": "user", "content": extract_prompt}]
)

# Analysis task — slight creativity allowed
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=2048,
    temperature=0.3,
    messages=[{"role": "user", "content": analysis_prompt}]
)
```

**`top_p`** (0.0 to 1.0): Nucleus sampling. Only sample from tokens whose cumulative probability exceeds `top_p`. Alternative to temperature. Anthropic recommends adjusting temperature OR top_p, not both.

**`top_k`** (integer): Only sample from the top-k most probable tokens. Less common. Skip this in most implementations.

**`stop_sequences`** (list of strings): Generation stops when any of these strings is produced. Useful for structured output:

```python
# Stop at end of JSON object — prevents model from continuing after the JSON
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=2048,
    stop_sequences=["```", "---"],  # Stop at code fence or divider
    messages=[{"role": "user", "content": "Output JSON, then stop."}]
)

# In practice, better to just ask for JSON and parse it
# stop_sequences is more useful in older completion-style APIs
```

### 1.8 Async API — Required for Production

For any production system or customer demo that handles multiple requests:

```python
import anthropic
import asyncio
from typing import Any

async_client = anthropic.AsyncAnthropic()

async def extract_invoice_data(invoice_text: str) -> dict:
    """Extract structured data from an invoice asynchronously."""
    response = await async_client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=1024,
        temperature=0.0,
        system=INVOICE_EXTRACTION_SYSTEM_PROMPT,
        messages=[
            {"role": "user", "content": f"<invoice>\n{invoice_text}\n</invoice>"}
        ]
    )
    return response

# Process 50 invoices in parallel
async def process_invoice_batch(invoices: list[str]) -> list[dict]:
    tasks = [extract_invoice_data(inv) for inv in invoices]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    
    processed = []
    for i, result in enumerate(results):
        if isinstance(result, Exception):
            print(f"Invoice {i} failed: {result}")
            processed.append(None)
        else:
            processed.append(result)
    return processed

# Run it
invoices = ["invoice text 1", "invoice text 2", ...]
results = asyncio.run(process_invoice_batch(invoices))
```

**FDE context:** When a customer asks "can you handle our volume?", knowing async/parallel processing is the answer. 4,000 invoices/month = ~130/day. At 2 seconds per invoice with parallel processing (20 concurrent), that's well under an hour's processing time. Be ready to size this on a whiteboard.

### 1.9 Model Selection Guide

This is a question you WILL be asked: "Which model should we use?"

| Model | Speed | Cost | Context | Best For |
|-------|-------|------|---------|----------|
| `claude-haiku-4-5-20251001` | Fastest | Cheapest (~50x less than Opus) | 200K | Extraction, classification, simple transforms |
| `claude-sonnet-4-6` | Balanced | Moderate | 200K | Most tasks, analysis, coding, complex extraction |
| `claude-opus-4-5` | Slowest | Most expensive | 200K | Complex reasoning, high-stakes decisions, nuanced analysis |

**Decision framework:**

```python
def select_model(task_type: str, accuracy_requirement: str, latency_requirement: str) -> str:
    """
    FDE model selection logic.
    
    In practice, this is a judgment call based on:
    - Task complexity
    - Error cost (what happens if the AI gets it wrong?)
    - Volume/budget
    - Latency requirements
    """
    
    # High-stakes decisions: always use Sonnet or better
    HIGH_STAKES_TASKS = [
        "customs_classification",      # Wrong HS code = fines/delays
        "compliance_check",            # Compliance failure = regulatory action
        "contract_review",             # Legal implications
        "safety_assessment"            # Worker/product safety
    ]
    
    # Simple extraction: Haiku is fine
    SIMPLE_EXTRACTION_TASKS = [
        "invoice_field_extraction",    # Tracking number, amounts, dates
        "address_parsing",             # Street, city, postal code
        "status_classification",       # Delivered/In Transit/Failed
        "language_detection"
    ]
    
    if task_type in HIGH_STAKES_TASKS:
        return "claude-sonnet-4-6"  # Or opus for highest stakes
    
    if task_type in SIMPLE_EXTRACTION_TASKS and accuracy_requirement == "standard":
        return "claude-haiku-4-5-20251001"
    
    # Default: Sonnet for the balance of capability and cost
    return "claude-sonnet-4-6"
```

**Real example — the cost math:**
- 4,000 invoices/month
- Average invoice: 500 input tokens + 200 output tokens
- **With Haiku**: 4000 × (500 × $0.00025/1K + 200 × $0.00125/1K) = $0.50 + $1.00 = **$1.50/month**
- **With Sonnet**: 4000 × (500 × $0.003/1K + 200 × $0.015/1K) = $6.00 + $12.00 = **$18/month**
- **With Opus**: 4000 × (500 × $0.015/1K + 200 × $0.075/1K) = $30.00 + $60.00 = **$90/month**

For a basic extraction task, Haiku at $1.50/month is the obvious choice. For a compliance check where errors cost $10K in fines, Sonnet at $18/month is cheap insurance.

---

## Part 2: Prompt Engineering Patterns

### 2.1 System Prompts — The Foundation

The system prompt defines who the AI is and how it behaves. It's the most important part of your prompt architecture.

**What system prompts control:**
- Persona: "You are an expert in dangerous goods shipping regulations"
- Output format: "Always respond in JSON"
- Scope constraints: "Only answer questions about logistics. For other topics, say 'I can only help with logistics questions.'"
- Reasoning approach: "Think through compliance requirements step by step"
- Error behavior: "If you cannot extract a field with confidence, use null for that field"

**System prompt anatomy:**

```python
INVOICE_EXTRACTION_SYSTEM_PROMPT = """You are a logistics data extraction specialist. Your job is to extract structured data from carrier invoices, freight bills, and shipping documents.

## Your Role
Extract data accurately and completely. When information is ambiguous, use your best judgment and flag it. When information is missing, use null.

## Output Format
Always respond with a single JSON object. No preamble, no explanation — just the JSON.

## Field Definitions
- invoice_number: The unique invoice identifier (string)
- invoice_date: Date in YYYY-MM-DD format
- carrier_name: The name of the shipping carrier
- shipper_name: Name of the party shipping the goods
- consignee_name: Name of the party receiving the goods  
- origin_port: Port/city of departure (string)
- destination_port: Port/city of arrival (string)
- weight_kg: Total weight in kilograms (float, convert if given in lbs: multiply by 0.453592)
- freight_charges_usd: Total freight cost in USD (float, convert from other currencies using approximate rates if needed)
- currency: Original currency of the invoice (string, 3-letter ISO code)

## Confidence Flagging
If you are not confident in a field value, append "_uncertain": true to the object. Example:
{"invoice_number": "INV-12345", "invoice_number_uncertain": true}

## Rules
- Never invent data. If a field is not present, use null.
- Dates must be in YYYY-MM-DD format.
- Numbers must be numeric types (not strings).
- Carrier names: standardize to common forms (e.g., "FedEx" not "Federal Express")
"""
```

**FDE context:** Notice the system prompt is detailed and specific. A vague system prompt like "Extract invoice data" produces inconsistent results. The more specific your constraints, the more consistent the output. This matters when you're building a system that feeds into a database — you can't have random field names or inconsistent date formats.

### 2.2 Zero-Shot Prompting

Direct instruction with no examples. Works for:
- Well-defined tasks with clear correct answers
- Tasks the model has been trained extensively on
- Simple extraction from clean documents

```python
# Zero-shot extraction
def extract_tracking_number_zero_shot(text: str) -> str:
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=100,
        temperature=0.0,
        system="Extract the tracking number from the text. Respond with only the tracking number, nothing else.",
        messages=[
            {"role": "user", "content": text}
        ]
    )
    return response.content[0].text.strip()

# Works well for clean inputs:
result = extract_tracking_number_zero_shot(
    "Your package with tracking number 1Z999AA10123456784 is out for delivery."
)
# Returns: "1Z999AA10123456784"

# May struggle with ambiguous inputs:
result = extract_tracking_number_zero_shot(
    "Reference: 456789, PO: 123456, AWB: 172-12345678"
)
# LLM has to guess which number is the "tracking number" — no guidance
```

### 2.3 Few-Shot Prompting

Include 2-5 examples in the prompt. This is the single highest-leverage prompt improvement you can make for extraction tasks.

```python
FEW_SHOT_SYSTEM_PROMPT = """You are a logistics data extraction specialist.

Extract the following fields from carrier invoices as JSON:
- invoice_number (string)
- total_amount_usd (float)
- service_type (string: "Express", "Ground", "Ocean", "Air")
- weight_kg (float)

Here are examples of correct extractions:

<example>
<input>
FEDEX INVOICE
Invoice Number: FX-2024-089234
Service: FedEx Priority Overnight
Billed Weight: 2.5 KG
Total Charges: USD 127.50
</input>
<output>
{"invoice_number": "FX-2024-089234", "total_amount_usd": 127.50, "service_type": "Express", "weight_kg": 2.5}
</output>
</example>

<example>
<input>
UPS FREIGHT BILL
Bill Number: 1Z4589W70345678901
Service Level: UPS Ground
Actual Weight: 45 lbs
Invoice Total: $234.67 USD
</input>
<output>
{"invoice_number": "1Z4589W70345678901", "total_amount_usd": 234.67, "service_type": "Ground", "weight_kg": 20.41}
</output>
</example>

<example>
<input>
OCEAN FREIGHT INVOICE
Ref: MAEU-2024-BL-789012
B/L Number: MAEUBKK2345678
Ocean Freight: EUR 2,450.00
Exchange Rate: 1 EUR = 1.08 USD
Gross Weight: 12,500 KGS
</input>
<output>
{"invoice_number": "MAEU-2024-BL-789012", "total_amount_usd": 2646.00, "service_type": "Ocean", "weight_kg": 12500.0}
</output>
</example>

Always respond with only the JSON object, no explanation."""

def extract_invoice_few_shot(invoice_text: str) -> dict:
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=512,
        temperature=0.0,
        system=FEW_SHOT_SYSTEM_PROMPT,
        messages=[
            {"role": "user", "content": f"<input>\n{invoice_text}\n</input>"}
        ]
    )
    import json
    return json.loads(response.content[0].text)
```

**Why few-shot works:** The examples teach the model the exact format you want, how to handle edge cases (lbs → kg conversion, EUR → USD conversion), and what to do with ambiguous data. The model generalizes from the examples to handle new inputs.

**FDE context:** When a customer says "the AI is extracting data inconsistently," the first fix is always: add more examples. Get 5-10 examples from the customer's actual data, covering their most common formats. This is usually an afternoon of work that dramatically improves accuracy.

### 2.4 Chain-of-Thought (CoT) Prompting

Force the model to reason step-by-step before giving an answer. Best for:
- Classification with complex rules
- Calculations
- Compliance checks
- Any task where the reasoning path matters

```python
COT_COMPLIANCE_PROMPT = """You are a logistics compliance specialist.

Analyze the shipment below and determine if it requires a special permit.

Rules requiring special permits:
1. Lithium batteries > 100Wh per cell require Class 9 dangerous goods declaration
2. Items over $2,500 value require formal entry customs clearance (US imports)
3. Food items require FDA prior notice (US imports)
4. Controlled substances always require DEA permit
5. Items on EAR (Export Administration Regulations) list require export license

Think through each rule carefully before giving your answer.

Format your response as:
<reasoning>
[Check each rule against the shipment details]
</reasoning>
<verdict>
{
  "requires_permit": true/false,
  "applicable_rules": ["list of triggered rules"],
  "permits_needed": ["list of specific permits"],
  "confidence": "high/medium/low",
  "notes": "any additional context"
}
</verdict>"""

def check_compliance_cot(shipment_details: str) -> dict:
    """
    Uses chain-of-thought to check compliance requirements.
    The reasoning trace is invaluable for debugging and customer explanation.
    """
    response = client.messages.create(
        model="claude-sonnet-4-6",  # Use Sonnet+ for compliance — stakes are high
        max_tokens=2048,
        temperature=0.0,
        system=COT_COMPLIANCE_PROMPT,
        messages=[
            {"role": "user", "content": f"Shipment details:\n{shipment_details}"}
        ]
    )
    
    text = response.content[0].text
    
    # Extract reasoning and verdict
    import re, json
    
    reasoning_match = re.search(r'<reasoning>(.*?)</reasoning>', text, re.DOTALL)
    verdict_match = re.search(r'<verdict>(.*?)</verdict>', text, re.DOTALL)
    
    reasoning = reasoning_match.group(1).strip() if reasoning_match else ""
    verdict_json = verdict_match.group(1).strip() if verdict_match else "{}"
    
    result = json.loads(verdict_json)
    result["reasoning_trace"] = reasoning  # Keep for audit trail
    
    return result

# Test it
shipment = """
Product: Laptop computers (20 units)
Value: $32,000 USD
Batteries: Li-ion, 72Wh per cell
Weight: 45 kg
Destination: United States (import from Taiwan)
"""

result = check_compliance_cot(shipment)
# reasoning_trace will show: "Rule 1 check: 72Wh < 100Wh, no DG required. Rule 2 check: $32,000 > $2,500, formal entry required..."
# verdict: {"requires_permit": true, "permits_needed": ["Formal Entry Customs Clearance"], ...}
```

**Extended Thinking** (Claude-specific feature for complex problems):

```python
# Extended thinking: Claude thinks before responding (hidden scratch pad)
# Available on claude-sonnet-4-6 with streaming
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000  # How many tokens for internal thinking
    },
    messages=[
        {"role": "user", "content": "Complex compliance scenario here..."}
    ]
)

# Response will have thinking blocks + text blocks
for block in response.content:
    if block.type == "thinking":
        print("Internal reasoning:", block.thinking)
    elif block.type == "text":
        print("Final answer:", block.text)
```

### 2.5 Structured Output Prompting

Getting the LLM to return machine-parseable output. This is the core skill for FDE work — you're always feeding LLM output into a database, API, or downstream system.

**Method 1: Explicit JSON instruction in system prompt**

```python
STRUCTURED_OUTPUT_PROMPT = """Extract shipment data and return ONLY a JSON object with no other text.

Required JSON schema:
{
  "tracking_number": string,
  "carrier": string,
  "origin": {
    "city": string,
    "country_code": string  // ISO 3166-1 alpha-2
  },
  "destination": {
    "city": string, 
    "country_code": string
  },
  "estimated_delivery": string,  // ISO 8601 date: YYYY-MM-DD
  "weight_kg": number,
  "status": string  // one of: "pending", "in_transit", "delivered", "exception"
}

If a field cannot be determined, use null. Never omit fields from the schema."""
```

**Method 2: Assistant pre-filling (Claude-specific trick)**

Pre-fill the assistant turn to force a specific start to the response:

```python
def extract_with_prefill(document_text: str) -> dict:
    """
    Pre-fill the assistant turn with '{' to force JSON-only output.
    This is a Claude-specific technique that significantly reduces formatting noise.
    """
    import json
    
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=1024,
        temperature=0.0,
        system="Extract the invoice data. Return only valid JSON.",
        messages=[
            {"role": "user", "content": document_text},
            {"role": "assistant", "content": "{"}  # Pre-fill forces JSON start
        ]
    )
    
    # The response will be the continuation of the JSON (without the opening '{')
    raw = "{" + response.content[0].text
    
    # Clean up if model added trailing text
    # Find the end of the JSON object
    depth = 0
    end_idx = 0
    for i, char in enumerate(raw):
        if char == '{':
            depth += 1
        elif char == '}':
            depth -= 1
            if depth == 0:
                end_idx = i + 1
                break
    
    return json.loads(raw[:end_idx])
```

**Method 3: Pydantic + JSON schema validation (production approach)**

```python
from pydantic import BaseModel, Field, validator
from typing import Optional
import json

class Address(BaseModel):
    city: str
    country_code: str = Field(pattern=r'^[A-Z]{2}$')  # ISO 3166-1 alpha-2
    postal_code: Optional[str] = None
    
class CarrierInvoice(BaseModel):
    invoice_number: str
    invoice_date: str = Field(description="ISO 8601 date YYYY-MM-DD")
    carrier_name: str
    shipper_name: Optional[str] = None
    consignee_name: Optional[str] = None
    origin: Address
    destination: Address
    weight_kg: float = Field(gt=0)
    freight_charges_usd: float = Field(ge=0)
    currency: str = Field(pattern=r'^[A-Z]{3}$')  # ISO 4217
    
    @validator('invoice_date')
    def validate_date_format(cls, v):
        from datetime import datetime
        try:
            datetime.strptime(v, '%Y-%m-%d')
            return v
        except ValueError:
            raise ValueError(f"Date must be YYYY-MM-DD, got: {v}")

def extract_and_validate_invoice(invoice_text: str) -> CarrierInvoice:
    """
    Full extraction pipeline with validation.
    Returns a validated Pydantic model or raises on failure.
    """
    
    schema = CarrierInvoice.model_json_schema()
    
    system_prompt = f"""You are an invoice data extraction specialist.

Extract the invoice data and return ONLY valid JSON matching this schema:
{json.dumps(schema, indent=2)}

Rules:
- Return ONLY JSON, no other text
- country_code must be 2-letter ISO code (e.g., "US", "CN", "DE")
- invoice_date must be YYYY-MM-DD
- weight must be in kg (convert if needed: 1 lb = 0.453592 kg)
- freight_charges must be in USD (convert if needed)
- If a field is missing from the document, use null"""

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        temperature=0.0,
        system=system_prompt,
        messages=[
            {"role": "user", "content": f"<invoice>\n{invoice_text}\n</invoice>"}
        ]
    )
    
    raw_json = response.content[0].text.strip()
    
    # Remove code fences if model added them
    if raw_json.startswith("```"):
        raw_json = raw_json.split("```")[1]
        if raw_json.startswith("json"):
            raw_json = raw_json[4:]
    
    data = json.loads(raw_json)
    return CarrierInvoice.model_validate(data)
```

### 2.6 Role Prompting

Setting the AI's identity to improve output quality:

```python
DOMAIN_EXPERT_PROMPTS = {
    "logistics_analyst": """You are a senior logistics analyst with 15 years of experience 
    in international freight forwarding. You have deep expertise in:
    - Incoterms 2020 (EXW, FOB, CIF, DDP, etc.)
    - Customs procedures in US, EU, China, India
    - Carrier rate structures (LCL, FCL, air freight)
    - Supply chain risk assessment
    
    You give practical, specific advice. You cite specific rules and thresholds when relevant.
    You are direct and concise — no corporate jargon.""",
    
    "customs_broker": """You are a licensed customs broker with expertise in:
    - HS code classification (6-digit and 10-digit US HTS)
    - Duty rate calculations
    - Free Trade Agreement eligibility (USMCA, CPTPP, etc.)
    - ISF filing requirements
    - ADD/CVD (Anti-Dumping/Countervailing Duty) cases
    
    When classifying products, cite the relevant chapter headings and explain your reasoning.""",
    
    "warehouse_manager": """You are an experienced warehouse manager familiar with:
    - WMS systems (Manhattan, Blue Yonder, SAP EWM)
    - Inventory management (FIFO, FEFO, LIFO)
    - Slotting and warehouse layout optimization
    - Pick/pack/ship operations
    - KPIs: OTIF, order accuracy, inventory accuracy
    
    You speak in practical operational terms, not theory."""
}
```

### 2.7 XML Tags in Prompts (Claude-Specific)

Claude is specifically trained to recognize XML-like tags as semantic boundaries. This is one of the most powerful Claude-specific features and dramatically improves extraction reliability.

```python
def build_extraction_prompt(
    document_text: str,
    field_definitions: dict,
    examples: list[tuple[str, dict]]
) -> str:
    """
    Build a well-structured extraction prompt using XML tags.
    
    XML tags serve as clear semantic boundaries:
    - <document> wraps the input data
    - <instructions> wraps the task
    - <field_definitions> wraps the schema
    - <examples> wraps few-shot examples
    - <example> wraps individual examples
    """
    
    # Build field definitions section
    fields_xml = "\n".join(
        f'  <field name="{name}">{desc}</field>'
        for name, desc in field_definitions.items()
    )
    
    # Build examples section
    examples_xml = ""
    for i, (doc, expected) in enumerate(examples):
        import json
        examples_xml += f"""
  <example id="{i+1}">
    <input>{doc}</input>
    <output>{json.dumps(expected, indent=2)}</output>
  </example>"""
    
    prompt = f"""<instructions>
Extract structured data from the shipping document below.
Return ONLY a JSON object with the fields defined in <field_definitions>.
Use null for missing fields. Never invent data.
</instructions>

<field_definitions>
{fields_xml}
</field_definitions>

<examples>
{examples_xml}
</examples>

<document>
{document_text}
</document>"""
    
    return prompt

# Usage
field_defs = {
    "invoice_number": "Unique identifier for this invoice (string)",
    "total_usd": "Total amount due in US dollars (float)",
    "carrier": "Shipping carrier name (string)",
    "ship_date": "Shipment date in YYYY-MM-DD format (string)"
}

examples = [
    (
        "FedEx Invoice #FX-2024-001\nDate: Jan 15, 2024\nTotal: $127.50",
        {"invoice_number": "FX-2024-001", "total_usd": 127.50, "carrier": "FedEx", "ship_date": "2024-01-15"}
    )
]

document = "UPS Freight Bill\nBill#: UPS-789456\nShipped: March 3, 2024\nAmount Due: USD 543.21"

prompt = build_extraction_prompt(document, field_defs, examples)

response = client.messages.create(
    model="claude-haiku-4-5-20251001",
    max_tokens=512,
    temperature=0.0,
    system="You are a logistics document extraction specialist. Return only JSON.",
    messages=[{"role": "user", "content": prompt}]
)
```

**Why XML tags matter:**
1. They prevent the model from confusing your example data with the actual document
2. They create clear signal about what's instruction vs what's data
3. They enable precise extraction of content: `re.search(r'<output>(.*?)</output>', text)`
4. Claude's training specifically optimizes for this format

### 2.8 Negative Prompting

Tell the model what NOT to do. This sounds obvious but is often forgotten:

```python
NEGATIVE_PROMPTING_EXAMPLES = """
# WRONG: No negative guidance
"Extract the invoice amount."

# BETTER: Explicit negatives
"Extract the invoice amount as a float. 
Do NOT include currency symbols.
Do NOT include commas.
Do NOT include the word 'USD' or any currency code.
Only return the numeric value."

# WRONG: Vague instruction
"Be helpful and answer questions."

# BETTER: Scoped with negative
"Answer questions about shipment status and tracking only.
Do NOT answer questions about:
- Pricing or commercial terms
- Internal company information  
- Topics unrelated to the specific shipment
If asked about something outside your scope, respond: 'I can only help with shipment tracking questions.'"
"""

# Practical example — preventing hallucination
NO_HALLUCINATION_SUFFIX = """
IMPORTANT:
- Only use information explicitly stated in the document.
- Do NOT infer, guess, or assume values not present in the text.
- Do NOT use general knowledge to fill in missing fields.
- If a field is not explicitly in the document, use null.
- Do NOT say "typically" or "usually" — only what this specific document says."""
```

### 2.9 Prompt Templates — Separating Code from Prompts

In production, prompts should be managed separately from code. This is a software engineering practice that matters for iteration speed — you want to update prompts without code deploys.

**Option 1: F-strings (simple, works for small projects)**

```python
# prompts/invoice_extraction.py

SYSTEM_PROMPT = """You are a logistics invoice extraction specialist.
Extract the following fields: {field_list}
Return JSON matching this schema: {schema}"""

USER_PROMPT = """Extract data from this invoice:

<invoice>
{invoice_text}
</invoice>

Expected output format:
{format_example}"""

def make_extraction_prompt(
    invoice_text: str,
    field_list: list[str],
    schema: dict,
    format_example: dict
) -> tuple[str, str]:
    import json
    system = SYSTEM_PROMPT.format(
        field_list=", ".join(field_list),
        schema=json.dumps(schema, indent=2)
    )
    user = USER_PROMPT.format(
        invoice_text=invoice_text,
        format_example=json.dumps(format_example, indent=2)
    )
    return system, user
```

**Option 2: Jinja2 templates (production-grade)**

```python
# pip install jinja2
from jinja2 import Environment, FileSystemLoader, select_autoescape
import json

# templates/invoice_extraction.j2
JINJA_TEMPLATE = """
You are a logistics invoice extraction specialist.

Your task: Extract structured data from carrier invoices.

## Fields to Extract
{% for field in fields %}
- **{{ field.name }}** ({{ field.type }}): {{ field.description }}
  {% if field.required %}Required{% else %}Optional (use null if missing){% endif %}
{% endfor %}

## Rules
- Return ONLY valid JSON
- Do not include explanations
- Dates must be in YYYY-MM-DD format
{% if strict_mode %}
- STRICT MODE: Any ambiguity must result in null (do not guess)
{% endif %}

{% if examples %}
## Examples
{% for example in examples %}
Input:
{{ example.input }}

Output:
{{ example.output | tojson(indent=2) }}

---
{% endfor %}
{% endif %}
"""

def render_extraction_prompt(
    fields: list[dict],
    examples: list[dict] = None,
    strict_mode: bool = False
) -> str:
    env = Environment(autoescape=False)
    template = env.from_string(JINJA_TEMPLATE)
    return template.render(
        fields=fields,
        examples=examples or [],
        strict_mode=strict_mode
    )

# Usage
fields = [
    {"name": "invoice_number", "type": "string", "description": "Invoice ID", "required": True},
    {"name": "total_usd", "type": "float", "description": "Total in USD", "required": True},
    {"name": "carrier", "type": "string", "description": "Carrier name", "required": False}
]

prompt = render_extraction_prompt(fields, strict_mode=True)
```

**Option 3: Prompt management in a database (enterprise)**

```python
import sqlite3
from dataclasses import dataclass
from datetime import datetime

@dataclass
class PromptVersion:
    prompt_id: str
    version: int
    system_prompt: str
    user_template: str
    created_at: datetime
    eval_score: float  # From your eval runs (see 05-evals.md)
    is_active: bool

class PromptRegistry:
    """
    Production prompt management.
    Tracks versions, eval scores, A/B testing.
    """
    
    def __init__(self, db_path: str):
        self.conn = sqlite3.connect(db_path)
        self._create_tables()
    
    def _create_tables(self):
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS prompts (
                prompt_id TEXT,
                version INTEGER,
                system_prompt TEXT,
                user_template TEXT,
                created_at TEXT,
                eval_score REAL,
                is_active BOOLEAN,
                PRIMARY KEY (prompt_id, version)
            )
        """)
        self.conn.commit()
    
    def save_prompt(self, prompt_id: str, system_prompt: str, user_template: str) -> int:
        cursor = self.conn.execute(
            "SELECT MAX(version) FROM prompts WHERE prompt_id = ?",
            (prompt_id,)
        )
        max_version = cursor.fetchone()[0] or 0
        new_version = max_version + 1
        
        self.conn.execute(
            """INSERT INTO prompts VALUES (?, ?, ?, ?, ?, ?, ?)""",
            (prompt_id, new_version, system_prompt, user_template, 
             datetime.now().isoformat(), None, False)
        )
        self.conn.commit()
        return new_version
    
    def get_active_prompt(self, prompt_id: str) -> PromptVersion:
        cursor = self.conn.execute(
            "SELECT * FROM prompts WHERE prompt_id = ? AND is_active = 1",
            (prompt_id,)
        )
        row = cursor.fetchone()
        if not row:
            # Fall back to latest version
            cursor = self.conn.execute(
                "SELECT * FROM prompts WHERE prompt_id = ? ORDER BY version DESC LIMIT 1",
                (prompt_id,)
            )
            row = cursor.fetchone()
        
        if not row:
            raise ValueError(f"No prompt found for id: {prompt_id}")
        
        return PromptVersion(*row)
    
    def activate_prompt(self, prompt_id: str, version: int):
        self.conn.execute(
            "UPDATE prompts SET is_active = 0 WHERE prompt_id = ?",
            (prompt_id,)
        )
        self.conn.execute(
            "UPDATE prompts SET is_active = 1 WHERE prompt_id = ? AND version = ?",
            (prompt_id, version)
        )
        self.conn.commit()
```

---

## Part 3: Structured Outputs — Complete Pipeline

### 3.1 The Full Extraction Pipeline

Here is the complete, production-ready pipeline for extracting structured data from a logistics document:

```python
import anthropic
import json
import time
from pydantic import BaseModel, Field, validator
from typing import Optional
from enum import Enum

# ---- Data Models ----

class ServiceType(str, Enum):
    EXPRESS = "Express"
    GROUND = "Ground"
    OCEAN = "Ocean"
    AIR = "Air"
    UNKNOWN = "Unknown"

class Address(BaseModel):
    city: Optional[str] = None
    country_code: Optional[str] = None
    postal_code: Optional[str] = None
    
class CarrierInvoiceExtraction(BaseModel):
    invoice_number: Optional[str] = None
    invoice_date: Optional[str] = None  # YYYY-MM-DD
    carrier_name: Optional[str] = None
    shipper_name: Optional[str] = None
    consignee_name: Optional[str] = None
    origin: Optional[Address] = None
    destination: Optional[Address] = None
    weight_kg: Optional[float] = None
    freight_charges_usd: Optional[float] = None
    service_type: Optional[ServiceType] = None
    extraction_confidence: str = "high"  # high/medium/low
    extraction_notes: Optional[str] = None

# ---- Extraction Functions ----

EXTRACTION_SYSTEM_PROMPT = """You are a logistics document extraction specialist.

Extract data from carrier invoices and return ONLY a JSON object.

Field specifications:
- invoice_date: Must be YYYY-MM-DD format. If only month/year given, use first of month.
- weight_kg: Convert to kg if in other units (1 lb = 0.453592 kg, 1 oz = 0.0283495 kg)
- freight_charges_usd: Convert to USD using approximate rates if in other currency.
  Common rates: 1 EUR ≈ 1.08 USD, 1 GBP ≈ 1.27 USD, 1 CAD ≈ 0.74 USD
- service_type: Must be one of: "Express", "Ground", "Ocean", "Air", "Unknown"
- country_code: 2-letter ISO 3166-1 alpha-2 (US, GB, DE, CN, JP, etc.)
- extraction_confidence: "high" if all fields found, "medium" if some ambiguous, "low" if many missing
- extraction_notes: Explain any conversions made or ambiguities encountered

Return ONLY JSON. No explanations before or after."""

def extract_invoice(
    invoice_text: str,
    client: anthropic.Anthropic,
    model: str = "claude-haiku-4-5-20251001",
    max_retries: int = 2
) -> CarrierInvoiceExtraction:
    """
    Extract structured data from invoice text.
    
    Args:
        invoice_text: Raw invoice text
        client: Anthropic client
        model: Model to use (Haiku for cost, Sonnet for accuracy)
        max_retries: Number of retries with correction prompt on parse failure
        
    Returns:
        Validated CarrierInvoiceExtraction model
        
    Raises:
        ValueError: If extraction fails after all retries
    """
    
    schema = CarrierInvoiceExtraction.model_json_schema()
    
    messages = [
        {
            "role": "user",
            "content": f"""<invoice>
{invoice_text}
</invoice>

Extract the invoice data. Return JSON matching this schema:
{json.dumps(schema, indent=2)}"""
        }
    ]
    
    last_error = None
    
    for attempt in range(max_retries + 1):
        try:
            response = client.messages.create(
                model=model,
                max_tokens=1024,
                temperature=0.0,
                system=EXTRACTION_SYSTEM_PROMPT,
                messages=messages
            )
            
            raw_text = response.content[0].text.strip()
            
            # Clean JSON fences
            if "```json" in raw_text:
                raw_text = raw_text.split("```json")[1].split("```")[0].strip()
            elif "```" in raw_text:
                raw_text = raw_text.split("```")[1].split("```")[0].strip()
            
            data = json.loads(raw_text)
            return CarrierInvoiceExtraction.model_validate(data)
            
        except json.JSONDecodeError as e:
            last_error = e
            if attempt < max_retries:
                # Add correction turn to conversation
                messages.append({"role": "assistant", "content": raw_text})
                messages.append({
                    "role": "user",
                    "content": f"The JSON you returned was invalid: {e}. Please return ONLY valid JSON, nothing else."
                })
                time.sleep(0.5)
                
        except Exception as e:
            last_error = e
            if attempt < max_retries:
                messages.append({"role": "assistant", "content": response.content[0].text})
                messages.append({
                    "role": "user", 
                    "content": f"There was an error with your response: {e}. Please try again with valid JSON."
                })
                time.sleep(0.5)
    
    raise ValueError(f"Extraction failed after {max_retries + 1} attempts. Last error: {last_error}")


# ---- Test with a real invoice ----

def demo_extraction():
    client = anthropic.Anthropic()
    
    sample_invoice = """
    MAERSK LINE
    Ocean Freight Invoice
    
    Invoice Date: 15-Mar-2024
    Invoice No.: MAEU-INV-2024-045678
    
    Shipper: Shanghai Electronics Co., Ltd.
    Consignee: TechImport USA LLC
    
    Port of Loading: Shanghai, China (CNSHA)
    Port of Discharge: Los Angeles, USA (USLAX)
    
    Container: 40'HC
    Gross Weight: 18,250 KGS
    
    Ocean Freight (FCL 40HC): USD 2,850.00
    Bunker Surcharge (BAF): USD 450.00
    Port Handling (DTHC): USD 275.00
    Documentation Fee: USD 75.00
    Total Invoice: USD 3,650.00
    """
    
    result = extract_invoice(sample_invoice, client)
    
    print("Extraction Result:")
    print(f"  Invoice Number: {result.invoice_number}")
    print(f"  Date: {result.invoice_date}")
    print(f"  Carrier: {result.carrier_name}")
    print(f"  Origin: {result.origin.city}, {result.origin.country_code}")
    print(f"  Destination: {result.destination.city}, {result.destination.country_code}")
    print(f"  Weight: {result.weight_kg} kg")
    print(f"  Charges: ${result.freight_charges_usd}")
    print(f"  Service: {result.service_type}")
    print(f"  Confidence: {result.extraction_confidence}")
    if result.extraction_notes:
        print(f"  Notes: {result.extraction_notes}")
    
    return result

if __name__ == "__main__":
    demo_extraction()
```

---

## Part 4: Context Window Management

### 4.1 What the Context Window Is

The context window is the maximum amount of text (measured in tokens) that can be in a single API call. This includes:
- System prompt
- All messages (both user and assistant turns)
- The response being generated

**Claude's context windows:**
- Haiku: 200,000 tokens (~150,000 words)
- Sonnet: 200,000 tokens
- Opus: 200,000 tokens

200K tokens sounds huge. But it fills up fast with:
- Long documents (1-page = ~500 tokens, 100-page PDF = ~50,000 tokens)
- Long conversations (100 turns × 500 tokens/turn = 50,000 tokens)
- Few-shot examples (5 examples × 1,000 tokens each = 5,000 tokens)
- Retrieved context from RAG (10 chunks × 500 tokens = 5,000 tokens)

### 4.2 Counting Tokens

```python
import anthropic

client = anthropic.Anthropic()

# Count tokens BEFORE making the call — useful for cost estimation
# and deciding whether to truncate

def count_message_tokens(
    messages: list[dict],
    system: str = "",
    model: str = "claude-sonnet-4-6"
) -> dict:
    """
    Count tokens for a set of messages without actually calling the model.
    Returns input token count.
    """
    response = client.messages.count_tokens(
        model=model,
        system=system,
        messages=messages
    )
    return {
        "input_tokens": response.input_tokens,
        "estimated_cost_usd": estimate_cost(response.input_tokens, model=model)
    }

def estimate_cost(input_tokens: int, output_tokens: int = 0, model: str = "claude-sonnet-4-6") -> float:
    """
    Estimate API cost. Prices as of early 2025 — check Anthropic pricing page for updates.
    These are per-million-token prices.
    """
    PRICING = {
        "claude-haiku-4-5-20251001": {"input": 0.80, "output": 4.00},    # $/million tokens
        "claude-sonnet-4-6": {"input": 3.00, "output": 15.00},
        "claude-opus-4-5": {"input": 15.00, "output": 75.00},
    }
    
    if model not in PRICING:
        model = "claude-sonnet-4-6"  # Default
    
    prices = PRICING[model]
    cost = (input_tokens / 1_000_000) * prices["input"]
    cost += (output_tokens / 1_000_000) * prices["output"]
    return cost

# Example usage
messages = [
    {"role": "user", "content": "Here is a 50-page shipping manifest: " + ("x" * 25000)}
]

token_info = count_message_tokens(
    messages=messages,
    system=EXTRACTION_SYSTEM_PROMPT,
    model="claude-haiku-4-5-20251001"
)

print(f"Input tokens: {token_info['input_tokens']}")
print(f"Estimated cost: ${token_info['estimated_cost_usd']:.4f}")

# If too many tokens, we need truncation or chunking
MAX_INPUT_TOKENS = 150_000  # Leave headroom for output

if token_info['input_tokens'] > MAX_INPUT_TOKENS:
    print("WARNING: Context too long, need to chunk or truncate")
```

### 4.3 Truncation Strategies

When you have more content than fits in the context window:

```python
def truncate_document_for_context(
    document_text: str,
    system_prompt: str,
    max_total_tokens: int = 150_000,
    model: str = "claude-haiku-4-5-20251001"
) -> str:
    """
    Truncate a document to fit within context window.
    
    Strategies (in order of preference):
    1. If fits: return as-is
    2. If slightly over: remove whitespace, headers
    3. If very over: take first + last portions (preserve invoice header and totals)
    """
    
    # Check if it fits as-is
    test_messages = [{"role": "user", "content": document_text}]
    token_count = client.messages.count_tokens(
        model=model,
        system=system_prompt,
        messages=test_messages
    ).input_tokens
    
    if token_count <= max_total_tokens:
        return document_text
    
    # Strategy: preserve beginning and end (most important for invoices)
    # Invoices have header info at top, totals at bottom
    lines = document_text.split('\n')
    total_lines = len(lines)
    
    # Try progressively shorter versions
    for fraction in [0.9, 0.75, 0.6, 0.5]:
        keep_lines = int(total_lines * fraction)
        half = keep_lines // 2
        
        # Keep first half and last half (skip middle)
        truncated = (
            '\n'.join(lines[:half]) +
            '\n\n[... document truncated for context ...]\n\n' +
            '\n'.join(lines[-half:])
        )
        
        test_messages = [{"role": "user", "content": truncated}]
        token_count = client.messages.count_tokens(
            model=model,
            system=system_prompt,
            messages=test_messages
        ).input_tokens
        
        if token_count <= max_total_tokens:
            return truncated
    
    # Last resort: hard character truncation
    char_limit = int(len(document_text) * 0.4)
    return document_text[:char_limit] + "\n[TRUNCATED]"


def chunk_and_extract(
    long_document: str,
    extract_fn,
    chunk_size_tokens: int = 50_000
) -> list[dict]:
    """
    Process a document too large for a single context window by chunking.
    
    For documents like multi-page carrier manifests:
    1. Split into chunks
    2. Extract from each chunk
    3. Merge results
    """
    # Simple character-based chunking (approximate)
    # Rule of thumb: 1 token ≈ 4 characters for English text
    chars_per_chunk = chunk_size_tokens * 4
    
    chunks = []
    start = 0
    while start < len(long_document):
        end = start + chars_per_chunk
        
        # Don't split in the middle of a line
        if end < len(long_document):
            last_newline = long_document.rfind('\n', start, end)
            if last_newline > start:
                end = last_newline
        
        chunks.append(long_document[start:end])
        start = end
    
    # Extract from each chunk
    results = []
    for i, chunk in enumerate(chunks):
        print(f"Processing chunk {i+1}/{len(chunks)}...")
        try:
            result = extract_fn(chunk)
            results.append(result)
        except Exception as e:
            print(f"Chunk {i+1} failed: {e}")
            results.append(None)
    
    return results


def summarize_for_context(
    document_text: str,
    what_to_preserve: str,
    client: anthropic.Anthropic
) -> str:
    """
    Summarize a long document to fit in context.
    Better than truncation when the full document needs to be understood.
    """
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",  # Use fast/cheap model for summarization
        max_tokens=2000,
        system="Summarize the following document, preserving all specific details about: " + what_to_preserve,
        messages=[
            {"role": "user", "content": document_text}
        ]
    )
    return response.content[0].text
```

### 4.4 Managing Long Conversations

For chatbot-style applications where conversation history grows:

```python
class ConversationManager:
    """
    Manages a conversation with Claude, handling context window limits
    by summarizing old messages when the conversation gets too long.
    """
    
    def __init__(
        self,
        client: anthropic.Anthropic,
        model: str = "claude-sonnet-4-6",
        system_prompt: str = "",
        max_tokens: int = 150_000,
        summarize_threshold: int = 100_000
    ):
        self.client = client
        self.model = model
        self.system_prompt = system_prompt
        self.max_tokens = max_tokens
        self.summarize_threshold = summarize_threshold
        self.messages: list[dict] = []
        self.summary: str = ""
    
    def chat(self, user_message: str) -> str:
        """Send a message and get a response, managing context automatically."""
        
        self.messages.append({"role": "user", "content": user_message})
        
        # Check if we're approaching the context limit
        current_tokens = self.client.messages.count_tokens(
            model=self.model,
            system=self.system_prompt,
            messages=self.messages
        ).input_tokens
        
        if current_tokens > self.summarize_threshold:
            self._summarize_old_messages()
        
        # Include summary in system prompt if we have one
        effective_system = self.system_prompt
        if self.summary:
            effective_system += f"\n\n## Conversation Summary (Earlier Messages)\n{self.summary}"
        
        response = self.client.messages.create(
            model=self.model,
            max_tokens=2048,
            system=effective_system,
            messages=self.messages
        )
        
        assistant_message = response.content[0].text
        self.messages.append({"role": "assistant", "content": assistant_message})
        
        return assistant_message
    
    def _summarize_old_messages(self):
        """Summarize the oldest half of the conversation and drop those messages."""
        
        if len(self.messages) < 4:
            return  # Not enough messages to summarize
        
        # Summarize the first half
        messages_to_summarize = self.messages[:len(self.messages)//2]
        
        summary_response = self.client.messages.create(
            model="claude-haiku-4-5-20251001",  # Cheap model for summarization
            max_tokens=500,
            system="Create a concise bullet-point summary of this conversation, preserving key facts, decisions, and context.",
            messages=messages_to_summarize + [
                {"role": "user", "content": "Summarize the above conversation in bullet points."}
            ]
        )
        
        new_summary = summary_response.content[0].text
        
        if self.summary:
            self.summary = self.summary + "\n\n" + new_summary
        else:
            self.summary = new_summary
        
        # Keep only the recent half of messages
        self.messages = self.messages[len(self.messages)//2:]
        
        print(f"Summarized {len(messages_to_summarize)//2} turns. Summary: {self.summary[:100]}...")
```

---

## Common Failure Modes

### Failure Mode 1: Inconsistent JSON Output

**Symptom:** Sometimes the model returns JSON, sometimes it wraps it in code fences or adds explanation text.

**Diagnosis:**
```python
# Check what you're actually getting
print(repr(response.content[0].text[:200]))
# Might see: '```json\n{\n  "invoice_number": ...'
# Or: 'Here is the extracted data:\n\n{"invoice_number": ...'
```

**Fix:**
```python
def safe_parse_json(text: str) -> dict:
    """Parse JSON from LLM output, handling common formatting issues."""
    import re
    
    text = text.strip()
    
    # Remove code fences
    code_fence_match = re.match(r'^```(?:json)?\n?(.*?)\n?```$', text, re.DOTALL)
    if code_fence_match:
        text = code_fence_match.group(1).strip()
    
    # Remove common preambles
    preamble_patterns = [
        r'^Here is the (?:extracted )?(?:JSON )?(?:data|output|result):?\s*',
        r'^(?:The )?(?:JSON )?(?:data|output|result) is:?\s*',
        r'^Extracted data:?\s*',
    ]
    for pattern in preamble_patterns:
        text = re.sub(pattern, '', text, flags=re.IGNORECASE)
    
    # Remove trailing text after JSON
    brace_count = 0
    end_idx = len(text)
    for i, c in enumerate(text):
        if c == '{':
            brace_count += 1
        elif c == '}':
            brace_count -= 1
            if brace_count == 0:
                end_idx = i + 1
                break
    text = text[:end_idx]
    
    return json.loads(text)
```

### Failure Mode 2: Missing or Null Fields

**Symptom:** Model returns null for fields that are clearly in the document.

**Diagnosis:** Usually means either (1) the field definition is ambiguous, or (2) the document format is different from training examples.

**Fix:** Add more specific examples for that document format. Add explicit field definitions:
```python
# BAD: Vague definition
"invoice_number": "The invoice identifier"

# GOOD: Specific definition with examples
"invoice_number: The unique identifier for this specific invoice. Look for labels like 'Invoice #', 'Invoice No.', 'Bill Number', 'Reference Number', 'AWB#'. Example values: 'INV-2024-001', '1Z999AA1', 'MAEUBKK2345'"
```

### Failure Mode 3: Hallucinated Field Values

**Symptom:** Model invents values that aren't in the document.

**Diagnosis:** Usually happens when the model is "helpful" and fills in what it thinks should be there.

**Fix:** Explicit instruction + few-shot examples that show null:
```python
# Add to system prompt:
"""CRITICAL: Never invent data. If a field is not explicitly stated in the document, return null.
Do not infer, estimate, or guess. Use null.

Example of correct null handling:
Document: "INVOICE #FX-001\nTotal: $127.50"
Correct output: {"invoice_number": "FX-001", "carrier": null, "weight_kg": null, "total_usd": 127.50}
Incorrect output: {"invoice_number": "FX-001", "carrier": "FedEx", "weight_kg": 0, "total_usd": 127.50}
Note how "FedEx" was NOT in the document — do not infer it from context."""
```

### Failure Mode 4: Temperature Too High for Extraction

**Symptom:** Different runs of the same invoice produce different field values.

**Fix:** Always use `temperature=0.0` for extraction tasks. This is the #1 most common mistake made by engineers new to LLMs.

### Failure Mode 5: Context Too Short for max_tokens

**Symptom:** Response is cut off mid-JSON with stop_reason: "max_tokens".

**Diagnosis:**
```python
if response.stop_reason == "max_tokens":
    print("WARNING: Response truncated! Increase max_tokens.")
    print(f"Used {response.usage.output_tokens} tokens — already at limit")
```

**Fix:** Increase max_tokens. For extraction tasks, 1024 is usually enough. For complex analysis, use 4096.

---

## Interview Angles

### "Walk me through how you would build a system to extract data from carrier invoices."

**Great answer:**
"I'd start by understanding the input — what formats do the invoices come in? PDF, image, plain text? For PDFs, I'd use a PDF-to-text library (pdfplumber, PyMuPDF) or Claude's vision capabilities for scanned documents. Then I'd design a Pydantic model for the output schema — invoice number, date, carrier, origin, destination, weight, charges. 

For the prompt: system prompt that defines the extraction task and field specifications clearly, with the document wrapped in XML tags. I'd use few-shot examples covering the 3-4 most common formats we see. Temperature 0, Haiku model for cost unless accuracy requires Sonnet.

For validation: Pydantic catches type errors and format issues. For logical validation (e.g., dates are in expected ranges), I'd add custom validators. For failures, I'd retry once with a correction prompt that explains the error.

For monitoring: log every extraction with input tokens, output tokens, success/failure, and a confidence score. Run weekly evals against a golden dataset of 50 known-good invoices."

### "What's the difference between temperature=0 and temperature=1?"

**Great answer:**
"Temperature controls sampling randomness. At temperature 0, the model always picks the highest-probability next token — it's deterministic. At temperature 1, it samples proportionally from the full probability distribution. For extraction tasks like pulling invoice data, temperature 0 is always correct — you want the same input to always produce the same output. For creative tasks, higher temperature adds variety. In practice, for logistics/supply chain work, I'm almost always at 0 or 0.1 — these are factual tasks, not creative ones."

### "How do you handle LLM outputs that don't parse correctly?"

**Great answer:**
"Three-layer approach: First, defensive parsing — strip code fences, handle common preambles, use a regex to find the JSON boundaries. Second, retry with correction — if parsing still fails, add a turn to the conversation saying 'Your response wasn't valid JSON: [error]. Please return only valid JSON.' This works about 90% of the time. Third, escalate — after 2-3 retries, log the failure with the raw response for human review, and return a structured error to the calling system so it knows to flag this invoice for manual processing. Don't silently swallow failures."

---

## Practice Exercise

**Scenario:** A 3PL client receives 2,000 shipping manifests per month from 5 different carriers (FedEx, UPS, DHL, Maersk, and a regional carrier). Each manifest lists multiple shipments (5-50 per manifest). They currently manually key this data into their WMS. They want to automate this.

**Your task (build and run this):**

1. **Design the data model**: Define a Pydantic model for a single shipment entry from a manifest. Include: tracking_number, carrier, origin_city, origin_country, destination_city, destination_country, weight_kg, service_level, estimated_delivery_date.

2. **Write the extraction prompt**: Write a system prompt and user prompt template that extracts ALL shipments from a manifest (it should return a JSON array of shipment objects).

3. **Build the extraction pipeline**:
   - Accept a manifest as a text string
   - Call Claude (use Haiku)
   - Parse and validate the output
   - Handle failures with retry

4. **Test with at least 3 different manifest formats** (simulate different carriers with different column names and date formats).

5. **Add cost tracking**: After each extraction, log the input tokens, output tokens, and estimated cost.

6. **Answer**: For 2,000 manifests/month averaging 15 shipments per manifest and 200 tokens input per manifest, what's the monthly cost with Haiku? Would you recommend Haiku or Sonnet for this task?

**Starter data** (create test manifests):
```
# Manifest 1 - FedEx style
FedEx Shipment Manifest
Date: 2024-03-15
Account: 123456789

Tracking#         | Dest City    | Dest Country | Weight | Service   | Est. Del
772612345678      | New York     | US           | 2.5 kg | Overnight | 2024-03-16
772612345679      | Los Angeles  | US           | 1.2 kg | 2-Day     | 2024-03-17

# Manifest 2 - DHL style  
DHL EXPRESS MANIFEST
Shipment Date: 15-MAR-24
Route: DELHUB-USNYC

HAWB           CONSIGNEE CITY  CTY   GW(KG)  PRD  ETA
1234567890     CHICAGO         US    5.500   EXP  16MAR24
1234567891     MIAMI           US    3.200   DOX  17MAR24
```

The goal: a single function that handles both formats and returns clean, consistent Pydantic models.

---

*Next: [02-tool-calling.md](./02-tool-calling.md) — Give your LLM the ability to take actions in your logistics systems.*
