# 02 — Tool Calling (Function Calling)
## FDE Course: AI/LLM Fundamentals for Logistics/Manufacturing/Supply Chain

**Prerequisites:** [01-llm-api-prompt-engineering.md](./01-llm-api-prompt-engineering.md)  
**Related files:**  
- [04-agents-orchestration.md](./04-agents-orchestration.md) — building agents with tool calling  
- [07-guardrails-cost-tradeoffs.md](./07-guardrails-cost-tradeoffs.md) — cost of tool-calling loops

---

## Why Tool Calling Is Your Most Important Skill

Of all the LLM capabilities, tool calling (also called function calling) is the one that makes AI actually useful in enterprise logistics software. Here's why:

Without tool calling, LLMs are read-only: they can analyze text and generate text, but they can't interact with the real world. Your customer's question "What's the status of shipment B/L MAEU123456?" requires looking up a real database. Your customer's request "Book a pickup for tomorrow at 9am" requires actually calling the carrier API. Tool calling is the bridge between "the AI says something" and "the AI does something."

As an FDE, you will demo tool-calling agents to customers. You will build them for POCs. You will be asked in interviews to explain how they work at a deep level. This is your technical centerpiece.

---

## Part 1: What Tool Calling Is (Mechanically)

### 1.1 The Core Mechanism

Tool calling is a structured protocol for an LLM to request the execution of external code. The flow is:

```
1. YOU define tools (name, description, JSON Schema for inputs)
2. YOU send user message + tool definitions to Claude
3. CLAUDE decides: "I need to call get_shipment_status to answer this"
4. CLAUDE returns a tool_use content block (NOT text) with: tool name + arguments
5. YOUR CODE executes get_shipment_status(arguments)
6. YOUR CODE returns the result back to Claude in a tool_result block
7. CLAUDE uses the result to formulate its final answer
```

The key insight: **the LLM never directly calls anything**. It just returns structured JSON saying "call this function with these arguments." Your code does the actual execution. The LLM is doing two things: (1) deciding which tool to call and (2) extracting the right arguments from natural language.

### 1.2 Why This Matters for Logistics

The tool calling capability directly maps to logistics use cases:

```python
# Every one of these is a tool call
tools_in_logistics = {
    "Carrier APIs":           ["get_shipment_tracking", "get_transit_time", "create_booking"],
    "WMS Integration":        ["get_inventory_level", "create_pick_order", "confirm_receipt"],
    "Customs/Compliance":     ["classify_hs_code", "get_duty_rate", "validate_dangerous_goods"],
    "Rate Shopping":          ["get_spot_rates", "compare_carrier_rates", "get_fuel_surcharge"],
    "Address Validation":     ["validate_delivery_address", "get_service_area"],
    "Document Processing":    ["generate_commercial_invoice", "create_packing_list", "submit_customs_declaration"],
    "ERP Integration":        ["create_purchase_order", "update_delivery_status", "post_goods_receipt"],
}
```

When you demo to a customer: "You can ask the AI in plain English to book a shipment, and it will validate the address, check rates, select the best carrier, and create the booking — all without the user touching 5 different screens." That's tool calling.

---

## Part 2: Claude Tool Calling — Deep Dive

### 2.1 Tool Definition Structure

A tool is defined by three things:

```python
import anthropic

tool_definition = {
    "name": "get_shipment_status",           # Snake_case verb_noun
    "description": """Look up the current status of a shipment by tracking number.
    
    Use this tool when the user asks about:
    - Where is my shipment?
    - Has my package been delivered?
    - What's the estimated delivery date?
    - What happened to shipment [number]?
    
    This tool returns current status, location, and estimated delivery. 
    It does NOT return historical tracking events — use get_tracking_history for that.
    Do NOT use this tool for rate quotes or booking — use get_rate_quote or create_booking instead.""",
    
    "input_schema": {
        "type": "object",
        "properties": {
            "tracking_number": {
                "type": "string",
                "description": "The carrier tracking number. Formats vary by carrier: FedEx (12 digits), UPS (1Z followed by 16 chars), DHL (10-11 digits), USPS (22 digits starting with 94). Maersk uses B/L numbers in format MAEUXXX followed by 7 digits."
            },
            "carrier": {
                "type": "string",
                "enum": ["fedex", "ups", "dhl", "usps", "maersk", "msc", "hapag-lloyd", "unknown"],
                "description": "The shipping carrier. If unknown, pass 'unknown' and the system will attempt auto-detection."
            }
        },
        "required": ["tracking_number"]
    }
}
```

**Critical design decisions:**

1. **The description is for the LLM, not for you.** Claude reads this to decide when to call the tool. Be explicit about when to use it AND when NOT to use it.

2. **Input schema is JSON Schema.** The LLM uses this schema to generate valid arguments. Provide descriptions for every field — they guide argument extraction.

3. **Use `enum` for constrained values.** If a field can only be a few values, enumerate them. This prevents the LLM from passing "Federal Express" when your code expects "fedex".

4. **`required` list matters.** Only put fields in required that truly can't proceed without them. Optional fields should have clear descriptions of when they're used.

### 2.2 The Full Tool Calling Loop — Step by Step

