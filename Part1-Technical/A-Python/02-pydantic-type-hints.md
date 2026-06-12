# 02 — Pydantic v2 and Python Type Hints: Schema Contracts at Customer Boundaries

> **Related files:** [01-async-await.md](./01-async-await.md) | [../B-FastAPI/02-pydantic-validation.md](../B-FastAPI/02-pydantic-validation.md) | [06-pandas-data-wrangling.md](./06-pandas-data-wrangling.md)

---

## Why This Is Your First Defense Line

Picture this: it's 11pm, your integration is live with a customer's warehouse management system, and alarms are firing. An order payload hit your endpoint looking like this:

```json
{
  "order_id": "ORD-5523",
  "ship_date": "13/07/2024",
  "weight_lbs": "12.5 lbs",
  "priority": "URGANT",
  "items": null
}
```

Your service needs to:
- Parse `"13/07/2024"` as a date (DD/MM/YYYY format, not ISO)
- Strip `" lbs"` from weight and convert to float
- Reject `"URGANT"` (not a valid priority) with a clear error message
- Treat `null` items as an empty list

Without Pydantic, you write 50 lines of parsing code that has bugs. With Pydantic, you write a model and validators, and the error messages are production-ready.

This module teaches you everything you need to handle the full range of real-world customer data.

---

## Part 1: Python Type Hints — From Basics to Advanced

Type hints don't change runtime behavior (Python doesn't enforce them natively), but they power Pydantic, FastAPI, and your IDE's autocomplete. You need to understand them deeply.

### Basic Types

```python
# Primitives
name: str = "FedEx"
count: int = 42
weight: float = 12.5
active: bool = True

# Collections
shipment_ids: list[str] = ["SHIP-001", "SHIP-002"]
order_map: dict[str, int] = {"ORD-001": 100}
unique_statuses: set[str] = {"IN_TRANSIT", "DELIVERED"}
coordinates: tuple[float, float] = (41.8781, -87.6298)

# Nested collections
tracking_by_order: dict[str, list[str]] = {
    "ORD-001": ["TRACK-A", "TRACK-B"]
}
```

### Optional and Union

```python
from typing import Optional, Union

# Optional[X] is exactly Union[X, None]
# Both mean "can be X or None"
customer_note: Optional[str] = None  # old style, still common
customer_note: str | None = None     # Python 3.10+ preferred style

# Union: can be any of these types
shipment_id: Union[str, int] = "SHIP-001"  # old style
shipment_id: str | int = "SHIP-001"        # Python 3.10+ preferred

# When to use Optional vs Union:
# Optional: field might be missing/null (very common in real data)
# Union: field can legitimately be different types (less common, treat carefully)
```

### Literal — Enumerated String Types

```python
from typing import Literal

# Only these exact values are valid
ShipmentStatus = Literal["PENDING", "IN_TRANSIT", "DELIVERED", "EXCEPTION"]
Priority = Literal["LOW", "MEDIUM", "HIGH", "URGENT"]
CarrierCode = Literal["FEDEX", "UPS", "USPS", "DHL", "ONTRAC"]

def update_shipment_status(
    shipment_id: str,
    status: ShipmentStatus
) -> None:
    # IDE knows status can only be those 5 strings
    pass

# Literal is great for discriminated unions (see later)
```

### TypeVar and Generic Types

```python
from typing import TypeVar, Generic

T = TypeVar('T')

class APIResponse(Generic[T]):
    """
    Generic response envelope used across all API endpoints.
    T is the type of the data payload.
    """
    def __init__(self, data: T, success: bool, message: str):
        self.data = data
        self.success = success
        self.message = message
    
    def unwrap(self) -> T:
        if not self.success:
            raise ValueError(f"Response was not successful: {self.message}")
        return self.data

# Usage — IDE knows the exact return type
order_response: APIResponse[dict] = APIResponse(
    data={"order_id": "123"},
    success=True,
    message="OK"
)
order_data: dict = order_response.unwrap()  # IDE knows this is dict
```

### Protocol — Structural Subtyping (Duck Typing With Types)

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Trackable(Protocol):
    """Any object that has these methods is Trackable — no inheritance needed."""
    
    def get_tracking_id(self) -> str: ...
    def get_carrier(self) -> str: ...
    def get_status(self) -> str: ...

class FedExShipment:
    """Doesn't inherit from Trackable, but is structurally compatible."""
    
    def get_tracking_id(self) -> str:
        return self._tracking_id
    
    def get_carrier(self) -> str:
        return "FEDEX"
    
    def get_status(self) -> str:
        return self._status

class UPSShipment:
    def get_tracking_id(self) -> str: ...
    def get_carrier(self) -> str: return "UPS"
    def get_status(self) -> str: ...

def process_trackable(item: Trackable) -> dict:
    """Works with any object implementing the Trackable protocol."""
    return {
        "tracking_id": item.get_tracking_id(),
        "carrier": item.get_carrier(),
        "status": item.get_status()
    }

# Both work without inheritance
fedex = FedExShipment()
ups = UPSShipment()
print(isinstance(fedex, Trackable))  # True (runtime_checkable)
```

### Type Aliases

```python
from typing import TypeAlias

# Simple aliases for readability
TrackingNumber: TypeAlias = str
OrderID: TypeAlias = str
WarehouseCode: TypeAlias = str

# Complex nested type aliases
ShipmentBatch: TypeAlias = list[dict[str, str | int | float]]
CarrierRateTable: TypeAlias = dict[str, dict[str, float]]

# In function signatures
def assign_carrier(
    shipment_id: TrackingNumber,
    warehouse: WarehouseCode
) -> CarrierCode:
    ...
