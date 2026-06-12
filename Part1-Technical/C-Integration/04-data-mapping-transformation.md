# Data Mapping and Transformation

**FDE Course | Part 1: Technical | Section C: Integration**

> **FDE Principle**: Data mapping is where integrations fail at 3am. Not because the logic is complex, but because every edge case is invisible until production. A field that was always a string suddenly arrives as an integer. An address that was always present is now null. A date format changes mid-rollout. The patterns in this file turn invisible failures into observable, debuggable events.

---

## Table of Contents

1. [The Mapping Architecture](#mapping-architecture)
2. [Canonical Schema Design](#canonical-schema)
3. [Field Mapping Patterns](#field-mapping-patterns)
4. [Type Coercion](#type-coercion)
5. [Handling Missing and Null Data](#null-handling)
6. [Nested ↔ Flat Transformations](#nested-flat)
7. [Array and List Transformations](#array-transformations)
8. [Data Quality Validation](#data-quality)
9. [Diff and Delta Detection](#diff-delta)
10. [Pydantic as a Mapping Engine](#pydantic-mapping)
11. [Real Logistics Schema: Shopify → Carrier](#real-logistics-schema)
12. [10 Real Edge Cases from Logistics Data](#edge-cases)
13. [Error Logging for 3am Debugging](#error-logging)
14. [Common Failure Modes](#common-failure-modes)
15. [Interview Angles](#interview-angles)
16. [Practice Exercise](#practice-exercise)

---

## The Mapping Architecture {#mapping-architecture}

### The three-schema pattern

```
Source System          Canonical Schema          Target System
(Shopify Order)     (Internal Standard)         (Carrier Booking)
      │                      │                         │
      │   source mapper       │   target mapper          │
      └──────────────────────►│─────────────────────────►│
                              │
                    "Lingua franca" of your integration
                    
Benefits:
1. Add a new source? Only write source→canonical mapper
2. Add a new target? Only write canonical→target mapper
3. N sources × M targets = N+M mappers (not N×M)
4. Business logic lives in canonical schema, not scattered
5. Canonical schema is where you validate — once
```

### When to use canonical vs direct mapping

```
Direct mapping (source → target directly):
✓ Simple 1:1 integration (one source, one target, won't grow)
✓ POC/prototype where simplicity > maintainability
✗ Multiple sources or targets
✗ Business logic in the transformation

Three-schema (source → canonical → target):
✓ Multiple carriers, multiple systems
✓ Complex business rules
✓ Long-lived integration that will evolve
```

---

## Canonical Schema Design {#canonical-schema}

```python
from __future__ import annotations
from dataclasses import dataclass, field
from datetime import datetime
from decimal import Decimal
from enum import Enum
from typing import Optional
from pydantic import BaseModel, Field, field_validator, model_validator


# ─── Canonical address ────────────────────────────────────────────────────────

class CanonicalAddress(BaseModel):
    """
    Canonical address representation.
    
    Design decisions:
    - street2 is Optional (not all addresses have it)
    - postal_code is a STRING (not int) — "02134" would lose leading zero as int
    - country_code is ISO 3166-1 alpha-2 (US, GB, DE) — not "United States"
    - state_province_code is optional — not all countries have states
    
    Why not use country_name?
    Shopify sends "United States", FedEx expects "US", DHL expects "USA".
    Canonical uses ISO code — each mapper handles the translation.
    """
    name: str
    company: Optional[str] = None
    street1: str
    street2: Optional[str] = None
    city: str
    state_province_code: Optional[str] = None  # US: "CA", "NY" etc.
    postal_code: str
    country_code: str  # ISO 3166-1 alpha-2: "US", "GB", "DE"
    phone: Optional[str] = None
    email: Optional[str] = None


# ─── Canonical package ────────────────────────────────────────────────────────

class WeightUnit(str, Enum):
    LB = "lb"
    KG = "kg"
    OZ = "oz"
    G = "g"


class DimensionUnit(str, Enum):
    IN = "in"
    CM = "cm"
    MM = "mm"


class CanonicalPackage(BaseModel):
    """
    Physical package dimensions in canonical units.
    
    Store in declared units but always include the unit.
    Don't convert at storage time — convert at target-mapping time.
    Why? Conversion introduces rounding error. Convert once, at output.
    """
    weight: Decimal
    weight_unit: WeightUnit
    length: Optional[Decimal] = None
    width: Optional[Decimal] = None
    height: Optional[Decimal] = None
    dimension_unit: Optional[DimensionUnit] = None
    declared_value: Optional[Decimal] = None
    currency_code: Optional[str] = None  # ISO 4217: "USD", "EUR"
    description: Optional[str] = None


# ─── Canonical line item ──────────────────────────────────────────────────────

class CanonicalLineItem(BaseModel):
    sku: Optional[str] = None
    name: str
    quantity: int
    unit_price: Decimal
    currency_code: str
    weight: Optional[Decimal] = None
    weight_unit: Optional[WeightUnit] = None
    hs_code: Optional[str] = None   # for customs
    country_of_origin: Optional[str] = None  # ISO alpha-2


# ─── Canonical shipment request ───────────────────────────────────────────────

class ServiceLevel(str, Enum):
    GROUND = "ground"
    EXPRESS = "express"
    OVERNIGHT = "overnight"
    ECONOMY = "economy"
    STANDARD = "standard"


class CanonicalShipmentRequest(BaseModel):
    """
    The canonical representation of a shipment booking request.
    
    This is the central object everything transforms into and out of.
    """
    # Identifiers
    order_id: str
    external_order_id: Optional[str] = None  # customer's reference
    
    # Addresses
    ship_from: CanonicalAddress
    ship_to: CanonicalAddress
    
    # Package
    packages: list[CanonicalPackage] = Field(min_length=1)
    
    # Line items (for customs, value declaration)
    line_items: list[CanonicalLineItem] = Field(default_factory=list)
    
    # Service preferences
    service_level: ServiceLevel = ServiceLevel.STANDARD
    carrier_code: Optional[str] = None  # None = let rate-shop decide
    carrier_service_code: Optional[str] = None
    
    # Options
    signature_required: bool = False
    insurance_value: Optional[Decimal] = None
    saturday_delivery: bool = False
    residential_delivery: Optional[bool] = None  # None = auto-detect
    
    # Metadata
    created_at: datetime = Field(default_factory=lambda: datetime.now())
    metadata: dict = Field(default_factory=dict)
```

---

## Field Mapping Patterns {#field-mapping-patterns}

### The DataMapper base class

```python
import logging
from typing import Any, Callable, TypeVar, Generic
from pydantic import BaseModel

logger = logging.getLogger(__name__)

T = TypeVar("T", bound=BaseModel)
S = TypeVar("S")


class MappingError(Exception):
    """Raised when a field cannot be mapped."""
    def __init__(
        self,
        message: str,
        field_name: str,
        source_value: Any,
        record_id: Optional[str] = None,
    ):
        super().__init__(message)
        self.field_name = field_name
        self.source_value = source_value
        self.record_id = record_id
    
    def __str__(self):
        return (
            f"Mapping error on field '{self.field_name}': {super().__str__()} "
            f"(value={self.source_value!r}, record_id={self.record_id})"
        )


class FieldMapper:
    """
    Toolkit for common field mapping operations.
    Each method handles a specific transformation pattern.
    """
    
    @staticmethod
    def direct(source: dict, source_key: str, required: bool = False) -> Any:
        """Direct copy: source['key'] → value, with optional requirement."""
        value = source.get(source_key)
        if required and value is None:
            raise MappingError(
                f"Required field missing",
                field_name=source_key,
                source_value=None,
            )
        return value
    
    @staticmethod
    def rename(source: dict, source_key: str, default: Any = None) -> Any:
        """Copy from source with different key name."""
        return source.get(source_key, default)
    
    @staticmethod
    def nested_get(source: dict, *keys: str, default: Any = None) -> Any:
        """
        Navigate nested dict: nested_get(order, "shipping", "address", "city")
        Returns default at any missing level.
        """
        current = source
        for key in keys:
            if not isinstance(current, dict):
                return default
            current = current.get(key)
            if current is None:
                return default
        return current
    
    @staticmethod
    def transform(source: dict, source_key: str, fn: Callable, default: Any = None) -> Any:
        """Apply transformation function to a value."""
        value = source.get(source_key)
        if value is None:
            return default
        try:
            return fn(value)
        except Exception as e:
            raise MappingError(
                f"Transform failed: {e}",
                field_name=source_key,
                source_value=value,
            )
    
    @staticmethod
    def conditional(source: dict, conditions: list[tuple[str, Any, Any]], default: Any = None) -> Any:
        """
        Conditional mapping: try keys in order, return first non-None.
        conditions: [(key_to_try, value_if_present, fallback)]
        """
        for key, if_fn, else_fn in conditions:
            value = source.get(key)
            if value is not None:
                return if_fn(value) if callable(if_fn) else if_fn
        return default
    
    @staticmethod
    def lookup(value: Any, mapping: dict, default: Any = None, strict: bool = False) -> Any:
        """
        Value substitution via lookup table.
        
        strict=True: raise error if value not in mapping
        strict=False: return default if value not in mapping (and log warning)
        """
        result = mapping.get(value, default)
        if result is None and strict:
            raise MappingError(
                f"No mapping defined for value",
                field_name="lookup",
                source_value=value,
            )
        if result is None and value is not None:
            logger.warning(
                "Lookup miss — using default",
                extra={"value": value, "mapping_keys": list(mapping.keys())[:10]},
            )
        return result


# ─── Country code lookup tables ───────────────────────────────────────────────

COUNTRY_NAME_TO_ISO2 = {
    "United States": "US",
    "United States of America": "US",
    "USA": "US",
    "Canada": "CA",
    "United Kingdom": "GB",
    "Great Britain": "GB",
    "UK": "GB",
    "Germany": "DE",
    "Deutschland": "DE",
    "France": "FR",
    "Australia": "AU",
    "Japan": "JP",
    "China": "CN",
    "Mexico": "MX",
    # ... add as needed
}

CARRIER_SERVICE_CODES = {
    "fedex": {
        "ground": "FEDEX_GROUND",
        "express": "FEDEX_2_DAY",
        "overnight": "PRIORITY_OVERNIGHT",
        "economy": "SMART_POST",
    },
    "ups": {
        "ground": "UPS_GROUND",
        "express": "UPS_2ND_DAY_AIR",
        "overnight": "UPS_NEXT_DAY_AIR",
        "economy": "UPS_SUREPOST",
    },
    "usps": {
        "ground": "USPS_PARCEL_SELECT",
        "express": "USPS_PRIORITY",
        "overnight": "USPS_EXPRESS",
        "economy": "USPS_MEDIA_MAIL",
    },
}
```

---

## Type Coercion {#type-coercion}

### String → datetime (multiple formats — the real challenge)

```python
from datetime import datetime, timezone, date
from typing import Optional
import re


# Date formats encountered in the wild (logistics data)
DATE_FORMATS = [
    "%Y-%m-%dT%H:%M:%S.%fZ",       # 2024-01-15T14:30:00.123Z (ISO 8601 UTC)
    "%Y-%m-%dT%H:%M:%SZ",           # 2024-01-15T14:30:00Z
    "%Y-%m-%dT%H:%M:%S%z",          # 2024-01-15T14:30:00+05:30 (with offset)
    "%Y-%m-%dT%H:%M:%S",            # 2024-01-15T14:30:00 (no timezone — assume UTC)
    "%Y-%m-%d %H:%M:%S",            # 2024-01-15 14:30:00 (space separator)
    "%Y-%m-%d",                      # 2024-01-15 (date only)
    "%m/%d/%Y %H:%M:%S",            # 01/15/2024 14:30:00 (US format)
    "%m/%d/%Y",                      # 01/15/2024 (US date only)
    "%d/%m/%Y",                      # 15/01/2024 (European format — AMBIGUOUS with US!)
    "%d-%m-%Y",                      # 15-01-2024 (another European)
    "%Y%m%d",                        # 20240115 (compact date, often in EDI)
    "%d %b %Y",                      # 15 Jan 2024
    "%B %d, %Y",                     # January 15, 2024
    "%b %d, %Y",                     # Jan 15, 2024
]


def parse_datetime_flexible(
    value: Any,
    field_name: str = "date",
    default_tz: timezone = timezone.utc,
    ambiguous_format: str = "mdy",  # "mdy" or "dmy" for ambiguous MM/DD vs DD/MM
) -> Optional[datetime]:
    """
    Parse a date/time string using multiple format attempts.
    
    The ambiguous format problem:
    "01/05/2024" — is this January 5th or May 1st?
    In US data: Jan 5. In EU data: May 1.
    The 'ambiguous_format' parameter handles this.
    
    Returns None if value is None, raises MappingError if unparseable.
    """
    if value is None:
        return None
    
    if isinstance(value, datetime):
        # Already a datetime — ensure it has timezone info
        if value.tzinfo is None:
            return value.replace(tzinfo=default_tz)
        return value
    
    if isinstance(value, date):
        return datetime(value.year, value.month, value.day, tzinfo=default_tz)
    
    if isinstance(value, (int, float)):
        # Unix timestamp — common in some APIs
        return datetime.fromtimestamp(value, tz=default_tz)
    
    if not isinstance(value, str):
        raise MappingError(
            f"Cannot parse datetime from {type(value).__name__}",
            field_name=field_name,
            source_value=value,
        )
    
    # Strip whitespace and common noise
    cleaned = value.strip()
    
    # Try Python 3.11+ fromisoformat first (handles most ISO 8601 variants)
    try:
        dt = datetime.fromisoformat(cleaned)
        if dt.tzinfo is None:
            dt = dt.replace(tzinfo=default_tz)
        return dt
    except ValueError:
        pass
    
    # Try each format
    for fmt in DATE_FORMATS:
        try:
            dt = datetime.strptime(cleaned, fmt)
            if dt.tzinfo is None:
                dt = dt.replace(tzinfo=default_tz)
            return dt
        except ValueError:
            continue
    
    raise MappingError(
        f"Unrecognized date format: {cleaned!r}",
        field_name=field_name,
        source_value=value,
    )


def parse_decimal_flexible(
    value: Any,
    field_name: str = "number",
    allow_negative: bool = True,
) -> Optional[Decimal]:
    """
    Parse a number from various string representations.
    
    Handles:
    - "$1,234.56" (US currency format)
    - "1.234,56" (European number format)
    - "1 234.56" (space thousands separator)
    - "1234.56 USD" (with currency suffix)
    - "1,234" (thousands only, no cents)
    - "1.5 lbs" (with unit suffix)
    - 1234 (integer)
    - 1234.56 (float)
    """
    if value is None:
        return None
    
    if isinstance(value, Decimal):
        return value
    
    if isinstance(value, (int, float)):
        return Decimal(str(value))
    
    if not isinstance(value, str):
        raise MappingError(
            f"Cannot parse number from {type(value).__name__}",
            field_name=field_name,
            source_value=value,
        )
    
    cleaned = value.strip()
    
    # Remove currency symbols and common prefixes
    cleaned = re.sub(r'^[£$€¥₹\s]+', '', cleaned)
    
    # Remove currency codes and unit suffixes
    cleaned = re.sub(r'\s*(USD|EUR|GBP|CAD|AUD|lbs?|kg|oz|g|cm|in|mm)\s*$', '', cleaned, flags=re.IGNORECASE)
    
    cleaned = cleaned.strip()
    
    # Detect European format (comma as decimal separator)
    # "1.234,56" → European → "1234.56"
    # "1,234.56" → US → "1234.56"
    if '.' in cleaned and ',' in cleaned:
        dot_pos = cleaned.rfind('.')
        comma_pos = cleaned.rfind(',')
        if comma_pos > dot_pos:
            # Last separator is comma → European decimal: 1.234,56
            cleaned = cleaned.replace('.', '').replace(',', '.')
        else:
            # Last separator is dot → US decimal: 1,234.56
            cleaned = cleaned.replace(',', '')
    elif ',' in cleaned and '.' not in cleaned:
        # Could be: "1,234" (US thousands) or "1,5" (European decimal)
        # Heuristic: if exactly one comma with 2 decimal places → decimal
        parts = cleaned.split(',')
        if len(parts) == 2 and len(parts[1]) <= 2:
            cleaned = cleaned.replace(',', '.')  # treat as decimal
        else:
            cleaned = cleaned.replace(',', '')  # treat as thousands separator
    
    # Remove remaining whitespace (space thousands separator: "1 234")
    cleaned = cleaned.replace(' ', '')
    
    try:
        result = Decimal(cleaned)
        if not allow_negative and result < 0:
            raise MappingError(
                "Negative value not allowed",
                field_name=field_name,
                source_value=value,
            )
        return result
    except Exception:
        raise MappingError(
            f"Cannot parse as number: {cleaned!r}",
            field_name=field_name,
            source_value=value,
        )


def parse_bool_flexible(value: Any, field_name: str = "bool") -> Optional[bool]:
    """
    Parse boolean from various representations.
    
    Covers:
    - True/False (Python bool)
    - "true"/"false" (JSON-style)
    - "Y"/"N" (legacy systems, EDI)
    - "yes"/"no"
    - "1"/"0" (some systems)
    - 1/0 (integer)
    """
    if value is None:
        return None
    
    if isinstance(value, bool):
        return value
    
    if isinstance(value, int):
        return bool(value)
    
    if isinstance(value, str):
        normalized = value.strip().lower()
        if normalized in ("true", "yes", "y", "1", "on", "enabled", "active"):
            return True
        if normalized in ("false", "no", "n", "0", "off", "disabled", "inactive"):
            return False
        raise MappingError(
            f"Cannot interpret as boolean: {value!r}",
            field_name=field_name,
            source_value=value,
        )
    
    raise MappingError(
        f"Cannot parse bool from {type(value).__name__}",
        field_name=field_name,
        source_value=value,
    )


def normalize_phone(value: Optional[str], country_code: str = "US") -> Optional[str]:
    """
    Normalize phone numbers to E.164 format (+15551234567).
    Handles:
    - "(555) 123-4567" (US format)
    - "555-123-4567"
    - "5551234567"
    - "+1 555 123 4567"
    """
    if not value:
        return None
    
    # Remove all non-digit characters except leading +
    digits_only = re.sub(r'[^\d+]', '', value.strip())
    
    # Already E.164
    if digits_only.startswith('+'):
        return digits_only
    
    # US/Canada number
    if country_code in ("US", "CA"):
        if len(digits_only) == 10:
            return f"+1{digits_only}"
        elif len(digits_only) == 11 and digits_only.startswith('1'):
            return f"+{digits_only}"
    
    # Return cleaned without country code (best effort)
    return digits_only if digits_only else None
```

---

## Handling Missing and Null Data {#null-handling}

```python
from typing import TypeVar, Optional, Callable, Any

T = TypeVar("T")


class NullHandlingStrategies:
    """
    Different strategies for handling missing/null data.
    Choose based on the semantic meaning of the field.
    """
    
    @staticmethod
    def require(value: Any, field_name: str, record_id: Optional[str] = None) -> Any:
        """
        Hard requirement: value MUST be present.
        Use for: order_id, ship_to address, weight.
        """
        if value is None or value == "":
            raise MappingError(
                "Required field is null or empty",
                field_name=field_name,
                source_value=value,
                record_id=record_id,
            )
        return value
    
    @staticmethod
    def default_to(value: Any, default: T) -> T:
        """
        Use default if null. 
        Use for: signature_required (default False), insurance (default None).
        """
        if value is None or value == "":
            return default
        return value
    
    @staticmethod
    def warn_if_missing(
        value: Any,
        field_name: str,
        record_id: Optional[str] = None,
        default: Any = None,
    ) -> Any:
        """
        Use default but log a warning.
        Use for: phone number, company name — useful but not critical.
        """
        if value is None or value == "":
            logger.warning(
                "Expected field is missing",
                extra={"field_name": field_name, "record_id": record_id},
            )
            return default
        return value
    
    @staticmethod
    def coerce_empty_to_none(value: Any) -> Optional[Any]:
        """
        Normalize empty string, "null", "NULL", "None" to Python None.
        
        Many legacy systems use empty strings or the string "null"
        to represent absent values.
        """
        if value is None:
            return None
        if isinstance(value, str):
            normalized = value.strip().lower()
            if normalized in ("", "null", "none", "n/a", "na", "-", "unknown"):
                return None
        return value
    
    @staticmethod
    def first_non_null(*values: Any) -> Optional[Any]:
        """
        Return first non-null, non-empty value from multiple sources.
        
        Use when the same concept appears in multiple source fields:
        first_non_null(order["billing_phone"], order["contact_phone"], order["phone"])
        """
        for v in values:
            cleaned = NullHandlingStrategies.coerce_empty_to_none(v)
            if cleaned is not None:
                return cleaned
        return None


# Postal code validation and completion
POSTAL_CODE_PATTERNS = {
    "US": re.compile(r'^\d{5}(-\d{4})?$'),
    "CA": re.compile(r'^[A-Z]\d[A-Z]\s?\d[A-Z]\d$', re.IGNORECASE),
    "GB": re.compile(r'^[A-Z]{1,2}\d[A-Z\d]?\s?\d[A-Z]{2}$', re.IGNORECASE),
    "DE": re.compile(r'^\d{5}$'),
    "AU": re.compile(r'^\d{4}$'),
}

def validate_and_format_postal_code(
    postal_code: Optional[str],
    country_code: str,
    field_name: str = "postal_code",
) -> Optional[str]:
    """
    Validate postal code format for country and normalize.
    
    Common issues:
    - US "02134" stored as integer → becomes "2134" (loses leading zero)
    - UK postcodes with spaces removed: "SW1A 1AA" vs "SW1A1AA"
    - Canadian postal codes lowercase: "m5v3c6" vs "M5V 3C6"
    """
    if postal_code is None:
        return None
    
    # Convert to string (handles case where stored as int)
    postal_str = str(postal_code).strip().upper()
    
    # Country-specific normalization
    if country_code == "US":
        # Pad to 5 digits if it was stored as integer (e.g., 2134 → 02134)
        if re.match(r'^\d{1,4}$', postal_str):
            postal_str = postal_str.zfill(5)
        # Strip ZIP+4 if present
        postal_str = postal_str[:5]
    
    elif country_code == "CA":
        # Normalize to "A1B 2C3" format
        digits_letters = postal_str.replace(' ', '')
        if len(digits_letters) == 6:
            postal_str = f"{digits_letters[:3]} {digits_letters[3:]}"
    
    elif country_code == "GB":
        # UK postcodes should have a space
        digits_letters = postal_str.replace(' ', '')
        if len(digits_letters) >= 5:
            postal_str = f"{digits_letters[:-3]} {digits_letters[-3:]}"
    
    # Validate against pattern
    pattern = POSTAL_CODE_PATTERNS.get(country_code)
    if pattern and not pattern.match(postal_str):
        logger.warning(
            "Postal code format may be invalid",
            extra={
                "postal_code": postal_str,
                "country_code": country_code,
                "original": postal_code,
            }
        )
        # Don't raise — carriers sometimes accept slightly wrong formats
        # Log and continue
    
    return postal_str
```

---

## Nested ↔ Flat Transformations {#nested-flat}

```python
from typing import Generator


def flatten_dict(
    d: dict,
    separator: str = ".",
    prefix: str = "",
) -> dict:
    """
    Flatten a nested dictionary.
    
    {"address": {"city": "NY", "state": "NY"}, "id": 1}
    → {"address.city": "NY", "address.state": "NY", "id": 1}
    
    Use case: some systems send nested JSON, your DB expects flat columns.
    """
    result = {}
    
    for key, value in d.items():
        new_key = f"{prefix}{separator}{key}" if prefix else key
        
        if isinstance(value, dict):
            result.update(flatten_dict(value, separator=separator, prefix=new_key))
        elif isinstance(value, list):
            # Lists: flatten with index
            for i, item in enumerate(value):
                item_key = f"{new_key}[{i}]"
                if isinstance(item, dict):
                    result.update(flatten_dict(item, separator=separator, prefix=item_key))
                else:
                    result[item_key] = item
        else:
            result[new_key] = value
    
    return result


def unflatten_dict(d: dict, separator: str = ".") -> dict:
    """
    Reverse of flatten_dict.
    {"address.city": "NY", "address.state": "NY"} → {"address": {"city": "NY", "state": "NY"}}
    """
    result = {}
    
    for key, value in d.items():
        parts = key.split(separator)
        current = result
        
        for part in parts[:-1]:
            if part not in current:
                current[part] = {}
            current = current[part]
        
        current[parts[-1]] = value
    
    return result


def unroll_nested_to_flat_records(
    records: list[dict],
    parent_key: str,
    child_array_key: str,
    child_prefix: str = "",
) -> list[dict]:
    """
    Unroll nested arrays into flat records (for SQL-style output).
    
    Input:
    [
      {"order_id": "O1", "line_items": [{"sku": "A", "qty": 2}, {"sku": "B", "qty": 1}]},
      {"order_id": "O2", "line_items": [{"sku": "C", "qty": 3}]},
    ]
    
    Output (order_id + line_item for each):
    [
      {"order_id": "O1", "sku": "A", "qty": 2},
      {"order_id": "O1", "sku": "B", "qty": 1},
      {"order_id": "O2", "sku": "C", "qty": 3},
    ]
    """
    result = []
    
    for record in records:
        parent_fields = {k: v for k, v in record.items() if k != child_array_key}
        children = record.get(child_array_key, [])
        
        if not children:
            # Parent with no children — include with empty child fields
            result.append({**parent_fields})
        else:
            for child in children:
                if child_prefix:
                    child_flat = {f"{child_prefix}{k}": v for k, v in child.items()}
                else:
                    child_flat = child
                result.append({**parent_fields, **child_flat})
    
    return result


def build_nested_from_flat(
    flat_rows: list[dict],
    parent_id_key: str,
    child_fields: list[str],
    children_key: str = "items",
) -> list[dict]:
    """
    Reverse: group flat rows into parent-child nested structure.
    
    Input (flat DB rows):
    [
      {"order_id": "O1", "sku": "A", "qty": 2},
      {"order_id": "O1", "sku": "B", "qty": 1},
      {"order_id": "O2", "sku": "C", "qty": 3},
    ]
    
    Output (nested):
    [
      {"order_id": "O1", "items": [{"sku": "A", "qty": 2}, {"sku": "B", "qty": 1}]},
      {"order_id": "O2", "items": [{"sku": "C", "qty": 3}]},
    ]
    """
    from collections import defaultdict, OrderedDict
    
    parents: dict[str, dict] = OrderedDict()
    
    for row in flat_rows:
        parent_id = row[parent_id_key]
        
        if parent_id not in parents:
            # Create parent record with non-child fields
            parents[parent_id] = {
                k: v for k, v in row.items()
                if k not in child_fields
            }
            parents[parent_id][children_key] = []
        
        # Add child record
        child = {k: row[k] for k in child_fields if k in row}
        parents[parent_id][children_key].append(child)
    
    return list(parents.values())
```

---

## Array and List Transformations {#array-transformations}

```python
from typing import Iterable


def filter_nulls(items: list) -> list:
    """Remove None and empty-string items from list."""
    return [item for item in items if item is not None and item != ""]


def deduplicate(items: list, key_fn: Callable = None) -> list:
    """
    Remove duplicates, preserving order.
    key_fn: function to extract comparison key (default: identity)
    """
    seen = set()
    result = []
    
    for item in items:
        key = key_fn(item) if key_fn else item
        if key not in seen:
            seen.add(key)
            result.append(item)
    
    return result


def group_by(items: list[dict], key: str) -> dict[str, list[dict]]:
    """
    Group list of dicts by a field value.
    group_by(shipments, "carrier_code") → {"FEDEX": [...], "UPS": [...]}
    """
    from collections import defaultdict
    result = defaultdict(list)
    for item in items:
        result[item[key]].append(item)
    return dict(result)


def safe_sum(items: list, field: str, coerce=Decimal) -> Decimal:
    """
    Sum a field across list items, skipping nulls.
    Returns Decimal(0) for empty/all-null lists.
    """
    total = Decimal("0")
    for item in items:
        value = item.get(field)
        if value is not None:
            try:
                total += coerce(str(value))
            except Exception:
                logger.warning(f"Cannot sum field {field}: {value!r}")
    return total


def aggregate_line_items(line_items: list[dict]) -> dict:
    """
    Common logistics aggregation: summarize line items.
    Returns total quantity, total value, unique SKU count.
    """
    total_qty = sum(item.get("quantity", 0) for item in line_items)
    total_value = safe_sum(line_items, "unit_price") 
    unique_skus = len({item.get("sku") for item in line_items if item.get("sku")})
    
    return {
        "total_quantity": total_qty,
        "total_value": total_value,
        "line_item_count": len(line_items),
        "unique_sku_count": unique_skus,
    }
```

---

## Data Quality Validation {#data-quality}

```python
from pydantic import BaseModel, ValidationError
from typing import Any
import logging

logger = logging.getLogger(__name__)


class ValidationResult(BaseModel):
    is_valid: bool
    errors: list[dict] = []
    warnings: list[dict] = []
    record_id: Optional[str] = None


def validate_shipment_request(
    data: dict,
    record_id: Optional[str] = None,
) -> ValidationResult:
    """
    Post-mapping validation before sending to carrier.
    
    Why validate after mapping, not before?
    Mapping may fix issues (normalize postal code, add defaults).
    We validate the RESULT of mapping, not the source.
    """
    errors = []
    warnings = []
    
    # Required fields
    ship_to = data.get("ship_to", {})
    
    if not ship_to.get("street1"):
        errors.append({"field": "ship_to.street1", "message": "Required — missing street address"})
    
    if not ship_to.get("city"):
        errors.append({"field": "ship_to.city", "message": "Required — missing city"})
    
    if not ship_to.get("postal_code"):
        errors.append({"field": "ship_to.postal_code", "message": "Required — missing postal code"})
    
    country = ship_to.get("country_code", "")
    if len(country) != 2:
        errors.append({
            "field": "ship_to.country_code",
            "message": f"Must be ISO 3166-1 alpha-2 (2 chars), got: {country!r}",
        })
    
    # Package validation
    packages = data.get("packages", [])
    if not packages:
        errors.append({"field": "packages", "message": "At least one package required"})
    
    for i, pkg in enumerate(packages):
        weight = pkg.get("weight")
        if weight is None:
            errors.append({"field": f"packages[{i}].weight", "message": "Weight required"})
        elif Decimal(str(weight)) <= 0:
            errors.append({"field": f"packages[{i}].weight", "message": f"Weight must be positive, got {weight}"})
        elif Decimal(str(weight)) > 150 and pkg.get("weight_unit") == "lb":
            warnings.append({
                "field": f"packages[{i}].weight",
                "message": f"Weight {weight}lb exceeds standard carrier limit of 150lb — may require special handling",
            })
    
    # Phone number warning (not required but useful)
    phone = ship_to.get("phone")
    if not phone:
        warnings.append({
            "field": "ship_to.phone",
            "message": "Phone recommended for delivery notifications",
        })
    
    # Residential delivery warning
    if not data.get("residential_delivery") and country == "US":
        warnings.append({
            "field": "residential_delivery",
            "message": "residential_delivery not set — may affect rate accuracy",
        })
    
    # Log any errors with full context
    if errors:
        logger.error(
            "Shipment validation failed",
            extra={
                "record_id": record_id,
                "error_count": len(errors),
                "errors": errors,
                "data_excerpt": {
                    "ship_to_city": ship_to.get("city"),
                    "ship_to_country": ship_to.get("country_code"),
                    "package_count": len(packages),
                },
            }
        )
    
    if warnings:
        logger.warning(
            "Shipment validation warnings",
            extra={
                "record_id": record_id,
                "warning_count": len(warnings),
                "warnings": warnings,
            }
        )
    
    return ValidationResult(
        is_valid=len(errors) == 0,
        errors=errors,
        warnings=warnings,
        record_id=record_id,
    )
```

---

## Diff and Delta Detection {#diff-delta}

```python
import hashlib
import json
from typing import TypeVar

T = TypeVar("T")


def compute_hash(data: dict, fields: Optional[list[str]] = None) -> str:
    """
    Compute a stable hash of a dict for change detection.
    
    fields: if provided, only hash these fields (ignore others)
    
    Use case: sync scenario where you only want to re-send records that changed.
    Hash the record, compare to stored hash, skip if unchanged.
    """
    if fields:
        subset = {k: data[k] for k in fields if k in data}
    else:
        subset = data
    
    # Use sorted keys for stable ordering
    canonical_json = json.dumps(subset, sort_keys=True, default=str)
    return hashlib.sha256(canonical_json.encode()).hexdigest()


class DeltaDetector:
    """
    Detect which records have changed since last sync.
    
    Pattern:
    1. On each sync, compute hash of each record
    2. Compare to stored hashes from previous sync
    3. Only process changed/new records
    4. Update stored hashes after successful processing
    
    This prevents re-sending 50,000 unchanged records every hour.
    """
    
    def __init__(self, storage: dict = None):
        # In production: use Redis or a DB table for persistence
        self._hashes: dict[str, str] = storage or {}
    
    def get_changed_records(
        self,
        records: list[dict],
        id_field: str,
        hash_fields: Optional[list[str]] = None,
    ) -> tuple[list[dict], list[dict], list[str]]:
        """
        Compare records to stored hashes.
        
        Returns:
        - new_records: records not seen before
        - changed_records: records where hash changed
        - unchanged_ids: record IDs that haven't changed (skip these)
        """
        new_records = []
        changed_records = []
        unchanged_ids = []
        
        for record in records:
            record_id = str(record.get(id_field))
            current_hash = compute_hash(record, hash_fields)
            stored_hash = self._hashes.get(record_id)
            
            if stored_hash is None:
                new_records.append(record)
            elif stored_hash != current_hash:
                changed_records.append(record)
            else:
                unchanged_ids.append(record_id)
        
        return new_records, changed_records, unchanged_ids
    
    def mark_processed(
        self,
        records: list[dict],
        id_field: str,
        hash_fields: Optional[list[str]] = None,
    ) -> None:
        """Update stored hashes after successful processing."""
        for record in records:
            record_id = str(record.get(id_field))
            self._hashes[record_id] = compute_hash(record, hash_fields)
    
    def get_deleted_ids(
        self,
        current_ids: set[str],
    ) -> set[str]:
        """
        Find records that were in the last sync but aren't anymore.
        These have been deleted in the source system.
        """
        return set(self._hashes.keys()) - current_ids
```

---

## Pydantic as a Mapping Engine {#pydantic-mapping}

```python
from pydantic import BaseModel, field_validator, model_validator, Field
from typing import Any


class ShopifyOrderAddress(BaseModel):
    """
    Source schema: Shopify's address format.
    """
    first_name: Optional[str]
    last_name: Optional[str]
    company: Optional[str]
    address1: str
    address2: Optional[str]
    city: str
    province_code: Optional[str]  # "CA", "NY"
    country_code: str  # "US"
    zip: str  # postal code
    phone: Optional[str]


class ShopifyLineItem(BaseModel):
    id: int
    title: str
    sku: Optional[str]
    quantity: int
    price: str  # Shopify sends price as string!
    grams: int  # weight in grams


class ShopifyOrder(BaseModel):
    id: int
    name: str  # "#1001"
    created_at: str
    shipping_address: ShopifyOrderAddress
    billing_address: Optional[ShopifyOrderAddress]
    line_items: list[ShopifyLineItem]
    total_price: str
    currency: str


class CanonicalShipmentFromShopify(BaseModel):
    """
    Canonical schema populated from a Shopify order.
    Uses Pydantic validators as the transformation engine.
    """
    order_id: str
    ship_to: CanonicalAddress
    packages: list[CanonicalPackage]
    line_items_canonical: list[CanonicalLineItem]
    total_value: Decimal
    currency_code: str
    
    @classmethod
    def from_shopify_order(
        cls,
        order: ShopifyOrder,
        record_id: Optional[str] = None,
    ) -> "CanonicalShipmentFromShopify":
        """
        Transform a Shopify order into canonical shipment request.
        
        Handles all the edge cases:
        - Name field (split first+last)
        - Weight in grams → lbs
        - Price as string → Decimal
        - Missing phone
        """
        addr = order.shipping_address
        
        # Combine first+last name
        name_parts = filter(None, [addr.first_name, addr.last_name])
        full_name = " ".join(name_parts) or addr.company or "Unknown"
        
        ship_to = CanonicalAddress(
            name=full_name,
            company=addr.company,
            street1=addr.address1,
            street2=addr.address2 or None,
            city=addr.city,
            state_province_code=addr.province_code,
            postal_code=validate_and_format_postal_code(addr.zip, addr.country_code),
            country_code=addr.country_code.upper(),
            phone=normalize_phone(addr.phone, addr.country_code),
        )
        
        # Convert line items to canonical + compute total weight
        canonical_line_items = []
        total_grams = 0
        
        for item in order.line_items:
            unit_price = parse_decimal_flexible(item.price, f"line_items[{item.id}].price")
            canonical_line_items.append(
                CanonicalLineItem(
                    sku=item.sku,
                    name=item.title,
                    quantity=item.quantity,
                    unit_price=unit_price,
                    currency_code=order.currency,
                )
            )
            total_grams += item.grams * item.quantity
        
        # Convert total weight grams → lbs for carrier API
        total_lbs = Decimal(str(total_grams)) / Decimal("453.592")
        
        # Minimum package weight
        package_weight = max(total_lbs, Decimal("0.1"))
        
        packages = [
            CanonicalPackage(
                weight=package_weight.quantize(Decimal("0.01")),
                weight_unit=WeightUnit.LB,
                declared_value=parse_decimal_flexible(order.total_price, "total_price"),
                currency_code=order.currency,
            )
        ]
        
        return cls(
            order_id=str(order.id),
            ship_to=ship_to,
            packages=packages,
            line_items_canonical=canonical_line_items,
            total_value=parse_decimal_flexible(order.total_price, "total_price"),
            currency_code=order.currency,
        )
```

---

## Real Logistics Schema: Shopify → Carrier {#real-logistics-schema}

```python
class FedExBookingRequest(BaseModel):
    """
    Target schema: FedEx Ship API v1.
    """
    labelResponseOptions: str = "LABEL"
    requestedShipment: dict
    accountNumber: dict


def map_canonical_to_fedex(
    canonical: CanonicalShipmentRequest,
    fedex_account_number: str,
    fedex_meter_number: str,
) -> FedExBookingRequest:
    """
    Map canonical shipment request → FedEx-specific booking request.
    
    This is the target mapper — handles FedEx-specific quirks:
    - FedEx wants weight in LB (convert from canonical if KG)
    - FedEx service codes differ from canonical service levels
    - FedEx address has specific field names
    - FedEx uses specific date format for ship date
    """
    
    # Weight conversion
    pkg = canonical.packages[0]
    if pkg.weight_unit == WeightUnit.KG:
        weight_lb = float(pkg.weight * Decimal("2.20462"))
    elif pkg.weight_unit == WeightUnit.LB:
        weight_lb = float(pkg.weight)
    else:
        raise MappingError(
            f"Cannot convert {pkg.weight_unit} to LB for FedEx",
            field_name="weight_unit",
            source_value=pkg.weight_unit,
        )
    
    # Service code mapping
    service_map = {
        ServiceLevel.GROUND: "FEDEX_GROUND",
        ServiceLevel.EXPRESS: "FEDEX_2_DAY",
        ServiceLevel.OVERNIGHT: "PRIORITY_OVERNIGHT",
        ServiceLevel.ECONOMY: "SMART_POST",
        ServiceLevel.STANDARD: "FEDEX_GROUND",
    }
    service_code = service_map.get(canonical.service_level, "FEDEX_GROUND")
    
    def format_fedex_address(addr: CanonicalAddress) -> dict:
        return {
            "streetLines": [addr.street1] + ([addr.street2] if addr.street2 else []),
            "city": addr.city,
            "stateOrProvinceCode": addr.state_province_code,
            "postalCode": addr.postal_code,
            "countryCode": addr.country_code,
        }
    
    return FedExBookingRequest(
        labelResponseOptions="LABEL",
        requestedShipment={
            "shipper": {
                "contact": {
                    "personName": canonical.ship_from.name,
                    "phoneNumber": canonical.ship_from.phone or "5551234567",
                },
                "address": format_fedex_address(canonical.ship_from),
            },
            "recipients": [{
                "contact": {
                    "personName": canonical.ship_to.name,
                    "phoneNumber": canonical.ship_to.phone or "",
                    "companyName": canonical.ship_to.company or "",
                },
                "address": {
                    **format_fedex_address(canonical.ship_to),
                    "residential": canonical.residential_delivery or False,
                },
            }],
            "pickupType": "USE_SCHEDULED_PICKUP",
            "serviceType": service_code,
            "packagingType": "YOUR_PACKAGING",
            "requestedPackageLineItems": [{
                "weight": {
                    "units": "LB",
                    "value": round(weight_lb, 1),
                },
                "dimensions": {
                    "length": int(canonical.packages[0].length or 12),
                    "width": int(canonical.packages[0].width or 12),
                    "height": int(canonical.packages[0].height or 6),
                    "units": "IN",
                } if canonical.packages[0].length else None,
            }],
            "shippingChargesPayment": {
                "paymentType": "SENDER",
                "payor": {
                    "responsibleParty": {
                        "accountNumber": {"value": fedex_account_number},
                    }
                },
            },
            "shipDatestamp": datetime.now().strftime("%Y-%m-%d"),
        },
        accountNumber={"value": fedex_account_number},
    )
```

---

## 10 Real Edge Cases from Logistics Data {#edge-cases}

```python
"""
Real edge cases encountered in logistics integrations.
Each has a comment explaining where it was found in the wild.
"""

def handle_edge_cases():
    
    # ─── Edge Case 1: Postal code as integer ──────────────────────────────────
    # Shopify sometimes sends zip as integer in their bulk export CSVs
    # "02134" → 2134 (integer) → "2134" (wrong!) → needs zero-padding
    postal_code_int = 2134
    postal_str = str(postal_code_int).zfill(5)  # "02134" ✓
    
    
    # ─── Edge Case 2: Country codes that aren't ISO 2 ─────────────────────────
    # Some ERPs use 3-letter codes (GBR), some use full names (United Kingdom)
    COUNTRY_CODE_ALIASES = {
        "USA": "US", "GBR": "GB", "CAN": "CA", "DEU": "DE",
        "FRA": "FR", "AUS": "AU", "JPN": "JP", "CHN": "CN",
        "MEX": "MX", "BRA": "BR", "IND": "IN", "ITA": "IT",
        "ESP": "ES", "NLD": "NL", "CHE": "CH", "SWE": "SE",
    }
    
    def normalize_country_code(code: str) -> str:
        if len(code) == 2:
            return code.upper()
        if len(code) == 3:
            return COUNTRY_CODE_ALIASES.get(code.upper(), code.upper()[:2])
        # Full name — try lookup table
        return COUNTRY_NAME_TO_ISO2.get(code, code[:2].upper())
    
    
    # ─── Edge Case 3: Weight as string with units attached ────────────────────
    # "5.2 lbs", "2.4kg", "16 oz" — all valid in different source systems
    def parse_weight_with_unit(value: str) -> tuple[Decimal, WeightUnit]:
        """Returns (weight, unit)"""
        match = re.match(r'^([\d.]+)\s*(lbs?|kg|oz|g)$', value.strip(), re.IGNORECASE)
        if not match:
            raise MappingError("Cannot parse weight", "weight", value)
        
        amount = Decimal(match.group(1))
        unit_str = match.group(2).lower().rstrip('s')  # "lbs" → "lb"
        
        unit_map = {"lb": WeightUnit.LB, "kg": WeightUnit.KG, "oz": WeightUnit.OZ, "g": WeightUnit.G}
        unit = unit_map.get(unit_str, WeightUnit.LB)
        
        return amount, unit
    
    
    # ─── Edge Case 4: Address without state for non-US countries ──────────────
    # UK, Germany, Japan don't have state codes — but some ERPs put city in state field
    # UK: county is optional. Germany has no states in shipping context.
    def normalize_address_fields(addr: dict, country_code: str) -> dict:
        if country_code not in ("US", "CA", "AU", "MX", "BR"):
            # Countries without required state codes
            addr = addr.copy()
            addr["state_province_code"] = None  # Don't send empty string
        return addr
    
    
    # ─── Edge Case 5: Duplicate line items in Shopify (split by fulfillment) ──
    # Shopify can send the same SKU in multiple line items if it's been partially fulfilled
    def merge_duplicate_line_items(items: list[dict]) -> list[dict]:
        """Merge items with same SKU (summing quantities)."""
        merged = {}
        for item in items:
            sku = item.get("sku") or item.get("name")  # fallback to name if no SKU
            if sku in merged:
                merged[sku]["quantity"] += item["quantity"]
            else:
                merged[sku] = item.copy()
        return list(merged.values())
    
    
    # ─── Edge Case 6: Price in minor currency units ───────────────────────────
    # Some APIs send price in cents (Stripe): 1500 → $15.00
    # Others send in major units (Shopify): "15.00" → $15.00
    # Key: check the API docs for each integration
    def normalize_price(value: Any, is_minor_unit: bool = False) -> Decimal:
        """
        is_minor_unit=True: value is in cents/pence/etc (divide by 100)
        is_minor_unit=False: value is in major units ($15.00)
        """
        amount = parse_decimal_flexible(value, "price")
        if is_minor_unit:
            return amount / Decimal("100")
        return amount
    
    
    # ─── Edge Case 7: Multi-line street address in a single field ─────────────
    # Some systems store "123 Main St\nApt 4" in one address field
    def split_address_line(combined: str) -> tuple[str, Optional[str]]:
        """Split a combined address field into street1, street2."""
        if not combined:
            return "", None
        
        # Split on newline, "\n", or "|"
        parts = re.split(r'\n|\\n|\|', combined, maxsplit=1)
        
        if len(parts) == 2:
            return parts[0].strip(), parts[1].strip() or None
        return combined.strip(), None
    
    
    # ─── Edge Case 8: Phone numbers with extensions ───────────────────────────
    # "555-123-4567 x234" or "(555) 123-4567 ext 234"
    def extract_phone_and_extension(value: str) -> tuple[str, Optional[str]]:
        """Returns (clean_phone, extension_or_None)"""
        ext_match = re.search(r'(?:ext?\.?|x)\s*(\d+)', value, re.IGNORECASE)
        extension = ext_match.group(1) if ext_match else None
        
        # Remove extension from number string
        if ext_match:
            phone_part = value[:ext_match.start()].strip()
        else:
            phone_part = value
        
        return normalize_phone(phone_part), extension
    
    
    # ─── Edge Case 9: Inconsistent boolean representations ────────────────────
    # Seen in ERP exports: "Y", "N", "YES", "NO", "1", "0", "TRUE", "FALSE", "T", "F"
    # Use parse_bool_flexible from the type coercion section above
    
    # ─── Edge Case 10: Timezone-naive datetimes from legacy systems ───────────
    # Oracle, SAP often export datetime without timezone in their own local time
    # You must know the customer's timezone to interpret correctly
    import zoneinfo
    
    def parse_legacy_datetime(
        value: str,
        customer_timezone: str = "America/Chicago",
    ) -> datetime:
        """
        Parse a datetime string that has no timezone info.
        Assumes it's in the customer's local timezone.
        
        Example: customer is in Chicago, their ERP outputs "2024-01-15 14:30:00"
        This means 14:30 Central Time, which is 20:30 UTC.
        """
        naive_dt = datetime.fromisoformat(value)
        local_tz = zoneinfo.ZoneInfo(customer_timezone)
        
        # Attach the customer's timezone (not convert — it WAS this timezone)
        local_dt = naive_dt.replace(tzinfo=local_tz)
        
        # Convert to UTC for storage
        return local_dt.astimezone(timezone.utc)
```

---

## Error Logging for 3am Debugging {#error-logging}

> **FDE Context**: When an integration fails at 3am, you need to diagnose the problem from logs alone. Design your logs so every mapping error tells you: what field failed, what value caused the failure, which record it was, and the full context.

```python
import json
import traceback
import logging
from contextlib import contextmanager
from typing import Generator

logger = logging.getLogger(__name__)


class MappingContext:
    """
    Context manager that adds structured context to all mapping errors.
    
    Use this to wrap the mapping of each record so that every error
    includes the record identifier and whatever context helps debugging.
    """
    
    def __init__(self, record_id: str, source_system: str, target_system: str):
        self.record_id = record_id
        self.source_system = source_system
        self.target_system = target_system
        self.warnings: list[dict] = []
    
    def warn(self, field: str, message: str, value: Any = None) -> None:
        """Record a non-fatal mapping warning."""
        warning = {
            "field": field,
            "message": message,
            "value": value,
            "record_id": self.record_id,
        }
        self.warnings.append(warning)
        logger.warning(
            "Mapping warning",
            extra={**warning, "source": self.source_system, "target": self.target_system},
        )


@contextmanager
def mapping_context(
    record_id: str,
    source_data: dict,
    source_system: str,
    target_system: str,
) -> Generator[MappingContext, None, None]:
    """
    Context manager for mapping a single record.
    
    On success: logs completion with timing.
    On failure: logs full context including the source data excerpt.
    
    Usage:
        with mapping_context("ORD-001", order_data, "shopify", "fedex") as ctx:
            result = map_order_to_shipment(order_data)
            ctx.warn("phone", "Missing phone number")
    """
    import time
    ctx = MappingContext(record_id, source_system, target_system)
    start = time.monotonic()
    
    try:
        yield ctx
        elapsed = time.monotonic() - start
        
        logger.debug(
            "Record mapped successfully",
            extra={
                "record_id": record_id,
                "source_system": source_system,
                "target_system": target_system,
                "warning_count": len(ctx.warnings),
                "elapsed_ms": round(elapsed * 1000, 1),
            }
        )
    
    except MappingError as e:
        elapsed = time.monotonic() - start
        
        # Extract a small excerpt of source data for debugging
        # Don't log the full payload (may be large/sensitive)
        # Log the fields most likely to help debug
        source_excerpt = {
            "id": source_data.get("id"),
            "order_id": source_data.get("order_id") or source_data.get("name"),
            "created_at": source_data.get("created_at"),
            # Include the failing field's parent object
            "failing_field_context": _extract_field_context(source_data, e.field_name),
        }
        
        logger.error(
            "Record mapping failed",
            extra={
                "record_id": record_id,
                "source_system": source_system,
                "target_system": target_system,
                "field_name": e.field_name,
                "field_value": repr(e.source_value),
                "error_message": str(e),
                "source_excerpt": source_excerpt,
                "elapsed_ms": round(elapsed * 1000, 1),
            }
        )
        raise
    
    except Exception as e:
        elapsed = time.monotonic() - start
        
        logger.error(
            "Unexpected error during record mapping",
            extra={
                "record_id": record_id,
                "source_system": source_system,
                "target_system": target_system,
                "error_type": type(e).__name__,
                "error_message": str(e),
                "traceback": traceback.format_exc(),
                "elapsed_ms": round(elapsed * 1000, 1),
            }
        )
        raise


def _extract_field_context(source: dict, field_name: str) -> dict:
    """
    Extract the immediate parent object of a failing field for log context.
    "shipping.address.postal_code" → return the address dict
    """
    parts = field_name.replace("[", ".").replace("]", "").split(".")
    
    current = source
    for part in parts[:-1]:
        if isinstance(current, dict):
            current = current.get(part, {})
        elif isinstance(current, list):
            try:
                current = current[int(part)]
            except (IndexError, ValueError):
                return {}
    
    if isinstance(current, dict):
        return {k: v for k, v in current.items() if not isinstance(v, (list, dict))}
    return {}


# Full pipeline with logging
def process_batch_with_logging(
    records: list[dict],
    source_system: str,
    target_system: str,
) -> tuple[list[dict], list[str]]:
    """
    Process a batch of records, logging each mapping attempt.
    Returns (successful_results, failed_record_ids).
    
    Never throws — failed records go to failed list, processing continues.
    This is critical: one bad record should not stop 10,000 good ones.
    """
    results = []
    failed = []
    
    for record in records:
        record_id = str(record.get("id") or record.get("order_id") or "unknown")
        
        try:
            with mapping_context(record_id, record, source_system, target_system):
                result = map_record(record)  # your mapping function
                results.append(result)
        
        except (MappingError, ValueError) as e:
            # Expected mapping failures — logged in context manager
            failed.append(record_id)
        
        except Exception as e:
            # Unexpected errors — logged in context manager
            failed.append(record_id)
    
    logger.info(
        "Batch mapping complete",
        extra={
            "total": len(records),
            "successful": len(results),
            "failed": len(failed),
            "failure_rate": f"{len(failed)/len(records)*100:.1f}%" if records else "0%",
            "failed_ids": failed[:20],  # log first 20, not all (could be huge)
        }
    )
    
    return results, failed


def map_record(record: dict) -> dict:
    """Placeholder — replace with your actual mapping logic."""
    return {}
```

---

## Common Failure Modes {#common-failure-modes}

### 1. Postal code integer truncation

**Symptom**: Shipments to Massachusetts (02xxx), Connecticut (06xxx), New Jersey (07xxx, 08xxx) fail address validation.

**Root cause**: Postal code stored or transmitted as integer, leading zero lost.

**Fix**: Always coerce postal codes to string with zero-padding for US addresses.

### 2. Date format ambiguity in international data

**Symptom**: Shipment date is 12 days in the future (or past) for European customers.

**Root cause**: "05/06/2024" — US reads May 6th, EU reads June 5th.

**Fix**: Establish canonical date format (ISO 8601) in your canonical schema. Require source mappers to handle the ambiguity based on known source format.

### 3. Silent null propagation

**Symptom**: Shipments created with city = "None" or state = "null" (literal strings).

**Root cause**: `str(None)` = "None", not empty string.

**Fix**: Always use `NullHandlingStrategies.coerce_empty_to_none()` before checking for null.

### 4. Weight unit assumption

**Symptom**: FedEx rejects shipment with "weight exceeds maximum" — but the package weighs 2lb.

**Root cause**: Source sends weight in grams (2000), your code assumes it's already in lbs, FedEx receives 2000lb.

**Fix**: Canonical schema requires explicit weight_unit field — never assume.

### 5. Currency in minor units

**Symptom**: Insurance value or declared value is 100x too high.

**Root cause**: Source sends price in cents (Stripe, Square), your code treats it as dollars.

**Fix**: Document unit assumption per data source. Add validation: declared_value > 10000 USD is suspicious for small parcel.

---

## Interview Angles {#interview-angles}

### Q: "Walk me through how you'd design a data mapping pipeline for a new carrier integration."

**Great answer**: "I start with the three-schema pattern: source schema, canonical schema, target schema. I define the canonical schema first — it's the contract everything maps through. Then I write a source mapper that takes the carrier's raw response and produces a canonical object, validating required fields as it goes. The target mapper takes canonical and produces the format the downstream system expects. The key operational decision is: what happens when a field can't be mapped? I never silently drop errors — I use a mapping context manager that logs the failing field, the source value, and the record ID. Bad records go to a dead letter queue with enough context to replay them manually. One bad record should never stop a 10,000-record batch."

### Q: "A field that was always a string is suddenly arriving as a number. How do you handle that?"

**Great answer**: "First, I'd add defensive type coercion at the source mapper level — coerce everything to the canonical type rather than assuming the source type. The canonical schema defines the type contract; the source mapper's job is to bridge the gap. In practice: wrap all field extractions in coerce functions that handle common type variations. Then I'd add a validator that logs a warning whenever the type doesn't match the expected type, even after coercion succeeds — this surfaces the schema change so you can track it. In the longer term, I'd add a schema version header to API requests and handle both the old and new type in the mapper with a deprecation warning."

### Q: "How do you handle a data sync where you don't want to re-send unchanged records?"

**Great answer**: "Delta detection via hash comparison. On each sync cycle, I compute a SHA-256 hash of each record's content (using only the fields I care about — not metadata like updated_at which changes even when data doesn't). I compare to the stored hash from the previous sync. Only records with changed hashes get processed and sent downstream. The hashes are stored in Redis or a DB table keyed on record ID. After successful processing, I update the stored hash. The edge case to handle: deletions — records that were in the last sync but aren't in the current one have been deleted in the source. I track which IDs were in the last sync and compute the diff."

---

## Practice Exercise {#practice-exercise}

### Exercise: Build the Shopify → Multi-Carrier Mapper

**Scenario**: You're building the data mapping layer for a multi-carrier shipping platform. Orders come from Shopify. Shipments go to FedEx, UPS, or USPS depending on rate-shopping results.

**Data files** (create these as test fixtures):

```python
# test_order_1.json — clean order
shopify_order_clean = {
    "id": 5001,
    "name": "#1001",
    "created_at": "2024-01-15T14:30:00-05:00",
    "currency": "USD",
    "total_price": "89.99",
    "shipping_address": {
        "first_name": "Jane",
        "last_name": "Smith",
        "company": None,
        "address1": "123 Main St",
        "address2": "Apt 4B",
        "city": "Boston",
        "province_code": "MA",
        "country_code": "US",
        "zip": "02134",     # ← integer storage issue
        "phone": "(617) 555-0123",
    },
    "line_items": [
        {"id": 1, "title": "Blue Widget", "sku": "WGT-001", "quantity": 2, "price": "29.99", "grams": 500},
        {"id": 2, "title": "Red Gadget", "sku": "GDG-002", "quantity": 1, "price": "30.01", "grams": 750},
    ],
}

# test_order_2.json — edge cases
shopify_order_edge_cases = {
    "id": 5002,
    "name": "#1002",
    "created_at": "2024-01-15T19:30:00.000Z",
    "currency": "USD",
    "total_price": "45.00",
    "shipping_address": {
        "first_name": "John",
        "last_name": None,       # ← missing last name
        "company": "ACME Corp",
        "address1": "456 Oak Ave\nSuite 200",  # ← multi-line in one field
        "address2": None,
        "city": "New York",
        "province_code": "NY",
        "country_code": "US",
        "zip": 10001,            # ← integer zip
        "phone": None,           # ← missing phone
    },
    "line_items": [
        {"id": 3, "title": "Green Thing", "sku": None, "quantity": 3, "price": "15.00", "grams": 0},
    ],
}
```

**Requirements**:

1. `ShopifyToCanonicalMapper` — maps Shopify order → `CanonicalShipmentRequest`
   - Handles all 10 edge cases above
   - Uses `mapping_context` for logging
   - Returns warnings for: missing phone, zero weight, no SKU

2. `CanonicalToFedExMapper` — maps canonical → FedEx API format
3. `CanonicalToUPSMapper` — maps canonical → UPS API format

4. `validate_canonical_shipment(shipment)` — post-mapping validation
5. Test file that processes both test orders and asserts:
   - Order 1: successfully maps, phone normalized to E.164
   - Order 2: maps with warnings (missing phone, zero weight), address split correctly
   - Both: postal codes are strings with correct zero-padding

**Evaluation criteria**:
- Does zero-weight in order 2 produce a warning, not an error?
- Is order 2's multi-line address correctly split into street1/street2?
- Is the integer zip in order 2 correctly padded to "10001"?
- Are all MappingErrors logged with field_name and source_value?
- Does a batch with one bad record still process the other records?
- Does the logging include enough context to debug without re-running?

---

## Related Files

- [REST, GraphQL, and Webhooks](01-rest-graphql-webhooks.md) — the HTTP layer that delivers these payloads
- [Resilience patterns](03-resilience-patterns.md) — what to do when mapping fails at scale (DLQ)
- [Legacy connectivity (CSV/Excel)](05-legacy-connectivity.md) — ingesting source data from files before mapping
- [EDI awareness](07-edi-awareness.md) — EDI → JSON transformation uses these same mapping patterns