```python
import anthropic
import json
from typing import Any

client = anthropic.Anthropic()

# ---- Tool Implementations ----
# These are your actual business logic functions
# In production, these call real APIs, databases, etc.

def get_shipment_status(tracking_number: str, carrier: str = "unknown") -> dict:
    """Mock implementation — replace with real carrier API calls."""
    # Simulate carrier API lookup
    mock_data = {
        "1Z999AA10123456784": {
            "status": "in_transit",
            "current_location": "Memphis, TN",
            "estimated_delivery": "2024-03-18",
            "carrier": "UPS",
            "service": "Ground",
            "last_update": "2024-03-16T14:23:00Z"
        },
        "772612345678": {
            "status": "out_for_delivery",
            "current_location": "San Francisco, CA",
            "estimated_delivery": "2024-03-16",
            "carrier": "FedEx",
            "service": "Priority Overnight",
            "last_update": "2024-03-16T08:45:00Z"
        }
    }
    
    if tracking_number in mock_data:
        return {"success": True, "data": mock_data[tracking_number]}
    else:
        return {"success": False, "error": f"Tracking number {tracking_number} not found. Please verify the number is correct."}

def calculate_shipping_rate(
    origin_zip: str,
    destination_zip: str,
    weight_lbs: float,
    length_in: float,
    width_in: float, 
    height_in: float,
    service_level: str = "ground"
) -> dict:
    """Calculate shipping rate. Mock — replace with carrier API."""
    # Simulate dimensional weight calculation
    dim_weight = (length_in * width_in * height_in) / 139  # FedEx/UPS divisor
    billable_weight = max(weight_lbs, dim_weight)
    
    base_rates = {"ground": 0.85, "2day": 1.45, "overnight": 2.20}
    base = base_rates.get(service_level, 0.85)
    
    return {
        "success": True,
        "data": {
            "billable_weight_lbs": round(billable_weight, 2),
            "dimensional_weight_lbs": round(dim_weight, 2),
            "actual_weight_lbs": weight_lbs,
            "service_level": service_level,
            "estimated_cost_usd": round(billable_weight * base * 4.50 + 7.50, 2),
            "transit_days": {"ground": 3, "2day": 2, "overnight": 1}.get(service_level, 3),
            "currency": "USD"
        }
    }

def validate_address(
    street: str,
    city: str,
    state: str,
    postal_code: str,
    country: str = "US"
) -> dict:
    """Validate a shipping address. Mock — replace with address validation API."""
    # Simple mock validation
    if not postal_code or len(postal_code) < 5:
        return {
            "success": False,
            "valid": False,
            "error": "Invalid postal code format",
            "suggestions": []
        }
    
    return {
        "success": True,
        "valid": True,
        "standardized": {
            "street": street.upper(),
            "city": city.upper(),
            "state": state.upper(),
            "postal_code": postal_code[:5],
            "country": country
        },
        "residential": True,
        "deliverable": True
    }

def create_shipment_booking(
    origin_address: dict,
    destination_address: dict,
    weight_lbs: float,
    service_level: str,
    declared_value_usd: float = 0.0
) -> dict:
    """Create a shipment booking. Mock — would call carrier booking API."""
    import random
    import string
    
    confirmation_number = ''.join(random.choices(string.ascii_uppercase + string.digits, k=12))
    tracking_number = '1Z' + ''.join(random.choices(string.digits, k=16))
    
    return {
        "success": True,
        "data": {
            "confirmation_number": confirmation_number,
            "tracking_number": tracking_number,
            "carrier": "UPS",
            "service_level": service_level,
            "pickup_date": "2024-03-18",
            "estimated_delivery": "2024-03-21",
            "label_url": f"https://shipping.example.com/labels/{confirmation_number}.pdf",
            "cost_usd": round(weight_lbs * 1.25 + 8.50, 2)
        }
    }

# ---- Tool Registry ----

TOOL_DEFINITIONS = [
    {
        "name": "get_shipment_status",
        "description": """Look up the current status and location of a shipment.
        
        Use when user asks: where is my package, delivery status, tracking update.
        Returns: status (pending/in_transit/out_for_delivery/delivered/exception), 
                 current location, estimated delivery date.
        Do NOT use for rate quotes or creating new shipments.""",
        "input_schema": {
            "type": "object",
            "properties": {
                "tracking_number": {
                    "type": "string",
                    "description": "The tracking number from the carrier. Remove spaces."
                },
                "carrier": {
                    "type": "string",
                    "enum": ["fedex", "ups", "dhl", "usps", "unknown"],
                    "description": "The carrier. Use 'unknown' if not specified by user."
                }
            },
            "required": ["tracking_number"]
        }
    },
    {
        "name": "calculate_shipping_rate",
        "description": """Calculate shipping cost and transit time for a package.
        
        Use when user asks: how much will it cost to ship, shipping rate, quote.
        Requires package dimensions and weight plus origin/destination zip codes.
        Returns: cost estimate, transit days, billable weight.""",
        "input_schema": {
            "type": "object",
            "properties": {
                "origin_zip": {"type": "string", "description": "5-digit origin ZIP code"},
                "destination_zip": {"type": "string", "description": "5-digit destination ZIP code"},
                "weight_lbs": {"type": "number", "description": "Package weight in pounds"},
                "length_in": {"type": "number", "description": "Package length in inches"},
                "width_in": {"type": "number", "description": "Package width in inches"},
                "height_in": {"type": "number", "description": "Package height in inches"},
                "service_level": {
                    "type": "string",
                    "enum": ["ground", "2day", "overnight"],
                    "description": "Shipping service level. Default to 'ground' if not specified."
                }
            },
            "required": ["origin_zip", "destination_zip", "weight_lbs", "length_in", "width_in", "height_in"]
        }
    },
    {
        "name": "validate_address",
        "description": """Validate and standardize a shipping address.
        
        Always validate addresses before creating a shipment booking.
        Returns: whether address is valid and deliverable, standardized format.""",
        "input_schema": {
            "type": "object",
            "properties": {
                "street": {"type": "string", "description": "Street address including number"},
                "city": {"type": "string"},
                "state": {"type": "string", "description": "2-letter state abbreviation for US"},
                "postal_code": {"type": "string"},
                "country": {"type": "string", "description": "ISO 3166-1 alpha-2 country code. Default: US"}
            },
            "required": ["street", "city", "state", "postal_code"]
        }
    },
    {
        "name": "create_shipment_booking",
        "description": """Create an actual shipment booking and generate a shipping label.
        
        Use ONLY after:
        1. Destination address has been validated with validate_address
        2. User has confirmed the rate from calculate_shipping_rate
        3. User explicitly requests to proceed with booking
        
        This creates a real shipment and generates a label. It has real cost implications.
        ALWAYS confirm with the user before calling this tool.""",
        "input_schema": {
            "type": "object",
            "properties": {
                "origin_address": {
                    "type": "object",
                    "description": "Origin address object",
                    "properties": {
                        "street": {"type": "string"},
                        "city": {"type": "string"},
                        "state": {"type": "string"},
                        "postal_code": {"type": "string"},
                        "country": {"type": "string"}
                    },
                    "required": ["street", "city", "state", "postal_code"]
                },
                "destination_address": {
                    "type": "object",
                    "description": "Destination address object",
                    "properties": {
                        "street": {"type": "string"},
                        "city": {"type": "string"},
                        "state": {"type": "string"},
                        "postal_code": {"type": "string"},
                        "country": {"type": "string"}
                    },
                    "required": ["street", "city", "state", "postal_code"]
                },
                "weight_lbs": {"type": "number"},
                "service_level": {"type": "string", "enum": ["ground", "2day", "overnight"]},
                "declared_value_usd": {"type": "number", "description": "Declared value for insurance purposes. 0 if none."}
            },
            "required": ["origin_address", "destination_address", "weight_lbs", "service_level"]
        }
    }
]

# ---- Tool Dispatcher ----

TOOL_FUNCTIONS = {
    "get_shipment_status": get_shipment_status,
    "calculate_shipping_rate": calculate_shipping_rate,
    "validate_address": validate_address,
    "create_shipment_booking": create_shipment_booking
}

def execute_tool(tool_name: str, tool_input: dict) -> Any:
    """
    Execute a tool by name with the given input.
    Returns the result (will be serialized to JSON for the LLM).
    """
    if tool_name not in TOOL_FUNCTIONS:
        return {"success": False, "error": f"Unknown tool: {tool_name}. Available tools: {list(TOOL_FUNCTIONS.keys())}"}
    
    func = TOOL_FUNCTIONS[tool_name]
    
    try:
        result = func(**tool_input)
        return result
    except TypeError as e:
        # Missing required argument or wrong type
        return {
            "success": False, 
            "error": f"Invalid tool arguments: {e}. Check required fields and types."
        }
    except Exception as e:
        return {
            "success": False,
            "error": f"Tool execution failed: {str(e)}"
        }
```