```

---

## Part 2: Pydantic v2 Fundamentals

### The Pydantic v2 Mental Model

Pydantic v2 (released June 2023) was a complete rewrite in Rust. Key differences from v1:
- **10-50x faster** than v1
- Different import paths (`from pydantic import field_validator` not `validator`)
- `model_dump()` instead of `.dict()`
- `model_validate()` instead of `.from_orm()` or `.parse_obj()`
- `model_json_schema()` instead of `.schema()`

### Basic Model

```python
from pydantic import BaseModel, Field
from datetime import datetime, date
from decimal import Decimal

class Address(BaseModel):
    street1: str
    street2: str | None = None
    city: str
    state: str
    postal_code: str
    country: str = "US"  # default value

class Shipment(BaseModel):
    shipment_id: str
    tracking_number: str
    carrier: str
    status: str
    ship_date: date
    estimated_delivery: date | None = None
    weight_lbs: float
    declared_value: Decimal | None = None
    origin: Address
    destination: Address
    tags: list[str] = Field(default_factory=list)  # mutable default: use default_factory

# Creating a model
shipment = Shipment(
    shipment_id="SHIP-001",
    tracking_number="1Z999AA1012345678",
    carrier="UPS",
    status="IN_TRANSIT",
    ship_date="2024-01-15",  # Pydantic auto-converts string to date
    weight_lbs=12.5,
    origin=Address(street1="100 Main St", city="Chicago", state="IL", postal_code="60601"),
    destination=Address(street1="200 Oak Ave", city="Dallas", state="TX", postal_code="75201"),
)

# Accessing fields
print(shipment.carrier)            # "UPS"
print(shipment.ship_date)         # date(2024, 1, 15) — a Python date object, not a string
print(shipment.origin.city)       # "Chicago"
print(shipment.tags)              # [] — default_factory gave us a fresh list

# Immutability by default in v2 (sort of — see model_config)
# shipment.carrier = "FedEx"  # This works by default (models are mutable)
```

### Field() — The Full Power

```python
from pydantic import BaseModel, Field
from typing import Annotated

class Order(BaseModel):
    # Basic with description and example
    order_id: str = Field(
        description="Unique order identifier from ERP",
        examples=["ORD-2024-001", "PO-78234"]
    )
    
    # Numeric constraints
    quantity: int = Field(gt=0, le=10000, description="Must be 1-10000")
    
    # String constraints
    sku: str = Field(min_length=3, max_length=50, pattern=r"^SKU-[A-Z0-9]+$")
    
    # Float with constraints
    unit_price: float = Field(ge=0.0, description="Price must be non-negative")
    
    # Alias: accept "orderID" from JSON but use "order_id" in Python
    customer_po: str | None = Field(
        None,
        alias="customerPO",           # JSON field name
        description="Customer purchase order number"
    )
    
    # Validation alias (for input only), serialization alias (for output only)
    priority_code: str = Field(
        default="STANDARD",
        validation_alias="PriorityCode",   # accept this in input
        serialization_alias="priority",    # use this in output
    )

# Annotated style — reusable field definitions
PositiveInt = Annotated[int, Field(gt=0)]
ShortString = Annotated[str, Field(max_length=100)]
NonEmptyStr = Annotated[str, Field(min_length=1)]

class CleanOrder(BaseModel):
    order_id: NonEmptyStr
    quantity: PositiveInt
    notes: ShortString | None = None
```

### Validators — field_validator and model_validator

```python
from pydantic import BaseModel, Field, field_validator, model_validator
from typing import Self
import re

class ShipmentImport(BaseModel):
    """
    Model for importing shipments from a messy CSV or legacy system.
    Heavy validation to coerce real-world data.
    """
    shipment_id: str
    
    # Weight might come in as "12.5 lbs" or "12.5" or 12.5
    weight_raw: str | float = Field(alias="weight")
    weight_lbs: float = Field(default=0.0, exclude=True)  # computed, excluded from schema
    
    # Date might be "01/15/2024" or "2024-01-15" or "Jan 15, 2024"
    ship_date_raw: str = Field(alias="ship_date")
    
    # Priority might be lowercase or have typos
    priority: str = Field(default="STANDARD")
    
    carrier: str
    tracking_number: str
    
    # ─── Field validators ───────────────────────────────────────────────────
    
    @field_validator("weight_raw", mode="before")
    @classmethod
    def parse_weight(cls, v: str | float) -> float:
        """Accept '12.5 lbs', '12.5', 12.5, '12.5kg' and normalize to float lbs."""
        if isinstance(v, (int, float)):
            return float(v)
        
        if isinstance(v, str):
            # Strip units
            cleaned = v.strip().lower()
            
            # Handle kg conversion
            if "kg" in cleaned:
                num_str = cleaned.replace("kg", "").strip()
                return float(num_str) * 2.20462  # kg to lbs
            
            # Strip any other unit text
            num_str = re.sub(r"[^0-9.]", "", cleaned)
            if not num_str:
                raise ValueError(f"Cannot parse weight from '{v}'")
            return float(num_str)
        
        raise ValueError(f"Unexpected weight type: {type(v)}")
    
    @field_validator("priority", mode="before")
    @classmethod
    def normalize_priority(cls, v: str) -> str:
        """Normalize priority to uppercase and handle common typos."""
        if not isinstance(v, str):
            return "STANDARD"
        
        normalized = v.strip().upper()
        
        # Handle typos (FDE reality: customer data has these)
        typo_map = {
            "URGANT": "URGENT",
            "URGNET": "URGENT",
            "HIGHT": "HIGH",
            "HI": "HIGH",
            "LO": "LOW",
            "STD": "STANDARD",
            "NORM": "STANDARD",
        }
        
        return typo_map.get(normalized, normalized)
    
    @field_validator("carrier", mode="before")
    @classmethod
    def normalize_carrier(cls, v: str) -> str:
        """Normalize carrier names to standard codes."""
        carrier_aliases = {
            "federal express": "FEDEX",
            "fedex": "FEDEX",
            "fdx": "FEDEX",
            "united parcel service": "UPS",
            "ups": "UPS",
            "u.p.s.": "UPS",
            "united states postal service": "USPS",
            "usps": "USPS",
            "post office": "USPS",
            "dhl express": "DHL",
            "dhl": "DHL",
        }
        return carrier_aliases.get(v.strip().lower(), v.strip().upper())
    
    @field_validator("tracking_number", mode="before")
    @classmethod
    def clean_tracking_number(cls, v: str) -> str:
        """Remove spaces and dashes from tracking numbers."""
        return re.sub(r"[\s\-]", "", str(v)).upper()
    
    # ─── Model validator — runs after all field validators ───────────────────
    
    @model_validator(mode="after")
    def validate_carrier_tracking_format(self) -> Self:
        """Validate that tracking number format matches carrier."""
        carrier = self.carrier
        tracking = self.tracking_number
        
        patterns = {
            "FEDEX": r"^\d{12}$|^\d{15}$|^\d{20}$",
            "UPS": r"^1Z[A-Z0-9]{16}$",
            "USPS": r"^\d{20,22}$",
            "DHL": r"^\d{10}$",
        }
        
        if carrier in patterns:
            if not re.match(patterns[carrier], tracking):
                raise ValueError(
                    f"Tracking number '{tracking}' doesn't match expected format "
                    f"for carrier '{carrier}'. Expected pattern: {patterns[carrier]}"
                )
        
        return self
    
    @model_validator(mode="before")
    @classmethod
    def handle_legacy_field_names(cls, data: dict) -> dict:
        """
        mode="before" runs before field validation.
        Great for renaming fields from legacy systems.
        """
        if isinstance(data, dict):
            # Legacy system uses "wt" instead of "weight"
            if "wt" in data and "weight" not in data:
                data["weight"] = data.pop("wt")
            
            # Legacy system uses "ship_carrier" instead of "carrier"
            if "ship_carrier" in data and "carrier" not in data:
                data["carrier"] = data.pop("ship_carrier")
        
        return data
