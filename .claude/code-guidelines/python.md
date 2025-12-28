# Python Code Guidelines

## Variable Declarations

### Direct Reference to Configuration Data

Quick description: Always reference configuration data and constants directly without creating intermediate variables.

✅ DO:

```python
result = await walmart_client.upload_product(
    sku=inventory_data["sku"],
    account_id=config.WALMART_ACCOUNTS[0]["id"],
    region=config.DEFAULT_REGION
)

product = session.query(Product).filter_by(
    sku=inventory_data["sku"]
).first()
```

❌ DON'T:

```python
account = config.WALMART_ACCOUNTS[0]  # Unnecessary intermediate variable

result = await walmart_client.upload_product(
    sku=inventory_data["sku"],
    account_id=account["id"],
    region=config.DEFAULT_REGION
)

product = session.query(Product).filter_by(
    sku=inventory_data["sku"]
).first()
```

### Exceptions

Variables may be declared separately if:

- They are used multiple times
- The expression is complex and would harm readability if used inline
- The variable name provides important semantic context that would be lost with inline usage
- Debugging purposes require a specific variable to be inspectable

## Class vs Function Definitions

Quick description: Prefer standalone functions over classes when possible. Use classes only for stateful objects or when inheritance is needed.

✅ DO:

```python
def convert_gigab2b_to_walmart(product_data: dict, schema_config: dict) -> dict:
    """Convert Gigab2b product data to Walmart format."""
    return {
        "sku": product_data["sku"],
        "title": product_data["name"],
        "price": product_data["price"]
    }
```

❌ DON'T:

```python
class ProductConverter:
    def convert_gigab2b_to_walmart(self, product_data: dict, schema_config: dict) -> dict:
        """Convert Gigab2b product data to Walmart format."""
        return {
            "sku": product_data["sku"],
            "title": product_data["name"],
            "price": product_data["price"]
        }
```

## Python Class Design Guidelines: The "No-Init" Pattern

Quick description: Separate data definition from object construction. Never put business logic, I/O, or side effects inside `__init__`.

### Core Principle

Separate **Data Definition** from **Object Construction**. Use dataclasses for state and factory methods for construction logic.

### Implementation Rules

1. **Use Dataclasses for State**: Define the class using `@dataclass`. Rely on the auto-generated `__init__` which only performs simple attribute assignment.
2. **Factory Methods for Logic**: If any calculation, validation, transformation, or I/O is needed to create the object, place this logic in a `@classmethod` factory (e.g., `create()`, `open()`, `from_config()`).
3. **Strong Typing**: Use `typing.NewType` for primitive attributes (like `int` or `str`) that have specific domain meanings (e.g., `FileDescriptor`, `UserId`) to ensure the dataclass only accepts valid data.

✅ DO:

```python
from dataclasses import dataclass
from typing import NewType, Self

# 1. Use NewType for domain safety
FileDescriptor = NewType("FileDescriptor", int)
UserId = NewType("UserId", str)

@dataclass
class FileReader:
    # 2. Dataclass handles state definition (pure, no logic)
    _fd: FileDescriptor

    # 3. Classmethod handles construction logic & side effects
    @classmethod
    def open(cls, path: str) -> Self:
        # Perform I/O or logic here
        raw_fd = some_io_library.open(path)
        # Pass clean, validated data to the default __init__
        return cls(FileDescriptor(raw_fd))

    def read(self) -> str:
        # Instance methods use the valid state
        return some_io_library.read(self._fd)

@dataclass
class User:
    user_id: UserId
    name: str
    email: str

    @classmethod
    def from_database(cls, user_id: str) -> Self:
        # Database query and validation happens here
        data = database.query("SELECT * FROM users WHERE id = ?", user_id)
        return cls(
            user_id=UserId(data["id"]),
            name=data["name"],
            email=data["email"]
        )

    @classmethod
    def create(cls, name: str, email: str) -> Self:
        # Validation and ID generation happens here
        if not email or "@" not in email:
            raise ValueError("Invalid email")
        user_id = UserId(generate_uuid())
        return cls(user_id=user_id, name=name, email=email)
```

❌ DON'T:

```python
class FileReader:
    def __init__(self, path: str):
        # Don't put I/O in __init__
        self._fd = some_io_library.open(path)

    def read(self) -> str:
        return some_io_library.read(self._fd)

class User:
    def __init__(self, name: str, email: str, user_id: str | None = None):
        # Don't put validation and business logic in __init__
        if not email or "@" not in email:
            raise ValueError("Invalid email")

        # Don't put ID generation in __init__
        self.user_id = user_id or generate_uuid()
        self.name = name
        self.email = email
```

## Type Annotations

Quick description: Always use type hints for function parameters and return values. Use `TypedDict` or `Pydantic` for structured dictionaries.

### Modern Type Syntax

Quick description: Use lowercase built-in types (`list`, `dict`, `tuple`) and union syntax (`str | None`) instead of typing module equivalents (`List`, `Dict`, `Optional`).

✅ DO:

```python
from typing import TypedDict

class ProductData(TypedDict):
    sku: str
    name: str
    price: float
    category_id: str | None

def process_inventory_data(products: list[ProductData], account_id: str) -> dict[str, int]:
    """Process inventory data and return upload results."""
    return {"uploaded": len(products), "failed": 0}

def get_product_categories(product_ids: list[str]) -> list[tuple[str, str | None]]:
    """Return list of tuples with product ID and optional category."""
    return [(pid, None) for pid in product_ids]
```

❌ DON'T:

```python
from typing import List, Dict, Optional, Tuple

def process_inventory_data(products: List[ProductData], account_id: str) -> Dict[str, int]:
    """Process inventory data and return upload results."""
    return {"uploaded": len(products), "failed": 0}

def get_product_categories(product_ids: List[str]) -> List[Tuple[str, Optional[str]]]:
    """Return list of tuples with product ID and optional category."""
    return [(pid, None) for pid in product_ids]

# Also avoid missing type hints
def process_inventory_data(products, account_id):
    """Process inventory data and return upload results."""
    return {"uploaded": len(products), "failed": 0}
```

## Pydantic Models

Quick description: Use `Annotated` with `Field` instead of assigning `Field` directly. This separates type information from field configuration and improves code readability.

✅ DO:

```python
from typing import Annotated
from pydantic import BaseModel, Field

class ProductModel(BaseModel):
    sku: Annotated[str, Field(description="产品SKU", min_length=1)]
    name: Annotated[str, Field(description="产品名称")]
    price: Annotated[float, Field(description="价格", gt=0)]
    category_id: Annotated[str | None, Field(default=None, description="分类ID")]
    is_active: Annotated[bool, Field(default=True, description="是否启用")]

class UploadConfig(BaseModel):
    batch_size: Annotated[int, Field(default=100, ge=1, le=1000, description="批次大小")]
    enable_llm: Annotated[bool, Field(default=False, description="启用LLM")]
    retry_count: Annotated[int, Field(default=3, ge=0, description="重试次数")]
```

❌ DON'T:

```python
from pydantic import BaseModel, Field

class ProductModel(BaseModel):
    sku: str = Field(description="产品SKU", min_length=1)
    name: str = Field(description="产品名称")
    price: float = Field(description="价格", gt=0)
    category_id: str | None = Field(default=None, description="分类ID")
    is_active: bool = Field(default=True, description="是否启用")

class UploadConfig(BaseModel):
    batch_size: int = Field(default=100, ge=1, le=1000, description="批次大小")
    enable_llm: bool = Field(default=False, description="启用LLM")
    retry_count: int = Field(default=3, ge=0, description="重试次数")
```

## Function Parameters

Quick description: Use keyword-only arguments for functions with more than 2 parameters. Group related parameters using `TypedDict`.

✅ DO:

```python
class UploadConfig(TypedDict):
    account_id: str
    batch_size: int
    enable_llm: bool
    region: str

def upload_products(*, products: list[ProductData], config: UploadConfig) -> None:
    """Upload products with specified configuration."""
    pass
```

❌ DON'T:

```python
def upload_products(products: list[ProductData], account_id: str, batch_size: int, enable_llm: bool, region: str) -> None:
    """Upload products with specified configuration."""
    pass
```

## Error Handling

Quick description: Follow "let it crash" philosophy. Only catch exceptions you can meaningfully handle. Avoid catching broad `Exception`.

✅ DO:

```python
def fetch_product_data(sku: str) -> dict:
    """Fetch product data, let API errors propagate."""
    response = requests.get(f"/api/products/{sku}")
    response.raise_for_status()  # Let HTTP errors crash
    return response.json()

def safe_parse_price(price_str: str) -> float | None:
    """Parse price string, return None if invalid."""
    try:
        return float(price_str)
    except ValueError:
        return None  # Only handle expected parsing errors
```

❌ DON'T:

```python
def fetch_product_data(sku: str) -> dict | None:
    """Fetch product data with broad exception handling."""
    try:
        response = requests.get(f"/api/products/{sku}")
        response.raise_for_status()
        return response.json()
    except Exception as e:  # Too broad, hides real issues
        logger.error(f"Failed to fetch product: {e}")
        return None
```

## Import Organization

Quick description: Group imports by type and sort alphabetically within groups. Use explicit imports over wildcards.

✅ DO:

```python
# Standard library
import json
import logging
from datetime import datetime
from typing import TypedDict

# Third-party
import httpx
import sqlalchemy
from pydantic import BaseModel

# Local imports
from common.config import get_config
from database.models import Product, Inventory
```

❌ DON'T:

```python
import json, logging, datetime  # Avoid multiple imports on one line
from typing import *  # Avoid wildcard imports
from typing import List, Dict, Optional  # Use lowercase built-ins instead
import httpx
from common.config import get_config
import sqlalchemy  # Unsorted imports
```

## Dictionary and List Operations

Quick description: Use dictionary comprehensions and built-in methods for data transformations. Prefer explicit key access over `.get()` when keys are guaranteed to exist.

✅ DO:

```python
# When keys are guaranteed to exist
processed_products = [
    {
        "sku": product["sku"],
        "title": product["name"],
        "price": product["price"]
    }
    for product in products
    if product["status"] == "active"
]

# When keys might be missing
safe_prices = [
    product.get("price", 0.0)
    for product in products
]
```

❌ DON'T:

```python
# Unnecessary .get() for guaranteed keys
processed_products = [
    {
        "sku": product.get("sku"),
        "title": product.get("name"),
        "price": product.get("price")
    }
    for product in products
    if product.get("status") == "active"
]
```

## Async/Await Patterns

Quick description: Use async/await consistently. Prefer `asyncio.gather()` for concurrent operations.

✅ DO:

```python
async def sync_multiple_accounts(account_ids: list[str]) -> dict[str, bool]:
    """Sync inventory for multiple accounts concurrently."""
    tasks = [
        sync_account_inventory(account_id)
        for account_id in account_ids
    ]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    
    return {
        account_id: not isinstance(result, Exception)
        for account_id, result in zip(account_ids, results)
    }
```

❌ DON'T:

```python
async def sync_multiple_accounts(account_ids: list[str]) -> dict[str, bool]:
    """Sync inventory for multiple accounts sequentially."""
    results = {}
    for account_id in account_ids:  # Sequential, not concurrent
        try:
            await sync_account_inventory(account_id)
            results[account_id] = True
        except Exception:
            results[account_id] = False
    return results
```