### 2.3 The Core Tool Calling Loop

```python
def run_tool_calling_loop(
    user_message: str,
    system_prompt: str = "",
    tools: list = None,
    model: str = "claude-sonnet-4-6",
    max_iterations: int = 10,
    verbose: bool = True
) -> str:
    """
    The core agentic loop for tool calling.
    
    Continues calling Claude until stop_reason is "end_turn" 
    (meaning Claude has finished and doesn't need more tool calls).
    
    Args:
        user_message: The user's natural language request
        system_prompt: System prompt for Claude
        tools: List of tool definitions
        model: Claude model to use
        max_iterations: Safety limit on iterations
        verbose: Whether to print tool calls and results
        
    Returns:
        Claude's final text response
    """
    
    if tools is None:
        tools = TOOL_DEFINITIONS
    
    messages = [{"role": "user", "content": user_message}]
    iteration = 0
    
    while iteration < max_iterations:
        iteration += 1
        
        if verbose:
            print(f"\n--- Iteration {iteration} ---")
        
        # Call Claude
        response = client.messages.create(
            model=model,
            max_tokens=4096,
            system=system_prompt,
            tools=tools,
            messages=messages
        )
        
        if verbose:
            print(f"Stop reason: {response.stop_reason}")
        
        # Add Claude's response to message history
        messages.append({
            "role": "assistant",
            "content": response.content  # Note: content is a list of blocks
        })
        
        # Check stop reason
        if response.stop_reason == "end_turn":
            # Claude is done — extract the final text response
            for block in response.content:
                if hasattr(block, 'text'):
                    return block.text
            return ""  # No text in response (unusual)
        
        elif response.stop_reason == "tool_use":
            # Claude wants to call one or more tools
            tool_results = []
            
            for block in response.content:
                if block.type == "tool_use":
                    tool_name = block.name
                    tool_input = block.input
                    tool_use_id = block.id
                    
                    if verbose:
                        print(f"Tool call: {tool_name}({json.dumps(tool_input, indent=2)})")
                    
                    # Execute the tool
                    result = execute_tool(tool_name, tool_input)
                    
                    if verbose:
                        print(f"Tool result: {json.dumps(result, indent=2)[:200]}...")
                    
                    # Build tool_result content block
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": tool_use_id,
                        "content": json.dumps(result)
                    })
            
            # Add tool results to message history
            # Tool results go in a USER turn
            messages.append({
                "role": "user",
                "content": tool_results
            })
        
        elif response.stop_reason == "max_tokens":
            return "Error: Response exceeded maximum token limit. Try a simpler request."
        
        else:
            return f"Unexpected stop reason: {response.stop_reason}"
    
    return f"Error: Maximum iterations ({max_iterations}) reached without completing the task."
```

### 2.4 Understanding the Message Structure During Tool Calling

This is the part most engineers get wrong. Let me show the exact message array at each step:

```python
# STEP 1: Initial user message
messages_step1 = [
    {"role": "user", "content": "What's the status of tracking number 1Z999AA10123456784?"}
]

# STEP 2: After Claude's tool_use response
# Claude returns: content=[TextBlock(text="Let me check..."), ToolUseBlock(name="get_shipment_status", ...)]
messages_step2 = [
    {"role": "user", "content": "What's the status of tracking number 1Z999AA10123456784?"},
    {
        "role": "assistant",
        "content": [
            # Sometimes Claude includes text before the tool call
            {"type": "text", "text": "Let me check the status of that shipment."},
            # The actual tool call
            {
                "type": "tool_use",
                "id": "toolu_01A2B3C4D5E6F7G8H9I0J",
                "name": "get_shipment_status",
                "input": {
                    "tracking_number": "1Z999AA10123456784",
                    "carrier": "ups"
                }
            }
        ]
    }
]

# STEP 3: Add tool result — this goes in a USER turn
messages_step3 = [
    {"role": "user", "content": "What's the status of tracking number 1Z999AA10123456784?"},
    {
        "role": "assistant",
        "content": [
            {"type": "text", "text": "Let me check the status of that shipment."},
            {
                "type": "tool_use",
                "id": "toolu_01A2B3C4D5E6F7G8H9I0J",
                "name": "get_shipment_status",
                "input": {"tracking_number": "1Z999AA10123456784", "carrier": "ups"}
            }
        ]
    },
    {
        "role": "user",
        "content": [
            {
                "type": "tool_result",
                "tool_use_id": "toolu_01A2B3C4D5E6F7G8H9I0J",  # Must match the tool_use id
                "content": json.dumps({
                    "success": True,
                    "data": {
                        "status": "in_transit",
                        "current_location": "Memphis, TN",
                        "estimated_delivery": "2024-03-18"
                    }
                })
            }
        ]
    }
    # Claude will now respond with its final answer (stop_reason: "end_turn")
]
```