```

### Discriminated Unions — Handle Multiple Payload Shapes

This is the advanced pattern for handling webhooks from different carriers/systems that each send different payload structures.

```python
from typing import Literal, Annotated, Union
from pydantic import BaseModel, Field

# ─── FedEx event payload ──────────────────────────────────────────────────────

class FedExDeliveryEvent(BaseModel):
    carrier: Literal["FEDEX"]  # discriminator field
    event_type: str
    master_tracking_number: str
    actual_delivery_dt: str | None = None
    delivery_attempts: int = 0
    
class FedExExceptionEvent(BaseModel):
    carrier: Literal["FEDEX"]
    event_type: str
    master_tracking_number: str
    exception_code: str
    exception_description: str

# ─── UPS event payload ────────────────────────────────────────────────────────

class UPSTrackingEvent(BaseModel):
    carrier: Literal["UPS"]
    event_type: str
    inquiry_number: str  # UPS calls it this, not "tracking_number"
    activity_status: str
    scheduled_delivery: str | None = None

# ─── Generic wrapper using discriminated union ───────────────────────────────

# Discriminated union: Pydantic uses "carrier" field to choose which model
CarrierEvent = Annotated[
    Union[FedExDeliveryEvent, FedExExceptionEvent, UPSTrackingEvent],
    Field(discriminator="carrier")
]

class WebhookPayload(BaseModel):
    event_id: str
    timestamp: str
    event: CarrierEvent  # Pydantic picks the right model based on event.carrier

# Demo
fedex_payload = WebhookPayload.model_validate({
    "event_id": "EVT-001",
    "timestamp": "2024-01-15T10:30:00Z",
    "event": {
        "carrier": "FEDEX",
        "event_type": "DL",
        "master_tracking_number": "123456789012",
        "actual_delivery_dt": "2024-01-15T14:22:00"
    }
})

# fedex_payload.event is typed as FedExDeliveryEvent
assert isinstance(fedex_payload.event, FedExDeliveryEvent)
```

### Nested Models and Recursive Models

```python
from __future__ import annotations  # needed for self-referential models
from pydantic import BaseModel
from typing import Optional

# ─── Simple nesting ───────────────────────────────────────────────────────────

class ContactInfo(BaseModel):
    name: str
    email: str | None = None
    phone: str | None = None

class CompanyInfo(BaseModel):
    company_id: str
    legal_name: str
    primary_contact: ContactInfo
    billing_contact: ContactInfo | None = None

class Order(BaseModel):
    order_id: str
    customer: CompanyInfo
    ship_to: ContactInfo

# ─── Recursive models (e.g., category hierarchy, BOM) ────────────────────────

class BOMItem(BaseModel):
    """Bill of Materials item — can contain sub-components."""
    sku: str
    description: str
    quantity: float
    unit: str
    children: list[BOMItem] = []  # recursive — a BOM item has sub-items

# For Pydantic v2, model_rebuild() needed for self-referential models
BOMItem.model_rebuild()

bom = BOMItem(
    sku="ASSY-001",
    description="Motor Assembly",
    quantity=1,
    unit="EA",
    children=[
        BOMItem(
            sku="MOTOR-001",
            description="DC Motor",
            quantity=1,
            unit="EA",
            children=[
                BOMItem(sku="COIL-001", description="Copper Coil", quantity=3, unit="EA"),
                BOMItem(sku="MAGNET-001", description="Neodymium Magnet", quantity=6, unit="EA"),
            ]
        ),
        BOMItem(sku="HOUSING-001", description="Motor Housing", quantity=1, unit="EA"),
    ]
)
```

---

## Part 3: Serialization and Deserialization

### model_dump() — Model to Dict

```python
from pydantic import BaseModel, Field
from datetime import date