**The critical rule:** Tool results ALWAYS go in a USER turn. Even though you (the developer) are providing the results, they must be in the `user` role. The conversation structure is:
```
user (question)
→ assistant (tool_use)
→ user (tool_result)
→ assistant (final answer)
```

### 2.5 Handling Tool Errors

What happens when a tool fails? Return a structured error:

```python
def execute_tool_safe(tool_name: str, tool_input: dict) -> str:
    """
    Execute a tool and return JSON-serializable result.
    Always succeeds (returns error JSON if tool fails) so the LLM can handle it.
    """
    try:
        result = execute_tool(tool_name, tool_input)
        return json.dumps(result)
    except Exception as e:
        # Return error as JSON — LLM will see this and can tell user what happened
        error_result = {
            "success": False,
            "error": str(e),
            "tool": tool_name,
            "suggestion": "You may want to ask the user to provide different input or try again."
        }
        return json.dumps(error_result)

# You can also use is_error in the tool_result block
# This signals to Claude that something went wrong
tool_result_on_error = {
    "type": "tool_result",
    "tool_use_id": tool_use_id,
    "content": "Address validation API is temporarily unavailable",
    "is_error": True  # Claude will know this is an error, not a successful result
}
```

### 2.6 Parallel Tool Calling

When Claude needs multiple pieces of information that don't depend on each other, it may return multiple tool_use blocks in a single response. You should execute these in parallel:

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

def run_parallel_tool_calls(tool_use_blocks: list) -> list[dict]:
    """
    Execute multiple tool calls in parallel.
    Returns list of tool_result dicts in same order as input.
    """
    
    def execute_one(block):
        result = execute_tool_safe(block.name, block.input)
        return {
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": result
        }
    
    with ThreadPoolExecutor(max_workers=len(tool_use_blocks)) as executor:
        results = list(executor.map(execute_one, tool_use_blocks))
    
    return results

# In the tool calling loop:
def run_tool_calling_loop_parallel(user_message: str) -> str:
    """Enhanced loop with parallel tool execution."""
    
    messages = [{"role": "user", "content": user_message}]
    
    for _ in range(10):  # max iterations
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=4096,
            tools=TOOL_DEFINITIONS,
            messages=messages
        )
        
        messages.append({"role": "assistant", "content": response.content})
        
        if response.stop_reason == "end_turn":
            return next((b.text for b in response.content if hasattr(b, 'text')), "")
        
        if response.stop_reason == "tool_use":
            tool_use_blocks = [b for b in response.content if b.type == "tool_use"]
            
            print(f"Executing {len(tool_use_blocks)} tool(s) in parallel: {[b.name for b in tool_use_blocks]}")
            
            # Execute all tool calls in parallel
            tool_results = run_parallel_tool_calls(tool_use_blocks)
            
            messages.append({"role": "user", "content": tool_results})
    
    return "Max iterations reached"
```

---

## Part 3: Tool Design Best Practices

### 3.1 Tool Naming Conventions

```python
# GOOD: Verb_noun, clear action
"get_shipment_status"        # Read operation
"create_carrier_booking"     # Create operation
"update_delivery_address"    # Update operation
"cancel_shipment"            # Delete/cancel operation
"calculate_duty_rate"        # Computation
"validate_hs_code"           # Validation

# BAD: Vague, ambiguous
"shipment"                   # What does this do?
"process"                    # Process what?
"handleData"                 # camelCase, no verb
"do_thing"                   # Meaningless
"api_call"                   # Meta, not descriptive
```

### 3.2 Writing Tool Descriptions That Work

The description is what the LLM reads to decide whether to call your tool. This is the highest-leverage place to invest effort.

```python
# BAD DESCRIPTION: Too short, no guidance on when to use
{
    "name": "get_rates",
    "description": "Get shipping rates.",
    ...
}

# GOOD DESCRIPTION: When to use, what it returns, when NOT to use, examples
{
    "name": "get_carrier_rates",
    "description": """Get real-time shipping rate quotes from multiple carriers.

    USE THIS TOOL WHEN:
    - User asks "how much will it cost to ship..."
    - User wants to compare carrier options
    - User asks for a rate quote or shipping estimate
    - User wants to know transit time and cost together
    
    DO NOT USE THIS TOOL WHEN:
    - User is asking about a shipment that's already booked (use get_shipment_status)
    - User wants to create a booking (this tool is quotes only, use create_booking after confirming rate)
    - You don't have weight and dimensions — ask user first
    
    RETURNS:
    - List of carrier options with: carrier name, service level, cost, transit days
    - Sorted by price (cheapest first)
    - Does NOT create any booking or commitment
    
    EXAMPLE TRIGGER PHRASES:
    "ship a 10 lb box", "how much to ship from NYC to LA", "cheapest way to send this"
    """,
    ...
}
```

### 3.3 Input Schema Design

```python
# WRONG: No descriptions, missing required
{
    "type": "object",
    "properties": {
        "id": {"type": "string"},
        "type": {"type": "string"},
        "value": {"type": "number"}
    }
}

# RIGHT: Full descriptions, proper required, enums where applicable, examples
{
    "type": "object",
    "properties": {
        "booking_reference": {
            "type": "string",
            "description": "Unique booking reference number. Format: carrier-specific. FedEx: 12 digits. UPS: 1Z prefix + 16 chars. Example: '1Z999AA10123456784'"
        },
        "action": {
            "type": "string",
            "enum": ["cancel", "modify", "reschedule"],
            "description": "Action to take on the booking. 'cancel' permanently cancels. 'modify' changes details. 'reschedule' changes pickup date."
        },
        "modification_details": {
            "type": "object",
            "description": "Required when action is 'modify' or 'reschedule'. Can include: new_pickup_date (YYYY-MM-DD), new_address (object), new_weight_lbs (number).",
            "properties": {
                "new_pickup_date": {"type": "string", "description": "ISO 8601 date YYYY-MM-DD"},
                "new_weight_lbs": {"type": "number", "description": "Updated package weight in pounds"},
            }
        },
        "reason": {
            "type": "string",
            "description": "Reason for modification. Required for audit trail. Examples: 'customer request', 'address correction', 'weight discrepancy'"
        }
    },
    "required": ["booking_reference", "action", "reason"]
}
```

### 3.4 Making Tools Return Useful Error Information

Your tool functions should return errors that the LLM can understand and relay to users:

```python
def get_shipment_status_production(tracking_number: str, carrier: str = "unknown") -> dict:
    """
    Production-quality tool with proper error handling.
    """
    import re
    
    # Input validation with helpful error messages
    tracking_number = tracking_number.strip().replace(" ", "")
    
    if len(tracking_number) < 5:
        return {
            "success": False,
            "error_code": "INVALID_TRACKING_NUMBER",
            "error": "Tracking number is too short. Please provide the complete tracking number.",
            "user_message": "I couldn't look up that tracking number — it appears to be incomplete. Could you double-check and provide the full tracking number?"
        }
    
    # Carrier detection if unknown
    if carrier == "unknown":
        if re.match(r'^1Z[A-Z0-9]{16}$', tracking_number):
            carrier = "ups"
        elif re.match(r'^\d{12}$', tracking_number):
            carrier = "fedex"
        elif re.match(r'^\d{10,11}$', tracking_number):
            carrier = "dhl"
        elif re.match(r'^94\d{20}$', tracking_number):
            carrier = "usps"
    
    try:
        # In production: call carrier API here
        # result = carrier_api.track(tracking_number)
        result = {"status": "in_transit", "location": "Chicago, IL", "eta": "2024-03-18"}
        
        return {
            "success": True,
            "tracking_number": tracking_number,
            "carrier": carrier,
            "data": result
        }
    
    except TimeoutError:
        return {
            "success": False,
            "error_code": "CARRIER_TIMEOUT",
            "error": f"The {carrier.upper()} tracking API timed out.",
            "user_message": f"The {carrier.upper()} tracking system is not responding right now. Please try again in a few minutes or visit {carrier}.com/tracking directly.",
            "retry_suggested": True
        }
    
    except Exception as e:
        return {
            "success": False,
            "error_code": "UNKNOWN_ERROR",
            "error": str(e),
            "user_message": "There was an unexpected error checking this shipment. Our team has been notified."
        }
```

---

## Part 4: Complete Logistics Agent

Here is the full demo-quality logistics agent that ties everything together:

```python
import anthropic
import json
from datetime import datetime, timedelta
from typing import Optional
import random
import string

client = anthropic.Anthropic()

# ---- Full Tool Implementations ----

def get_shipment_tracking(tracking_number: str) -> dict:
    mock_shipments = {
        "DEMO123": {
            "status": "in_transit",
            "carrier": "FedEx",
            "service": "FedEx Ground",
            "origin": "Los Angeles, CA",
            "destination": "New York, NY",
            "current_location": "Dallas, TX",
            "estimated_delivery": (datetime.now() + timedelta(days=2)).strftime("%Y-%m-%d"),
            "last_update": datetime.now().strftime("%Y-%m-%dT%H:%M:%SZ"),
            "events": [
                {"time": "2024-03-14 08:00", "location": "Los Angeles, CA", "status": "Picked Up"},
                {"time": "2024-03-15 02:00", "location": "Phoenix, AZ", "status": "In Transit"},
                {"time": "2024-03-16 06:00", "location": "Dallas, TX", "status": "Arrived at Hub"},
            ]
        }
    }
    
    if tracking_number.upper() in mock_shipments:
        return {"success": True, "data": mock_shipments[tracking_number.upper()]}
    return {
        "success": False,
        "error": f"No shipment found with tracking number {tracking_number}. Please verify the number.",
        "suggestion": "Check that you have the complete tracking number. FedEx numbers are 12-15 digits."
    }

def get_carrier_rates(
    origin_zip: str,
    destination_zip: str,
    weight_lbs: float,
    length_in: float = 12.0,
    width_in: float = 12.0,
    height_in: float = 12.0
) -> dict:
    dim_weight = (length_in * width_in * height_in) / 139
    billable = max(weight_lbs, dim_weight)
    
    rates = [
        {
            "carrier": "UPS",
            "service": "Ground",
            "transit_days": 4,
            "cost_usd": round(billable * 0.85 * 4.25 + 8.50, 2),
            "delivery_date": (datetime.now() + timedelta(days=4)).strftime("%Y-%m-%d")
        },
        {
            "carrier": "FedEx",
            "service": "Ground",
            "transit_days": 3,
            "cost_usd": round(billable * 0.88 * 4.25 + 9.00, 2),
            "delivery_date": (datetime.now() + timedelta(days=3)).strftime("%Y-%m-%d")
        },
        {
            "carrier": "UPS",
            "service": "2nd Day Air",
            "transit_days": 2,
            "cost_usd": round(billable * 1.45 * 4.25 + 15.00, 2),
            "delivery_date": (datetime.now() + timedelta(days=2)).strftime("%Y-%m-%d")
        },
        {
            "carrier": "FedEx",
            "service": "Priority Overnight",
            "transit_days": 1,
            "cost_usd": round(billable * 2.20 * 4.25 + 25.00, 2),
            "delivery_date": (datetime.now() + timedelta(days=1)).strftime("%Y-%m-%d")
        },
    ]
    
    rates.sort(key=lambda x: x["cost_usd"])
    
    return {
        "success": True,
        "data": {
            "rates": rates,
            "package_info": {
                "actual_weight_lbs": weight_lbs,
                "dimensional_weight_lbs": round(dim_weight, 2),
                "billable_weight_lbs": round(billable, 2),
                "note": "Billable weight is the higher of actual and dimensional weight"
            }
        }
    }