class ShipmentSummary(BaseModel):
    shipment_id: str
    carrier: str
    status: str
    ship_date: date
    internal_notes: str | None = Field(None, exclude=True)  # never serialized
    _computed_score: float = 0.0  # private, never serialized

shipment = ShipmentSummary(
    shipment_id="SHIP-001",
    carrier="FEDEX",
    status="IN_TRANSIT",
    ship_date=date(2024, 1, 15),
    internal_notes="Customer is angry",
)

# Basic dump
print(shipment.model_dump())
# {'shipment_id': 'SHIP-001', 'carrier': 'FEDEX', 'status': 'IN_TRANSIT',
#  'ship_date': datetime.date(2024, 1, 15)}
# Note: internal_notes excluded because of exclude=True

# Exclude unset fields (fields not provided by caller)
partial = ShipmentSummary(shipment_id="SHIP-002", carrier="UPS", status="PENDING", ship_date=date.today())
print(partial.model_dump(exclude_unset=True))
# Only includes fields that were explicitly set

# Dump with mode (what to do with special types)
print(shipment.model_dump(mode="json"))
# {'ship_date': '2024-01-15'}  ← date becomes ISO string

# Include/exclude specific fields
print(shipment.model_dump(include={"shipment_id", "status"}))
# {'shipment_id': 'SHIP-001', 'status': 'IN_TRANSIT'}

print(shipment.model_dump(exclude={"ship_date"}))
# everything except ship_date
```

### model_validate() — Input Validation

```python
from pydantic import BaseModel, ValidationError
from datetime import date

class ShipmentCreate(BaseModel):
    shipment_id: str
    carrier: str
    weight_lbs: float
    ship_date: date

# From dict (e.g., parsed JSON)
try:
    shipment = ShipmentCreate.model_validate({
        "shipment_id": "SHIP-001",
        "carrier": "FEDEX",
        "weight_lbs": "12.5",  # string — Pydantic coerces to float
        "ship_date": "2024-01-15",  # string — Pydantic coerces to date
    })
    print(shipment.weight_lbs)  # 12.5 (float)
    print(type(shipment.ship_date))  # <class 'datetime.date'>
except ValidationError as e:
    print(e.json(indent=2))

# From JSON string
shipment = ShipmentCreate.model_validate_json(
    '{"shipment_id": "SHIP-002", "carrier": "UPS", "weight_lbs": 8.2, "ship_date": "2024-01-16"}'
)

# From ORM object (e.g., SQLAlchemy model)
# model_config = ConfigDict(from_attributes=True)  # needed for ORM objects
# shipment = ShipmentCreate.model_validate(orm_object)
```

### model_json_schema() — OpenAPI Schema Generation

```python
from pydantic import BaseModel, Field
import json

class ShipmentCreate(BaseModel):
    """Create a new shipment."""
    
    shipment_id: str = Field(description="Unique ID from your system", examples=["SHIP-2024-001"])
    carrier: str = Field(description="Carrier code", examples=["FEDEX", "UPS"])
    weight_lbs: float = Field(gt=0, description="Gross weight in pounds")
    ship_date: str = Field(description="ISO date: YYYY-MM-DD", examples=["2024-01-15"])

schema = ShipmentCreate.model_json_schema()
print(json.dumps(schema, indent=2))
# {
#   "title": "ShipmentCreate",
#   "description": "Create a new shipment.",
#   "properties": {
#     "shipment_id": {
#       "description": "Unique ID from your system",
#       "examples": ["SHIP-2024-001"],
#       "type": "string"
#     },
#     ...
#   }
# }

# FastAPI uses this automatically for OpenAPI/Swagger
```

---

## Part 4: Model Configuration

### model_config with ConfigDict

```python
from pydantic import BaseModel, ConfigDict, Field

class FlexibleModel(BaseModel):
    """Model that accepts data from multiple naming conventions."""
    
    model_config = ConfigDict(
        # Accept both snake_case (Python) and camelCase (JSON API)
        populate_by_name=True,       # allow field name AND alias
        
        # Prevent extra fields from causing errors
        extra="ignore",              # silently ignore unknown fields
        # extra="forbid"             # raise error on unknown fields
        # extra="allow"              # accept and store unknown fields
        
        # Coerce types where possible (e.g., "123" → 123 for int fields)
        coerce_numbers_to_str=False,
        
        # For ORM integration
        from_attributes=True,        # allow .model_validate(orm_object)
        
        # Frozen (immutable after creation)
        frozen=False,                # set True for immutable models
        
        # Validation on assignment (check types when setting attributes)
        validate_assignment=True,
        
        # String representation
        str_strip_whitespace=True,   # auto-strip whitespace from all strings
        str_to_lower=False,          # don't auto-lowercase
        str_min_length=1,            # all strings must have at least 1 char
    )
    
    shipment_id: str = Field(alias="shipmentId")
    carrier_code: str = Field(alias="carrierCode")

# With populate_by_name=True, both work:
m1 = FlexibleModel(shipment_id="SHIP-001", carrier_code="FEDEX")      # Python field names
m2 = FlexibleModel(shipmentId="SHIP-001", carrierCode="FEDEX")        # aliases
m3 = FlexibleModel(**{"shipmentId": "SHIP-001", "carrierCode": "FEDEX"})

# ─── Alias generators ─────────────────────────────────────────────────────────

from pydantic.alias_generators import to_camel, to_snake, to_pascal

class CamelCaseModel(BaseModel):
    """All fields use camelCase in JSON automatically."""
    
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True
    )
    
    order_id: str        # → "orderId" in JSON
    customer_name: str   # → "customerName" in JSON
    ship_date: str       # → "shipDate" in JSON