def validate_shipping_address(
    street: str,
    city: str,
    state: str,
    postal_code: str,
    country: str = "US"
) -> dict:
    # Basic validation
    issues = []
    
    if not postal_code.replace("-", "").isdigit():
        issues.append("ZIP code must be numeric")
    
    if len(state) != 2:
        issues.append("State must be 2-letter abbreviation (e.g., CA, NY, TX)")
    
    if not street[0].isdigit():
        issues.append("Street address should start with a number")
    
    if issues:
        return {
            "success": True,
            "valid": False,
            "issues": issues,
            "suggestion": "Please correct the address and try again"
        }
    
    return {
        "success": True,
        "valid": True,
        "standardized": {
            "street": street.upper(),
            "city": city.upper(),
            "state": state.upper(),
            "postal_code": postal_code[:5],
            "country": country
        },
        "residential": True,
        "deliverable": True,
        "corrections_made": []
    }

def create_booking(
    origin_street: str,
    origin_city: str,
    origin_state: str,
    origin_zip: str,
    destination_street: str,
    destination_city: str,
    destination_state: str,
    destination_zip: str,
    weight_lbs: float,
    carrier: str,
    service: str,
    pickup_date: str
) -> dict:
    booking_ref = ''.join(random.choices(string.ascii_uppercase + string.digits, k=10))
    tracking = "1Z" + ''.join(random.choices(string.digits, k=16))
    
    service_days = {"Ground": 4, "2nd Day Air": 2, "Priority Overnight": 1}
    days = service_days.get(service, 4)
    delivery = (datetime.strptime(pickup_date, "%Y-%m-%d") + timedelta(days=days)).strftime("%Y-%m-%d")
    
    cost = weight_lbs * (1.2 if "Ground" in service else 2.5 if "2nd" in service else 4.5) * 4.25 + 10
    
    return {
        "success": True,
        "data": {
            "booking_reference": booking_ref,
            "tracking_number": tracking,
            "carrier": carrier,
            "service": service,
            "pickup_date": pickup_date,
            "estimated_delivery": delivery,
            "cost_usd": round(cost, 2),
            "label_url": f"https://api.shipping.demo/labels/{booking_ref}.pdf",
            "status": "confirmed",
            "message": f"Booking confirmed! Your shipment will be picked up on {pickup_date} and delivered by {delivery}."
        }
    }

def get_compliance_check(
    product_description: str,
    hs_code: Optional[str],
    origin_country: str,
    destination_country: str,
    declared_value_usd: float
) -> dict:
    flags = []
    
    if declared_value_usd > 800 and destination_country == "US":
        flags.append({
            "rule": "US Formal Entry",
            "description": "Shipments over $800 to the US require formal customs entry",
            "action_required": "Prepare commercial invoice and packing list for customs broker"
        })
    
    if declared_value_usd > 2500 and destination_country == "US":
        flags.append({
            "rule": "US Electronic Export Information (EEI)",
            "description": "Shipments over $2500 require EEI filing through AES",
            "action_required": "File Electronic Export Information before shipping"
        })
    
    if "lithium" in product_description.lower() or "battery" in product_description.lower():
        flags.append({
            "rule": "Dangerous Goods - Class 9",
            "description": "Lithium batteries are classified as Class 9 dangerous goods",
            "action_required": "Ensure proper DG declaration, UN3480 or UN3481 label, and watt-hour documentation"
        })
    
    if origin_country in ["IR", "KP", "CU", "SY"] or destination_country in ["IR", "KP", "CU", "SY"]:
        flags.append({
            "rule": "OFAC Sanctioned Country",
            "description": "Shipment involves an OFAC-sanctioned country",
            "action_required": "STOP - Legal review required before proceeding"
        })
    
    return {
        "success": True,
        "data": {
            "compliance_status": "HOLD" if any(f["rule"].startswith("OFAC") for f in flags) else ("REVIEW" if flags else "CLEAR"),
            "flags": flags,
            "summary": f"Found {len(flags)} compliance item(s) requiring attention" if flags else "No compliance issues detected"
        }
    }

# ---- Tool Definitions ----

LOGISTICS_TOOLS = [
    {
        "name": "get_shipment_tracking",
        "description": """Track a shipment by tracking number. Returns current status, location, 
        estimated delivery date, and tracking history. Use when user asks about shipment status, 
        where a package is, or delivery updates.""",
        "input_schema": {
            "type": "object",
            "properties": {
                "tracking_number": {
                    "type": "string",
                    "description": "The carrier tracking number. Remove spaces. Example: 'DEMO123', '1Z999AA10123456784'"
                }
            },
            "required": ["tracking_number"]
        }
    },
    {
        "name": "get_carrier_rates",
        "description": """Get shipping rate quotes from multiple carriers. Returns list of options 
        sorted by price with transit times. Use when user asks about shipping costs or wants to 
        compare carrier options. Does NOT create a booking.""",
        "input_schema": {
            "type": "object",
            "properties": {
                "origin_zip": {"type": "string", "description": "5-digit origin ZIP code"},
                "destination_zip": {"type": "string", "description": "5-digit destination ZIP code"},
                "weight_lbs": {"type": "number", "description": "Package weight in pounds (required)"},
                "length_in": {"type": "number", "description": "Length in inches (default 12)"},
                "width_in": {"type": "number", "description": "Width in inches (default 12)"},
                "height_in": {"type": "number", "description": "Height in inches (default 12)"}
            },
            "required": ["origin_zip", "destination_zip", "weight_lbs"]
        }
    },
    {
        "name": "validate_shipping_address",
        "description": """Validate a shipping address before creating a booking. Returns whether 
        address is valid and deliverable. Always call this before create_booking.""",
        "input_schema": {
            "type": "object",
            "properties": {
                "street": {"type": "string", "description": "Street address including house number"},
                "city": {"type": "string"},
                "state": {"type": "string", "description": "2-letter US state abbreviation"},
                "postal_code": {"type": "string", "description": "ZIP code"},
                "country": {"type": "string", "description": "Country code, default US"}
            },
            "required": ["street", "city", "state", "postal_code"]
        }
    },
    {
        "name": "create_booking",
        "description": """Create an actual shipment booking. IMPORTANT: Only call after:
        1. validate_shipping_address has confirmed destination is valid
        2. User has confirmed the rate and carrier
        3. User has explicitly asked to proceed with booking
        This creates a real booking with cost implications.""",
        "input_schema": {
            "type": "object",
            "properties": {
                "origin_street": {"type": "string"},
                "origin_city": {"type": "string"},
                "origin_state": {"type": "string"},
                "origin_zip": {"type": "string"},
                "destination_street": {"type": "string"},
                "destination_city": {"type": "string"},
                "destination_state": {"type": "string"},
                "destination_zip": {"type": "string"},
                "weight_lbs": {"type": "number"},
                "carrier": {"type": "string", "description": "Carrier name: UPS, FedEx"},
                "service": {"type": "string", "description": "Service level: Ground, 2nd Day Air, Priority Overnight"},
                "pickup_date": {"type": "string", "description": "Pickup date in YYYY-MM-DD format"}
            },
            "required": ["origin_street", "origin_city", "origin_state", "origin_zip",
                        "destination_street", "destination_city", "destination_state", "destination_zip",
                        "weight_lbs", "carrier", "service", "pickup_date"]
        }
    },
    {
        "name": "get_compliance_check",
        "description": """Check shipment compliance requirements: customs rules, dangerous goods, 
        export controls, sanctions. Use for international shipments or when product might have 
        regulatory requirements (batteries, chemicals, high-value goods).""",
        "input_schema": {
            "type": "object",
            "properties": {
                "product_description": {"type": "string", "description": "What is being shipped"},
                "hs_code": {"type": "string", "description": "HS/HTS code if known, otherwise null"},
                "origin_country": {"type": "string", "description": "2-letter ISO country code of origin"},
                "destination_country": {"type": "string", "description": "2-letter ISO country code of destination"},
                "declared_value_usd": {"type": "number", "description": "Declared value in USD"}
            },
            "required": ["product_description", "origin_country", "destination_country", "declared_value_usd"]
        }
    }
]