model = CamelCaseModel(order_id="ORD-001", customer_name="Acme Corp", ship_date="2024-01-15")
print(model.model_dump(by_alias=True))
# {'orderId': 'ORD-001', 'customerName': 'Acme Corp', 'shipDate': '2024-01-15'}
```

---

## Part 5: Custom Types and Annotated Types

```python
from pydantic import BaseModel, field_validator, GetCoreSchemaHandler
from pydantic.functional_validators import AfterValidator, BeforeValidator
from typing import Annotated, Any
import re

# ─── Simple annotated types with validators ───────────────────────────────────

def validate_zip_code(v: str) -> str:
    """US ZIP code: 5 digits or 5+4 format."""
    if not re.match(r"^\d{5}(-\d{4})?$", v):
        raise ValueError(f"Invalid US ZIP code: '{v}'")
    return v

def validate_phone_number(v: str) -> str:
    """Accept various formats, normalize to E.164."""
    digits = re.sub(r"\D", "", v)
    if len(digits) == 10:
        return f"+1{digits}"
    elif len(digits) == 11 and digits[0] == "1":
        return f"+{digits}"
    elif len(digits) >= 7:
        return f"+{digits}"  # international
    raise ValueError(f"Cannot parse phone number: '{v}'")

def validate_tracking_number(v: str) -> str:
    """Clean and validate tracking number format."""
    cleaned = re.sub(r"[\s\-]", "", v).upper()
    if len(cleaned) < 10:
        raise ValueError(f"Tracking number too short: '{v}'")
    return cleaned

# Create reusable annotated types
USZipCode = Annotated[str, AfterValidator(validate_zip_code)]
PhoneNumber = Annotated[str, BeforeValidator(validate_phone_number)]
TrackingNumber = Annotated[str, AfterValidator(validate_tracking_number)]

class ShipmentAddress(BaseModel):
    street1: str
    city: str
    state: str
    postal_code: USZipCode       # automatically validated
    phone: PhoneNumber | None = None  # automatically normalized

class Shipment(BaseModel):
    tracking_number: TrackingNumber  # automatically cleaned
    origin: ShipmentAddress
    destination: ShipmentAddress

# ─── Custom Pydantic type with full schema integration ────────────────────────

from decimal import Decimal, InvalidOperation

class CurrencyAmount(Decimal):
    """
    A Decimal that validates as a currency amount (2 decimal places, non-negative).
    Can be used as a type annotation.
    """
    
    @classmethod
    def __get_pydantic_core_schema__(
        cls,
        source_type: Any,
        handler: GetCoreSchemaHandler
    ):
        from pydantic_core import core_schema
        
        return core_schema.no_info_after_validator_function(
            cls._validate,
            core_schema.str_schema(),
        )
    
    @classmethod
    def _validate(cls, v: str) -> "CurrencyAmount":
        try:
            amount = Decimal(str(v)).quantize(Decimal("0.01"))
        except InvalidOperation:
            raise ValueError(f"Cannot convert '{v}' to currency amount")
        
        if amount < 0:
            raise ValueError(f"Currency amount must be non-negative: {v}")
        
        return cls(amount)

class InvoiceLineItem(BaseModel):
    sku: str
    description: str
    quantity: int
    unit_price: CurrencyAmount
    line_total: CurrencyAmount | None = None
    
    @model_validator(mode="after")
    def calculate_line_total(self) -> "InvoiceLineItem":
        if self.line_total is None:
            self.line_total = CurrencyAmount(self.unit_price * self.quantity)
        return self
```

---

## Part 6: Real-World Logistics Payload Modeling

This is what you'll actually build for customers. A complete model for a logistics integration.

```python
"""
logistics_models.py

Production-grade Pydantic models for a 3PL integration.
Handles: incoming orders from customer ERP, outgoing shipments to carrier,
         webhook events from carriers.
"""

from __future__ import annotations
from pydantic import BaseModel, Field, field_validator, model_validator, ConfigDict
from pydantic.alias_generators import to_camel
from typing import Annotated, Literal, Union
from datetime import datetime, date
from decimal import Decimal
from enum import Enum
import re

# ─── Enums ───────────────────────────────────────────────────────────────────

class ShipmentStatus(str, Enum):
    CREATED = "CREATED"
    LABEL_PRINTED = "LABEL_PRINTED"
    PICKED_UP = "PICKED_UP"
    IN_TRANSIT = "IN_TRANSIT"
    OUT_FOR_DELIVERY = "OUT_FOR_DELIVERY"
    DELIVERED = "DELIVERED"
    EXCEPTION = "EXCEPTION"
    RETURNED = "RETURNED"

class ServiceLevel(str, Enum):
    GROUND = "GROUND"
    TWO_DAY = "2DAY"
    OVERNIGHT = "OVERNIGHT"
    SAME_DAY = "SAMEDAY"

class PackageType(str, Enum):
    BOX = "BOX"
    ENVELOPE = "ENVELOPE"
    PALLET = "PALLET"
    TUBE = "TUBE"
    BAG = "BAG"

# ─── Base config ─────────────────────────────────────────────────────────────

class LogisticsBaseModel(BaseModel):
    """All logistics models inherit from this for consistent config."""
    
    model_config = ConfigDict(
        populate_by_name=True,
        str_strip_whitespace=True,
        use_enum_values=True,   # store enum .value, not enum object
        extra="ignore",          # real carrier APIs add random extra fields
    )

# ─── Address ─────────────────────────────────────────────────────────────────

class Address(LogisticsBaseModel):
    name: str = Field(min_length=1, max_length=100)
    company: str | None = Field(None, max_length=100)
    address_line1: str = Field(min_length=1, alias="addressLine1")
    address_line2: str | None = Field(None, alias="addressLine2")
    city: str = Field(min_length=1)
    state_province: str = Field(alias="stateProvince")
    postal_code: str = Field(alias="postalCode")
    country_code: str = Field(default="US", alias="countryCode", max_length=2)
    phone: str | None = None
    email: str | None = None
    is_residential: bool = Field(default=True, alias="isResidential")
    
    @field_validator("country_code", mode="before")
    @classmethod
    def normalize_country(cls, v: str) -> str:
        return v.strip().upper()
    
    @field_validator("postal_code", mode="before")
    @classmethod
    def clean_postal_code(cls, v: str) -> str:
        """Handle ZIP+4, Canadian postal codes, etc."""
        return str(v).strip()
    
    @field_validator("phone", mode="before")
    @classmethod
    def clean_phone(cls, v: str | None) -> str | None:
        if v is None:
            return None
        # Remove all non-digit characters
        digits = re.sub(r"\D", "", str(v))
        if len(digits) < 7:
            return None  # too short to be a real phone number
        return digits

# ─── Package dimensions and weight ───────────────────────────────────────────

class Dimensions(LogisticsBaseModel):
    length_in: float = Field(gt=0, alias="lengthIn")
    width_in: float = Field(gt=0, alias="widthIn")
    height_in: float = Field(gt=0, alias="heightIn")
    
    @property
    def dimensional_weight_lbs(self) -> float:
        """Standard dimensional weight calculation (DIM factor = 139)."""
        cubic_inches = self.length_in * self.width_in * self.height_in
        return cubic_inches / 139.0

class Package(LogisticsBaseModel):
    package_type: PackageType = Field(default=PackageType.BOX, alias="packageType")
    weight_lbs: float = Field(gt=0, alias="weightLbs")
    dimensions: Dimensions | None = None
    insured_value: Decimal | None = Field(None, alias="insuredValue")
    signature_required: bool = Field(default=False, alias="signatureRequired")
    reference_numbers: list[str] = Field(default_factory=list, alias="referenceNumbers")
    
    @property
    def billable_weight_lbs(self) -> float:
        """Carriers bill on the higher of actual vs dimensional weight."""
        if self.dimensions is None:
            return self.weight_lbs
        return max(self.weight_lbs, self.dimensions.dimensional_weight_lbs)

# ─── Order line item ─────────────────────────────────────────────────────────

class OrderLineItem(LogisticsBaseModel):
    line_number: int = Field(alias="lineNumber")
    sku: str = Field(min_length=1)
    description: str
    quantity: int = Field(gt=0)
    unit_of_measure: str = Field(default="EA", alias="unitOfMeasure")
    unit_price: Decimal | None = Field(None, alias="unitPrice")
    total_weight_lbs: float | None = Field(None, alias="totalWeightLbs")
    hazmat_class: str | None = Field(None, alias="hazmatClass")
    
    @field_validator("sku", mode="before")
    @classmethod
    def normalize_sku(cls, v: str) -> str:
        return str(v).strip().upper()

# ─── Inbound Order (from customer ERP) ───────────────────────────────────────

class InboundOrder(LogisticsBaseModel):
    """
    Order received from customer's ERP system.
    This is the messiest data you'll handle — be defensive.
    """
    order_id: str = Field(alias="orderId")
    customer_order_number: str | None = Field(None, alias="customerOrderNumber")
    purchase_order: str | None = Field(None, alias="purchaseOrder")
    
    order_date: date | None = Field(None, alias="orderDate")
    required_ship_date: date | None = Field(None, alias="requiredShipDate")
    required_delivery_date: date | None = Field(None, alias="requiredDeliveryDate")
    
    service_level: ServiceLevel = Field(default=ServiceLevel.GROUND, alias="serviceLevel")
    
    ship_from: Address = Field(alias="shipFrom")
    ship_to: Address = Field(alias="shipTo")
    
    line_items: list[OrderLineItem] = Field(alias="lineItems")
    packages: list[Package] = Field(default_factory=list)
    
    special_instructions: str | None = Field(None, alias="specialInstructions")
    customer_reference: str | None = Field(None, alias="customerReference")
    
    @field_validator("line_items", mode="before")
    @classmethod
    def ensure_non_empty_line_items(cls, v: list) -> list:
        if not v:
            raise ValueError("Order must have at least one line item")
        return v
    
    @field_validator("required_ship_date", "required_delivery_date", mode="before")
    @classmethod
    def parse_flexible_date(cls, v) -> date | None:
        """
        Customer ERPs send dates in many formats.
        Handle them all.
        """
        if v is None:
            return None
        
        if isinstance(v, date):
            return v
        
        if isinstance(v, str):
            v = v.strip()
            if not v:
                return None
            
            # Try multiple formats
            formats = [
                "%Y-%m-%d",       # ISO: 2024-01-15
                "%m/%d/%Y",       # US: 01/15/2024
                "%d/%m/%Y",       # EU: 15/01/2024
                "%Y%m%d",         # Compact: 20240115
                "%m-%d-%Y",       # US with dashes: 01-15-2024
                "%B %d, %Y",      # Written: January 15, 2024
                "%b %d, %Y",      # Abbreviated: Jan 15, 2024
            ]
            
            from datetime import datetime as dt
            for fmt in formats:
                try:
                    return dt.strptime(v, fmt).date()
                except ValueError:
                    continue
            
            raise ValueError(f"Cannot parse date from '{v}'. Expected formats: YYYY-MM-DD, MM/DD/YYYY, etc.")
        
        return None
    
    @model_validator(mode="after")
    def validate_ship_dates(self) -> InboundOrder:
        """Required ship date can't be after required delivery date."""
        if (
            self.required_ship_date is not None
            and self.required_delivery_date is not None
            and self.required_ship_date > self.required_delivery_date
        ):
            raise ValueError(
                f"required_ship_date ({self.required_ship_date}) cannot be "
                f"after required_delivery_date ({self.required_delivery_date})"
            )
        return self

# ─── Carrier Webhook Event ────────────────────────────────────────────────────