TOOL_FUNCTIONS = {
    "get_shipment_tracking": lambda **kwargs: get_shipment_tracking(**kwargs),
    "get_carrier_rates": lambda **kwargs: get_carrier_rates(**kwargs),
    "validate_shipping_address": lambda **kwargs: validate_shipping_address(**kwargs),
    "create_booking": lambda **kwargs: create_booking(**kwargs),
    "get_compliance_check": lambda **kwargs: get_compliance_check(**kwargs)
}

SYSTEM_PROMPT = """You are a logistics assistant for a freight forwarding company. 
You help customers track shipments, get rate quotes, validate addresses, check compliance, 
and create bookings.

## Guidelines
- Be concise and practical in your responses
- Always validate addresses before creating bookings
- For international shipments, always run compliance check
- Present rate options in a clear table format when showing multiple options
- Confirm with the user before creating bookings (they have cost implications)
- If a tool returns an error, explain what went wrong and what the user can do

## Capabilities
You have access to tools for: tracking shipments, getting carrier rates, validating addresses,
checking compliance requirements, and creating bookings."""

def run_logistics_agent(user_message: str, verbose: bool = True) -> str:
    """
    Full logistics agent with tool calling.
    This is the demo-quality implementation.
    """
    messages = [{"role": "user", "content": user_message}]
    total_input_tokens = 0
    total_output_tokens = 0
    
    for iteration in range(10):
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=4096,
            system=SYSTEM_PROMPT,
            tools=LOGISTICS_TOOLS,
            messages=messages
        )
        
        total_input_tokens += response.usage.input_tokens
        total_output_tokens += response.usage.output_tokens
        
        if verbose:
            print(f"\n[Iteration {iteration+1}] stop_reason={response.stop_reason}, "
                  f"tokens: {response.usage.input_tokens}in/{response.usage.output_tokens}out")
        
        messages.append({"role": "assistant", "content": response.content})
        
        if response.stop_reason == "end_turn":
            final_text = next((b.text for b in response.content if hasattr(b, 'text')), "")
            
            if verbose:
                print(f"\n[Total tokens: {total_input_tokens} input, {total_output_tokens} output]")
                estimated_cost = (total_input_tokens/1_000_000 * 3.0) + (total_output_tokens/1_000_000 * 15.0)
                print(f"[Estimated cost: ${estimated_cost:.4f}]")
            
            return final_text
        
        elif response.stop_reason == "tool_use":
            tool_results = []
            
            for block in response.content:
                if block.type == "tool_use":
                    if verbose:
                        print(f"  → Tool: {block.name}({json.dumps(block.input)[:100]})")
                    
                    try:
                        fn = TOOL_FUNCTIONS[block.name]
                        result = fn(**block.input)
                    except Exception as e:
                        result = {"success": False, "error": str(e)}
                    
                    if verbose:
                        print(f"  ← Result: {json.dumps(result)[:150]}")
                    
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": json.dumps(result)
                    })
            
            messages.append({"role": "user", "content": tool_results})
    
    return "I wasn't able to complete this request. Please try again."


# ---- Demo Scenarios ----

def run_demos():
    print("=" * 60)
    print("LOGISTICS AGENT DEMO")
    print("=" * 60)
    
    demos = [
        "Where is my package? Tracking number DEMO123",
        
        "I need to ship a 15 lb box (12x10x8 inches) from ZIP 10001 (NYC) to ZIP 90210 (LA). What are my options?",
        
        """I need to ship 20 laptops (4.5 lbs each, 15x11x2 inches each) with lithium batteries 
        from our warehouse at 123 Main St, Austin TX 78701 to a customer at 456 Oak Ave, 
        Chicago IL 60601. The value is $35,000. What do I need to know and what will it cost?""",
        
        "Ship 5 boxes from 789 Commerce St, Dallas TX 75201 to 321 Harbor Blvd, Los Angeles CA 90731, each 10 lbs, using UPS Ground. Today is 2024-03-18."
    ]
    
    for i, scenario in enumerate(demos):
        print(f"\n{'='*60}")
        print(f"SCENARIO {i+1}:")
        print(f"User: {scenario[:100]}{'...' if len(scenario)>100 else ''}")
        print("-" * 40)
        
        result = run_logistics_agent(scenario, verbose=True)
        
        print(f"\nAgent: {result[:500]}{'...' if len(result)>500 else ''}")
        print()

if __name__ == "__main__":
    run_demos()