class FedExTrackEvent(LogisticsBaseModel):
    carrier: Literal["FEDEX"]
    tracking_number: str = Field(alias="trackingNumber")
    event_type: str = Field(alias="eventType")
    event_timestamp: datetime = Field(alias="eventTimestamp")
    event_description: str = Field(alias="eventDescription")
    location: str | None = None
    exception_code: str | None = Field(None, alias="exceptionCode")

class UPSTrackEvent(LogisticsBaseModel):
    carrier: Literal["UPS"]
    inquiry_number: str = Field(alias="inquiryNumber")
    activity_type: str = Field(alias="activityType")
    activity_timestamp: datetime = Field(alias="activityTimestamp")
    activity_description: str = Field(alias="activityDescription")
    city: str | None = None
    state: str | None = None

class DHLTrackEvent(LogisticsBaseModel):
    carrier: Literal["DHL"]
    waybill_number: str = Field(alias="waybillNumber")
    status_code: str = Field(alias="statusCode")
    timestamp: datetime
    description: str
    service_area: str | None = Field(None, alias="serviceArea")

# Discriminated union — pick model based on "carrier" field
TrackingEvent = Annotated[
    Union[FedExTrackEvent, UPSTrackEvent, DHLTrackEvent],
    Field(discriminator="carrier")
]

class CarrierWebhookPayload(LogisticsBaseModel):
    webhook_id: str = Field(alias="webhookId")
    received_at: datetime = Field(alias="receivedAt")
    events: list[TrackingEvent]

# ─── Usage and Validation Error Handling ─────────────────────────────────────

def process_inbound_order(raw_payload: dict) -> InboundOrder:
    """
    Parse and validate an inbound order.
    Returns human-readable errors for the customer's dev team.
    """
    from pydantic import ValidationError
    
    try:
        return InboundOrder.model_validate(raw_payload)
    except ValidationError as e:
        errors = e.errors(include_url=False)
        
        # Format errors in a way a customer's dev team can act on
        formatted_errors = []
        for error in errors:
            location = " → ".join(str(loc) for loc in error["loc"])
            message = error["msg"]
            input_value = error.get("input", "N/A")
            
            formatted_errors.append({
                "field": location,
                "error": message,
                "received": str(input_value)[:100],  # truncate long values
            })
        
        raise ValueError(
            f"Order payload validation failed with {len(formatted_errors)} error(s):\n" +
            "\n".join(
                f"  • {e['field']}: {e['error']} (received: {e['received']})"
                for e in formatted_errors
            )
        )

# ─── Demo ─────────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    from pydantic import ValidationError
    
    # Test with messy real-world data
    messy_payload = {
        "orderId": "ORD-2024-001",
        "customerOrderNumber": "PO-78234",
        "orderDate": "01/15/2024",           # US format
        "requiredShipDate": "2024-01-18",    # ISO format
        "requiredDeliveryDate": "20240122",   # compact format
        "serviceLevel": "2DAY",
        "shipFrom": {
            "name": "Warehouse Team",
            "addressLine1": "  100 Industrial Blvd  ",  # extra whitespace
            "city": "Chicago",
            "stateProvince": "IL",
            "postalCode": "60601",
            "countryCode": "us",  # lowercase — will be normalized
            "isResidential": False
        },
        "shipTo": {
            "name": "Jane Smith",
            "company": "Acme Corp",
            "addressLine1": "200 Oak Avenue",
            "city": "Dallas",
            "stateProvince": "TX",
            "postalCode": "75201-1234",
            "phone": "(214) 555-1234",  # various formats — will be normalized
            "isResidential": True
        },
        "lineItems": [
            {
                "lineNumber": 1,
                "sku": "sku-widget-l",   # lowercase — will be normalized
                "description": "Large Widget",
                "quantity": 5,
                "unitPrice": "12.50",
                "totalWeightLbs": 2.5
            }
        ],
        "unknown_extra_field": "this will be ignored",  # extra="ignore"
    }
    
    order = process_inbound_order(messy_payload)
    print(f"Order ID: {order.order_id}")
    print(f"Ship date: {order.required_ship_date}")
    print(f"Country: {order.ship_from.country_code}")  # "US" (normalized)
    print(f"Phone: {order.ship_to.phone}")             # "2145551234" (normalized)
    print(f"SKU: {order.line_items[0].sku}")           # "SKU-WIDGET-L" (normalized)
    print(f"Address: '{order.ship_from.address_line1}'")  # "100 Industrial Blvd" (stripped)
    
    # Test validation error output
    try:
        process_inbound_order({
            "orderId": "BAD-ORDER",
            "serviceLevel": "TELEPORTATION",  # invalid
            "shipFrom": {"name": "", "city": "Chicago"},  # missing required fields
            "shipTo": {"name": "Jane"},  # missing required fields
            "lineItems": [],  # empty — not allowed
        })
    except ValueError as e:
        print("\nValidation errors (human-readable):")
        print(e)
```

---

## Part 7: Handling Nullable and Optional Fields from Legacy Systems

Real-world customer data has infinite variations of "no value."

```python
from pydantic import BaseModel, field_validator
from typing import Any

class LegacyOrderField(BaseModel):
    """
    Handles all the ways legacy systems express "no value":
    null, "NULL", "N/A", "", "none", "undefined", 0 (for IDs), etc.
    """
    
    order_id: str | None = None
    reference_number: str | None = None
    weight: float | None = None
    notes: str | None = None
    
    @field_validator("order_id", "reference_number", "notes", mode="before")
    @classmethod
    def normalize_null_strings(cls, v: Any) -> str | None:
        """Convert all the ways a system can say 'nothing' to None."""
        if v is None:
            return None
        
        if isinstance(v, str):
            cleaned = v.strip()
            null_values = {
                "", "null", "NULL", "Null",
                "n/a", "N/A", "na", "NA",
                "none", "None", "NONE",
                "undefined", "UNDEFINED",
                "#N/A",  # Excel null
                "-",
                ".",
            }
            if cleaned in null_values:
                return None
            return cleaned
        
        return str(v)
    
    @field_validator("weight", mode="before")
    @classmethod
    def normalize_null_numbers(cls, v: Any) -> float | None:
        if v is None:
            return None
        
        if isinstance(v, str):
            cleaned = v.strip()
            null_values = {"", "null", "NULL", "n/a", "N/A", "none", "None", "-"}
            if cleaned in null_values:
                return None
            
            # Strip units
            import re
            num_str = re.sub(r"[^0-9.]", "", cleaned)
            if not num_str:
                return None
            return float(num_str)
        
        if isinstance(v, (int, float)):
            return float(v)
        
        return None
```

---

## FDE Context Callouts

> **FDE Context: Pydantic Errors Are Customer Communication**
>
> When a customer's system sends you bad data, how you communicate the error matters enormously. A raw `ValidationError` dump with JSON paths and error codes is not customer-friendly. Build a translation layer that converts Pydantic's machine-readable errors into plain English: "Field 'shipTo.postalCode': value '1234' is not a valid US ZIP code (must be 5 digits). You sent: '1234'." This is the difference between a customer filing a ticket and a customer fixing their data in 5 minutes.

> **FDE Context: Model Your Contracts, Not Your Assumptions**
>
> The first model you write for a customer integration will be wrong. Not because you're bad at your job — because the customer's API docs don't match what their system actually sends. Always add `extra="ignore"` to start, log the extra fields, and after a week of production data, you'll know what their system actually sends. Then tighten the model.

> **FDE Context: Version Your Models**
>
> When a customer upgrades their ERP, the payload changes. Have a strategy for this before you go live. Use Pydantic discriminated unions or separate model versions (`OrderV1`, `OrderV2`) that map to the same internal representation. Never have the customer's breaking change break your integration silently.

---

## Common Failure Modes

| Symptom | Cause | Fix |
|---|---|---|
| `ValidationError: field required` for a field that exists | Field alias mismatch — JSON has `orderId`, model has `order_id` but no alias | Add `alias="orderId"` or use `alias_generator=to_camel` |
| Model accepts wrong values silently | `extra="ignore"` silently drops validation failures too? No — check if you have a default | Add `mode="strict"` to validators or remove defaults |
| `float` field getting `"12.5 lbs"` — works once, fails with `"12.5kg"` | Validator only handles known units | Make validator exhaustive; log unknown formats |
| Nested model validation error has deeply nested path | Hard to surface to customers | Flatten the error: `" → ".join(str(l) for l in error["loc"])` |
| `datetime` deserialization fails from some customers | They send in different timezone formats | Use `field_validator` with multiple format attempts |
| Pydantic model is slow under high load | Too many validators, large nested models | Use `model_config = ConfigDict(revalidate_instances="never")`, cache repeated validates |
| `RecursionError` in recursive model | Forgot `model_rebuild()` after self-reference | Call `MyModel.model_rebuild()` after class definition |

---

## Interview Angle

**Q: "What's the difference between `field_validator` and `model_validator`?"**

Great answer: "`field_validator` operates on a single field in isolation — it gets just that field's value. `model_validator` has access to the entire model's data, either before field validation (mode='before', gets a dict) or after (mode='after', gets the model instance). I use `field_validator` for single-field coercion like normalizing a carrier name, and `model_validator` for cross-field validation like 'ship date can't be after delivery date' or for remapping fields from a legacy schema before field-level processing runs."

**Q: "How do you handle a customer API that changes its payload format?"**

Great answer: "I use `extra='ignore'` and `populate_by_name=True` to be resilient. For major schema changes, I use a model versioning strategy — either discriminated unions based on a version field, or separate model classes (`OrderV1`, `OrderV2`) that both map to the same internal canonical model. I also always log the raw payload on validation failure, so I can diagnose what changed."

**Q: "A customer sends you dates in 5 different formats. How do you handle that in Pydantic?"**

Great answer: "I write a `field_validator` with `mode='before'` that tries each format using `datetime.strptime()` in a loop. I keep a list of known formats ordered by specificity — ISO first, then regional variants. When none match, I raise a `ValueError` with a message that shows what format we expected and what we received. I also log the format that succeeded, which helps me see if a customer changed their system."

---

## Practice Exercise

**Scenario:** A 3PL customer sends you a CSV export from their WMS. Here's a sample of the messy data:

```csv
order_id,carrier,tracking,weight,ship_date,priority,items_count,declared_value
ORD001,Federal Express,1Z 999 AA1 01 2345 678,12.5 lbs,01/15/2024,HIGH,3,150.00
ORD002,ups,1z999AA1012345679,8.2,2024-01-16,urgant,1,
ORD003,USPS,9400111899223399744295,1.1 kg,Jan 17 2024,,2,25
ORD004,dhl,1234567890,0.5,20240118,low,5,N/A
ORD005,,bad-tracking,INVALID,,CRITICAL,0,
```

**Task 1:** Write a Pydantic model `CSVOrderRow` that handles all the messiness in this data — multiple weight formats, multiple date formats, carrier name normalization, priority normalization, null-equivalent strings for declared_value, and validates that items_count > 0.

**Task 2:** Write a function `parse_csv_orders(csv_string: str) -> tuple[list[CSVOrderRow], list[dict]]` that returns valid rows and a list of error dicts (one per failed row, with row number and human-readable error message).

**Task 3:** Add a model-level validator that rejects rows where the tracking number format doesn't match the carrier.

**Task 4:** Write a test that verifies ORD003 parses correctly (weight converted from kg, date parsed from "Jan 17 2024" format, carrier normalized to "USPS").

---

*Previous: [01-async-await.md](./01-async-await.md) | Next: [03-pytest-testing.md](./03-pytest-testing.md)*