```

---

## Common Failure Modes

### Failure Mode 1: Tool Not Getting Called

**Symptom:** LLM answers from its training knowledge instead of calling the relevant tool.

**Diagnosis:** The tool description doesn't match the user's query semantics. The LLM doesn't see the connection.

**Fix:** Add explicit trigger phrases to the tool description:
```python
"description": """...
TRIGGERS (use this tool when user says any of these):
- "where is my package"
- "track my shipment"  
- "status of order"
- "has it been delivered"
- "tracking update"
"""
```

Also check: are you passing `tools=` in the API call? Easy to forget.

### Failure Mode 2: Wrong Tool Arguments

**Symptom:** LLM calls the right tool but with wrong argument types or missing required fields.

**Diagnosis:** Usually enum mismatch (model passes "FedEx" when enum expects "fedex") or type coercion (passes "15.5" instead of 15.5).

**Fix:**
- Add example values in field descriptions
- Make enums case-insensitive in your tool implementation
- Handle string-to-number coercion in your tools:
```python
def get_carrier_rates(origin_zip: str, destination_zip: str, weight_lbs, **kwargs):
    weight_lbs = float(weight_lbs)  # Coerce just in case
```

### Failure Mode 3: Infinite Tool Calling Loop

**Symptom:** Agent keeps calling tools without reaching a final answer.

**Diagnosis:** Usually the tool is returning results the LLM doesn't understand, or there's a dependency loop (tool A needs info from tool B which needs info from A).

**Fix:**
- Add max_iterations guard (already in the loop above)
- Improve tool error messages so LLM can break the loop
- Add "if you've called a tool more than twice with no progress, tell the user you cannot complete the request" to the system prompt

### Failure Mode 4: Tool Results Ignored

**Symptom:** Claude calls the tool, gets the result, but then gives an answer that ignores the tool result.

**Diagnosis:** Usually the tool result format is confusing, or there's so much irrelevant data in the result that the relevant part is buried.

**Fix:** Return clean, focused JSON. Don't return 500 fields when the LLM only needs 5:
```python
# BAD: Returns entire API response with 50 fields
return carrier_api.full_response_dict()

# GOOD: Return only what the LLM needs to answer the user's question
return {
    "status": "in_transit",
    "location": "Chicago, IL",
    "estimated_delivery": "2024-03-18",
    "on_time": True
}
```

### Failure Mode 5: Tool Called With Incomplete Information

**Symptom:** LLM calls `create_booking` without having all the required information.

**Fix:** The tool definition should make dependencies explicit:
```python
"description": """... ONLY call this tool after:
1. You have validated the destination address
2. You have confirmed the rate with the user
3. The user has said 'yes, proceed' or equivalent

If you are missing any required information, ask the user BEFORE calling this tool."""
```

---

## Interview Angles

### "Explain how tool calling works in Claude."

**Great answer:**
"Tool calling is a protocol where you give the LLM a set of function definitions — name, description, and JSON Schema for inputs. When the user asks something that requires one of those functions, Claude returns a `tool_use` content block instead of a text response. This block contains the function name and the extracted arguments. Your code then executes that function and returns the result back to Claude in a `tool_result` block. Claude uses the result to formulate its final answer.

The key insight is that Claude never directly executes anything — it just produces structured JSON. You execute it. This separation means you can add authorization checks, rate limiting, logging, and validation on your side before actually running the function. The LLM is doing argument extraction and routing, your code is doing execution."

### "How do you handle tool call failures in an agent?"

**Great answer:**
"Two levels: first, tool-level error handling — the function itself should return structured errors that the LLM can understand and relay to the user, not throw exceptions that crash the agent. Something like `{"success": false, "error": "Carrier API unavailable", "suggestion": "Try again in 5 minutes"}`.

Second, system-level protection — a max_iterations guard prevents infinite loops, and I catch any uncaught exceptions from tool execution and return them as error tool_results so the LLM can handle them gracefully rather than the whole agent crashing.

For persistent failures, I'd want the agent to tell the user what it tried and what failed, rather than silently retrying forever."

### "What's the difference between tool calling and RAG?"

**Great answer:**
"RAG is about providing the LLM with relevant context before it answers — you retrieve documents, inject them into the prompt, and the LLM generates a grounded answer. It's read-only and works at inference time.

Tool calling is about giving the LLM the ability to take actions — query APIs, run calculations, write to databases. It's not just about reading context; it's about doing things.

They're complementary. In a logistics context: RAG might retrieve the relevant tariff schedule from a vector store to answer 'what's the duty rate for electronics from China?' Tool calling might then call the customs API to file the actual declaration. Both end up in the prompt via tool_result, but the mechanism and use case are different."

---

## Practice Exercise

**Scenario:** Build a "Carrier Rate Optimizer" agent for a small e-commerce business. The business ships 50-200 packages per day and wants to automatically select the cheapest carrier for each shipment based on destination, weight, and required delivery date.

**Requirements:**

1. **Tools to implement:**
   - `get_multi_carrier_rates(origin_zip, destination_zip, weight_lbs, dimensions)` — returns rates from at least 3 carriers
   - `check_carrier_service_area(carrier, destination_zip)` — some carriers don't serve certain areas
   - `get_carrier_reliability_score(carrier, destination_state)` — returns historical on-time % for this lane
   - `log_rate_decision(package_id, selected_carrier, rate, reason)` — logs the decision for analytics

2. **Build the agent** that:
   - Takes a package (weight, dimensions, destination, required delivery date)
   - Gets rates from all carriers
   - Checks service area coverage
   - Incorporates reliability scores (hint: define your own weighting)
   - Selects the best carrier with a clear explanation
   - Logs the decision

3. **Add a "cheapest safe" mode** vs "fastest" mode: the user can tell the agent their priority.

4. **Test with edge cases:**
   - Package destination is in a rural area (hint: some carriers don't serve it)
   - Required delivery date is tomorrow (eliminates ground options)
   - Package exceeds carrier weight limit for some services

5. **Monitoring:** After running 10 test packages, report total token usage and estimated cost. At what volume (packages/month) does this approach pay for itself vs a manual rate-shopping process that takes 3 minutes per package at $30/hr labor?

**Bonus:** Add parallel tool execution so rates from all carriers are fetched simultaneously.

---

*Next: [03-rag.md](./03-rag.md) — Ground your agent in customer-specific knowledge with retrieval-augmented generation.*  
*Also see: [04-agents-orchestration.md](./04-agents-orchestration.md) — LangGraph for complex multi-step agent workflows.*
