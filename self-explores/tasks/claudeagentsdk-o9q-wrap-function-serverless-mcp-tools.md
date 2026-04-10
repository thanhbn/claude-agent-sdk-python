---
date: 2026-03-29
type: task-worklog
task: claudeagentsdk-o9q
title: "[MAS-2] Wrap function/serverless thành MCP tools"
status: open
detailed_at: 2026-03-29
detail_score: ready-for-dev
tags: [mas, mcp, wrapper, factory, async, sync, type-hints, pattern]
depends_on: [claudeagentsdk-5w2]
blocks: [MAS-4]
estimated_hours: 1.5
output_file: examples/mas/wrap_patterns.py
---

# [MAS-2] Wrap function/serverless thành MCP tools

## 1. Mục tiêu (Objective)

Xây dựng 3 wrapper patterns biến Python functions bình thường (sync và async) thành MCP tools hoạt động với `@tool` decorator, bao gồm 1 generic factory function `wrap_function()` có khả năng auto-generate `input_schema` từ type hints. Demo trên 3 domain functions tự tạo: tax calculator (sync), weather fetcher (async), data validator (sync). Tất cả chạy được end-to-end qua `ClaudeSDKClient`.

## 2. Phạm vi (Scope)

**Trong phạm vi (In-scope):**
- Pattern 1: Wrap sync function thành async `@tool` bằng `asyncio.to_thread()`
- Pattern 2: Wrap async function trực tiếp thành `@tool`
- Pattern 3: Generic factory `wrap_function()` tự động:
  - Phát hiện sync/async bằng `inspect.iscoroutinefunction()`
  - Generate `input_schema` JSON Schema từ type hints bằng `inspect.signature()` + `get_type_hints()`
  - Wrap sync functions bằng `asyncio.to_thread()`
  - Trả về `SdkMcpTool` instance sẵn sàng dùng với `create_sdk_mcp_server()`
- Demo 3 wrapped functions chạy trong `ClaudeSDKClient`
- Output file: `examples/mas/wrap_patterns.py`

**Ngoài phạm vi (Out-of-scope):**
- Wrap class methods hoặc classmethods
- Support cho complex types (List[str], Dict[str, int], Optional[T], Union types)
- Validate input data theo schema trước khi gọi function
- Wrap generator functions hoặc streaming functions
- Tích hợp Pydantic model hoặc TypedDict cho schema
- Decorator chaining (nhiều decorators trên 1 function)
- Wrapping functions có `*args` hoặc `**kwargs`

## 3. Đầu vào / Đầu ra (Input / Output)

**Đầu vào:**
- Kiến thức từ MAS-1 (task `claudeagentsdk-5w2`) về `@tool`, `create_sdk_mcp_server()`, `ClaudeSDKClient`
- SDK source: `src/claude_agent_sdk/__init__.py` -- đặc biệt `SdkMcpTool` dataclass và `tool` decorator
- Python stdlib: `inspect`, `typing`, `asyncio`

**Đầu ra:**
- `examples/mas/wrap_patterns.py` -- File Python chạy được, chứa:
  1. Pattern 1: Manual sync-to-async wrap + `@tool`
  2. Pattern 2: Direct async function wrap + `@tool`
  3. Pattern 3: `wrap_function()` factory
  4. 3 domain functions gốc (tax calculator, weather fetcher, data validator)
  5. Demo `main()` chạy cả 3 patterns qua `ClaudeSDKClient`

## 4. Phụ thuộc (Dependencies)

**Packages (Python):**
- `claude_agent_sdk` -- SDK chính (từ source `src/`)
- `inspect` -- stdlib, introspect function signatures và type hints
- `typing` -- stdlib, `get_type_hints()`
- `asyncio` -- stdlib, `to_thread()` cho sync wrapping
- `json` -- stdlib, serialize results

**Biến môi trường:**
- `ANTHROPIC_API_KEY` -- bắt buộc
- `PATH` -- chứa đường dẫn đến `claude` CLI binary

**Phụ thuộc task:**
- **MAS-1 (`claudeagentsdk-5w2`)** -- BẮT BUỘC. Cần hiểu `@tool` decorator, `create_sdk_mcp_server()`, `allowed_tools` naming convention, và error handling pattern trước khi xây wrapper.

## 5. Luồng xử lý (Flow)

### Bước 1: Imports và domain functions gốc (~10 phút)

Tạo 3 domain functions bình thường (chưa là MCP tools):

```python
import asyncio
import inspect
import json
from typing import get_type_hints

from claude_agent_sdk import (
    tool,
    create_sdk_mcp_server,
    ClaudeSDKClient,
    ClaudeAgentOptions,
    AssistantMessage,
    TextBlock,
    ToolUseBlock,
    ResultMessage,
    SdkMcpTool,
)


# === Domain Functions (CHƯA phải MCP tools) ===

def calculate_tax(income: float, tax_rate: float, deductions: float) -> float:
    """Tính thuế thu nhập: (income - deductions) * tax_rate / 100.

    Args:
        income: Thu nhập trước thuế (VND)
        tax_rate: Thuế suất phần trăm (VD: 10 cho 10%)
        deductions: Các khoản giảm trừ (VND)

    Returns:
        Số tiền thuế phải đóng (VND)
    """
    if income < 0:
        raise ValueError("Income cannot be negative")
    if tax_rate < 0 or tax_rate > 100:
        raise ValueError("Tax rate must be between 0 and 100")
    if deductions < 0:
        raise ValueError("Deductions cannot be negative")

    taxable = max(0, income - deductions)
    return taxable * tax_rate / 100


async def fetch_weather(city: str, units: str) -> dict:
    """Lấy thông tin thời tiết giả lập cho một thành phố.

    Args:
        city: Tên thành phố (VD: "Hanoi", "Ho Chi Minh")
        units: Đơn vị nhiệt độ ("celsius" hoặc "fahrenheit")

    Returns:
        Dict chứa thông tin thời tiết
    """
    # Giả lập async I/O (thay vì gọi API thật)
    await asyncio.sleep(0.1)

    weather_db = {
        "hanoi": {"temp_c": 28, "humidity": 75, "condition": "Partly cloudy"},
        "ho chi minh": {"temp_c": 33, "humidity": 80, "condition": "Thunderstorms"},
        "da nang": {"temp_c": 30, "humidity": 70, "condition": "Sunny"},
    }

    city_lower = city.lower()
    if city_lower not in weather_db:
        raise ValueError(f"City not found: {city}. Available: {list(weather_db.keys())}")

    data = weather_db[city_lower]
    temp = data["temp_c"]
    if units == "fahrenheit":
        temp = temp * 9 / 5 + 32

    return {
        "city": city,
        "temperature": temp,
        "units": units,
        "humidity": data["humidity"],
        "condition": data["condition"],
    }


def validate_data(data: str, schema_type: str) -> str:
    """Kiểm tra dữ liệu theo schema type đơn giản.

    Args:
        data: Chuỗi dữ liệu cần kiểm tra
        schema_type: Loại schema: "email", "phone_vn", "number"

    Returns:
        Chuỗi mô tả kết quả validation
    """
    import re

    if schema_type == "email":
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        if re.match(pattern, data):
            return f"Valid email: {data}"
        return f"Invalid email format: {data}"

    elif schema_type == "phone_vn":
        pattern = r'^(0|\+84)[0-9]{9,10}$'
        clean = data.replace(" ", "").replace("-", "")
        if re.match(pattern, clean):
            return f"Valid Vietnamese phone: {data}"
        return f"Invalid Vietnamese phone format: {data}"

    elif schema_type == "number":
        try:
            float(data)
            return f"Valid number: {data}"
        except ValueError:
            return f"Not a valid number: {data}"

    else:
        raise ValueError(f"Unknown schema_type: {schema_type}. Use: email, phone_vn, number")
```

**Kiểm tra:** 3 functions hoạt động độc lập: `calculate_tax(1000000, 10, 200000)` trả về `80000.0`, `validate_data("test@mail.com", "email")` trả về "Valid email".

### Bước 2: Pattern 1 -- Manual Sync-to-Async Wrap (~15 phút)

Wrap `calculate_tax` (sync) thành MCP tool bằng cách tạo async handler gọi `asyncio.to_thread()`:

```python
# === PATTERN 1: Sync function -> Async @tool (manual) ===
# Cách thủ công nhất: viết async handler wrapper, dùng asyncio.to_thread()
# để chạy sync function trong thread pool.

@tool(
    name="tax_calculator",
    description="Tính thuế thu nhập cá nhân Việt Nam. Nhận thu nhập, thuế suất (%), và các khoản giảm trừ.",
    input_schema={
        "type": "object",
        "properties": {
            "income": {
                "type": "number",
                "description": "Thu nhập trước thuế (VND)"
            },
            "tax_rate": {
                "type": "number",
                "description": "Thuế suất phần trăm (VD: 10 cho 10%)"
            },
            "deductions": {
                "type": "number",
                "description": "Tổng các khoản giảm trừ (VND)"
            }
        },
        "required": ["income", "tax_rate", "deductions"]
    }
)
async def tax_calculator_tool(args: dict) -> dict:
    """Async wrapper cho sync function calculate_tax()."""
    try:
        # asyncio.to_thread() chạy sync function trong thread pool
        # tránh block event loop
        result = await asyncio.to_thread(
            calculate_tax,
            income=args["income"],
            tax_rate=args["tax_rate"],
            deductions=args["deductions"],
        )
        return {
            "content": [{
                "type": "text",
                "text": (
                    f"Tax calculation:\n"
                    f"  Income: {args['income']:,.0f} VND\n"
                    f"  Tax rate: {args['tax_rate']}%\n"
                    f"  Deductions: {args['deductions']:,.0f} VND\n"
                    f"  Tax amount: {result:,.0f} VND"
                )
            }]
        }
    except ValueError as e:
        return {
            "content": [{"type": "text", "text": f"Validation error: {str(e)}"}],
            "is_error": True,
        }
    except Exception as e:
        return {
            "content": [{"type": "text", "text": f"Error: {str(e)}"}],
            "is_error": True,
        }
```

**Ưu điểm:** Toàn quyền kiểm soát error handling và output format.
**Nhược điểm:** Phải viết input_schema thủ công, phải map args dict sang function params.

**Kiểm tra:** `tax_calculator_tool` là `SdkMcpTool` instance. `tax_calculator_tool.name == "tax_calculator"`.

### Bước 3: Pattern 2 -- Direct Async Wrap (~10 phút)

Wrap `fetch_weather` (đã async sẵn) thành MCP tool -- không cần `to_thread()`:

```python
# === PATTERN 2: Async function -> @tool (direct) ===
# Đơn giản nhất: function đã async sẵn, chỉ cần wrap
# trong @tool handler với error handling.

@tool(
    name="weather_fetcher",
    description="Lấy thông tin thời tiết cho một thành phố Việt Nam. Hỗ trợ: Hanoi, Ho Chi Minh, Da Nang.",
    input_schema={
        "type": "object",
        "properties": {
            "city": {
                "type": "string",
                "description": "Tên thành phố (VD: Hanoi, Ho Chi Minh, Da Nang)"
            },
            "units": {
                "type": "string",
                "enum": ["celsius", "fahrenheit"],
                "description": "Đơn vị nhiệt độ"
            }
        },
        "required": ["city", "units"]
    }
)
async def weather_fetcher_tool(args: dict) -> dict:
    """Direct async wrapper cho fetch_weather()."""
    try:
        # Gọi trực tiếp vì fetch_weather() đã là async
        result = await fetch_weather(
            city=args["city"],
            units=args["units"],
        )
        return {
            "content": [{
                "type": "text",
                "text": (
                    f"Weather in {result['city']}:\n"
                    f"  Temperature: {result['temperature']}° {result['units']}\n"
                    f"  Humidity: {result['humidity']}%\n"
                    f"  Condition: {result['condition']}"
                )
            }]
        }
    except ValueError as e:
        return {
            "content": [{"type": "text", "text": f"Error: {str(e)}"}],
            "is_error": True,
        }
    except Exception as e:
        return {
            "content": [{"type": "text", "text": f"Error: {str(e)}"}],
            "is_error": True,
        }
```

**Ưu điểm:** Cực đơn giản nếu function đã async.
**Nhược điểm:** Vẫn phải viết input_schema thủ công.

**Kiểm tra:** `weather_fetcher_tool` là `SdkMcpTool` instance. `weather_fetcher_tool.name == "weather_fetcher"`.

### Bước 4: Pattern 3 -- Generic Factory `wrap_function()` (~30 phút)

Đây là pattern quan trọng nhất. Factory function nhận bất kỳ Python function có type hints và tự động tạo `SdkMcpTool`.

```python
# === PATTERN 3: Generic factory wrap_function() ===

# Mapping Python type -> JSON Schema type
PYTHON_TO_JSON_SCHEMA = {
    str: "string",
    int: "integer",
    float: "number",
    bool: "boolean",
}


def wrap_function(
    func,
    name: str | None = None,
    description: str | None = None,
) -> SdkMcpTool:
    """Biến bất kỳ Python function (sync hoặc async) có type hints thành SdkMcpTool.

    Auto-generates:
    - input_schema từ function signature + type hints
    - Async handler wrapper (dùng asyncio.to_thread nếu function là sync)
    - Error handling chuẩn MCP (is_error=True)

    Args:
        func: Python function cần wrap. PHẢI có type hints cho tất cả parameters.
        name: Tên tool (mặc định: func.__name__)
        description: Mô tả tool (mặc định: func.__doc__ hoặc f"Tool: {name}")

    Returns:
        SdkMcpTool instance sẵn sàng dùng với create_sdk_mcp_server()

    Hạn chế:
        - Chỉ hỗ trợ simple types: str, int, float, bool
        - Không hỗ trợ: Optional, Union, List, Dict, custom classes
        - Không hỗ trợ: *args, **kwargs
        - Return value được convert sang str nếu không phải str/dict
    """
    # Xác định tên và mô tả
    tool_name = name or func.__name__
    tool_description = description or func.__doc__ or f"Tool: {tool_name}"
    # Dọn dẹp docstring (bỏ indent thừa)
    if tool_description and "\n" in tool_description:
        lines = tool_description.strip().splitlines()
        tool_description = lines[0].strip()  # Chỉ lấy dòng đầu

    # Phân tích signature để tạo input_schema
    sig = inspect.signature(func)
    try:
        hints = get_type_hints(func)
    except Exception:
        hints = {}

    properties = {}
    required = []

    for param_name, param in sig.parameters.items():
        # Bỏ qua 'return' hint
        param_type = hints.get(param_name, str)  # default str nếu thiếu hint
        json_type = PYTHON_TO_JSON_SCHEMA.get(param_type, "string")

        properties[param_name] = {
            "type": json_type,
            "description": f"Parameter: {param_name} ({param_type.__name__})"
        }

        # Nếu parameter không có default value -> required
        if param.default is inspect.Parameter.empty:
            required.append(param_name)

    input_schema = {
        "type": "object",
        "properties": properties,
        "required": required,
    }

    # Detect sync vs async
    is_async = inspect.iscoroutinefunction(func)

    # Tạo async handler chuẩn MCP
    async def handler(args: dict) -> dict:
        try:
            # Map args dict sang keyword arguments
            kwargs = {k: args[k] for k in sig.parameters if k in args}

            if is_async:
                # Gọi trực tiếp nếu function đã async
                result = await func(**kwargs)
            else:
                # Wrap sync function bằng to_thread
                result = await asyncio.to_thread(func, **kwargs)

            # Convert result sang MCP response format
            if isinstance(result, dict):
                text = json.dumps(result, indent=2, ensure_ascii=False, default=str)
            else:
                text = str(result)

            return {"content": [{"type": "text", "text": text}]}

        except Exception as e:
            return {
                "content": [{"type": "text", "text": f"Error: {type(e).__name__}: {str(e)}"}],
                "is_error": True,
            }

    # Trả về SdkMcpTool instance (cùng type mà @tool decorator trả về)
    return SdkMcpTool(
        name=tool_name,
        description=tool_description,
        input_schema=input_schema,
        handler=handler,
    )
```

**Key decisions:**
1. `inspect.iscoroutinefunction(func)` -- phát hiện sync/async tại thời điểm wrap, không phải lúc gọi
2. `get_type_hints(func)` -- lấy type annotations đã resolved (xử lý forward references)
3. `inspect.signature(func)` -- lấy tên params và default values để xác định `required`
4. `asyncio.to_thread(func, **kwargs)` -- wrap sync trong thread pool, an toàn cho event loop
5. Trả về `SdkMcpTool` trực tiếp thay vì dùng `@tool` decorator -- cùng type, linh hoạt hơn

**Kiểm tra:** `wrap_function(calculate_tax)` trả về `SdkMcpTool` với:
- `.name == "calculate_tax"`
- `.input_schema["properties"]` có 3 keys: `income`, `tax_rate`, `deductions`
- `.input_schema["required"]` có 3 items (tất cả params đều required)
- `.handler` là async callable

### Bước 5: Wrap 3 domain functions bằng factory (~10 phút)

```python
# === Wrap tất cả domain functions bằng wrap_function() ===

# Sync function -> auto-wrapped với to_thread
tax_tool_auto = wrap_function(
    calculate_tax,
    name="auto_tax_calculator",
    description="[Auto-wrapped] Tính thuế thu nhập cá nhân Việt Nam",
)

# Async function -> auto-wrapped trực tiếp
weather_tool_auto = wrap_function(
    fetch_weather,
    name="auto_weather_fetcher",
    description="[Auto-wrapped] Lấy thời tiết cho thành phố VN",
)

# Sync function -> auto-wrapped (dùng default name từ __name__)
validator_tool_auto = wrap_function(
    validate_data,
    # name không truyền -> dùng "validate_data"
    # description không truyền -> dùng dòng đầu docstring
)

# In schema để verify
print("=== Auto-generated schemas ===")
for t in [tax_tool_auto, weather_tool_auto, validator_tool_auto]:
    print(f"\n{t.name}:")
    print(f"  description: {t.description}")
    print(f"  schema: {json.dumps(t.input_schema, indent=4)}")
    print(f"  is_async_original: {inspect.iscoroutinefunction(t.handler)}")
```

**Kiểm tra:** In ra 3 schemas đúng. `tax_tool_auto` có `income: number`, `tax_rate: number`, `deductions: number`. `weather_tool_auto` có `city: string`, `units: string`. `validator_tool_auto` có name `"validate_data"` và description từ docstring.

### Bước 6: Tạo server và chạy end-to-end (~15 phút)

```python
# === Setup SDK MCP Server với cả 3 patterns ===

# Server 1: Manual wrapped tools (Pattern 1 & 2)
manual_server = create_sdk_mcp_server(
    name="manual_tools",
    version="1.0.0",
    tools=[tax_calculator_tool, weather_fetcher_tool],
)

# Server 2: Auto-wrapped tools (Pattern 3)
auto_server = create_sdk_mcp_server(
    name="auto_tools",
    version="1.0.0",
    tools=[tax_tool_auto, weather_tool_auto, validator_tool_auto],
)

# Options cho ClaudeSDKClient
options = ClaudeAgentOptions(
    mcp_servers={
        "manual_tools": manual_server,
        "auto_tools": auto_server,
    },
    allowed_tools=[
        # Pattern 1 & 2 (manual)
        "mcp__manual_tools__tax_calculator",
        "mcp__manual_tools__weather_fetcher",
        # Pattern 3 (auto-wrapped)
        "mcp__auto_tools__auto_tax_calculator",
        "mcp__auto_tools__auto_weather_fetcher",
        "mcp__auto_tools__validate_data",
    ],
    permission_mode="bypassPermissions",
    max_turns=15,
)


async def main():
    """Test cả 3 patterns: manual sync wrap, manual async wrap, auto factory."""

    print("=" * 60)
    print("MAS-2: Testing 3 wrap patterns")
    print("=" * 60)

    client = ClaudeSDKClient(options=options)

    try:
        prompt = (
            "You have access to tools from two servers. Please test them ALL:\n\n"
            "FROM manual_tools server:\n"
            "1. tax_calculator: Calculate tax for income=50000000 VND, rate=10%, deductions=11000000 VND\n"
            "2. weather_fetcher: Get weather for Hanoi in celsius\n\n"
            "FROM auto_tools server:\n"
            "3. auto_tax_calculator: Calculate tax for income=100000000 VND, rate=20%, deductions=20000000 VND\n"
            "4. auto_weather_fetcher: Get weather for Da Nang in fahrenheit\n"
            "5. validate_data: Validate 'user@example.com' as email\n"
            "6. validate_data: Validate '0912345678' as phone_vn\n"
            "7. validate_data: Validate 'not-a-number' as number\n\n"
            "For each, show the result clearly."
        )

        await client.connect(prompt)

        manual_calls = set()
        auto_calls = set()

        async for message in client.receive_messages():
            if isinstance(message, AssistantMessage):
                for block in message.content:
                    if isinstance(block, TextBlock):
                        print(f"\n[Claude]: {block.text[:200]}")
                    elif isinstance(block, ToolUseBlock):
                        print(f"\n[Tool Call]: {block.name}({json.dumps(block.input, ensure_ascii=False)})")
                        # Phân loại tool call theo server
                        if block.name in ("tax_calculator", "weather_fetcher"):
                            manual_calls.add(block.name)
                        else:
                            auto_calls.add(block.name)

            elif isinstance(message, ResultMessage):
                print(f"\n[Result]: cost=${message.cost_usd:.4f}, turns={message.num_turns}")
                break

        # Báo cáo kết quả
        print("\n" + "=" * 60)
        print("Pattern 1 & 2 (manual) calls:", sorted(manual_calls))
        print("Pattern 3 (auto factory) calls:", sorted(auto_calls))

        all_calls = manual_calls | auto_calls
        expected = {"tax_calculator", "weather_fetcher", "auto_tax_calculator",
                    "auto_weather_fetcher", "validate_data"}
        missing = expected - all_calls
        if missing:
            print(f"\nWARNING: Not called: {missing}")
        else:
            print("\nSUCCESS: All 5 unique tools called!")
        print("=" * 60)

    finally:
        await client.disconnect()


if __name__ == "__main__":
    asyncio.run(main())
```

**Kiểm tra:** Script chạy hoàn tất. Console hiển thị tool calls từ cả 2 servers. Manual tools (Pattern 1, 2) và auto tools (Pattern 3) đều được gọi. Error cases (invalid number) trả về `is_error=True` và Claude xử lý gracefully.

## 6. Edge Cases & Error Handling

| Tình huống | Pattern | Hành vi mong đợi |
|---|---|---|
| Sync function block lâu (>5s) | P1, P3 | `to_thread()` chạy trong thread pool, không block event loop |
| Async function raise exception | P2, P3 | Handler bắt exception, trả về `is_error=True` |
| Sync function raise ValueError | P1, P3 | `to_thread()` propagate exception, handler bắt và trả `is_error=True` |
| Function thiếu type hints | P3 | `get_type_hints()` trả về `{}`, fallback tất cả params sang `"string"` |
| Function có default value | P3 | `inspect.Parameter.empty` check, param có default không nằm trong `required` |
| Return value là dict | P3 | `json.dumps()` serialize thành pretty JSON |
| Return value là number/string | P3 | `str()` convert, trả trong text content |
| Return value là complex object | P3 | `json.dumps(default=str)` fallback sang `str()` |
| Function name trùng | P3 | Override bằng `name` parameter |
| Empty docstring | P3 | Fallback sang `"Tool: {name}"` |
| Multiline docstring | P3 | Chỉ lấy dòng đầu làm description |
| Type hint là custom class (VD: `Decimal`) | P3 | Fallback sang `"string"` qua `PYTHON_TO_JSON_SCHEMA.get()` default |
| Claude truyền thiếu param (không required) | P3 | `kwargs` chỉ lấy params có trong `args`, function dùng default |
| Claude truyền thừa param | P3 | `kwargs` filter chỉ lấy params trong signature, thừa bị bỏ |
| `inspect.iscoroutinefunction` trên lambda | P3 | Trả về `False`, wrap bằng `to_thread()` |

## 7. Acceptance Criteria (Given/When/Then)

### AC1: Pattern 1 -- Sync function wrapped thành công
- **Given:** `calculate_tax` là sync function, wrapped bằng manual `@tool` + `asyncio.to_thread()`
- **When:** Claude gọi `tax_calculator` với `income=50000000, tax_rate=10, deductions=11000000`
- **Then:** Tool trả về text chứa "Tax amount: 3,900,000 VND" (không `is_error`)

### AC2: Pattern 2 -- Async function wrapped thành công
- **Given:** `fetch_weather` là async function, wrapped trực tiếp bằng `@tool`
- **When:** Claude gọi `weather_fetcher` với `city="Hanoi", units="celsius"`
- **Then:** Tool trả về text chứa "Temperature: 28° celsius" (không `is_error`)

### AC3: Factory auto-generates đúng input_schema
- **Given:** `calculate_tax(income: float, tax_rate: float, deductions: float)`
- **When:** `wrap_function(calculate_tax)` được gọi
- **Then:** `input_schema` chứa 3 properties với type `"number"`, và 3 required params

### AC4: Factory detect sync/async đúng
- **Given:** `calculate_tax` (sync) và `fetch_weather` (async)
- **When:** `wrap_function()` wrap cả hai
- **Then:** `inspect.iscoroutinefunction(calculate_tax)` = `False` (dùng `to_thread`), `inspect.iscoroutinefunction(fetch_weather)` = `True` (gọi trực tiếp)

### AC5: Factory wrapped tools chạy end-to-end
- **Given:** 3 auto-wrapped tools đăng ký trong `auto_tools` server
- **When:** Claude gọi `auto_tax_calculator`, `auto_weather_fetcher`, `validate_data`
- **Then:** Tất cả 3 trả về kết quả đúng, không crash

### AC6: Error handling từ factory-wrapped tools
- **Given:** `validate_data("not-a-number", "number")` trả về invalid message
- **When:** Claude gọi `validate_data` với `data="abc", schema_type="number"`
- **Then:** Tool trả về text "Not a valid number: abc" (validation logic từ function gốc)

### AC7: Factory wrapped tool xử lý exception gracefully
- **Given:** `calculate_tax` raise `ValueError` khi income < 0
- **When:** Claude gọi `auto_tax_calculator` với `income=-100`
- **Then:** Tool trả về `is_error=True` với "ValueError: Income cannot be negative"

## 8. Technical Notes

### inspect.iscoroutinefunction() vs inspect.isawaitable()

```python
import inspect

def sync_func(): pass
async def async_func(): pass

inspect.iscoroutinefunction(sync_func)   # False -> cần to_thread()
inspect.iscoroutinefunction(async_func)  # True  -> gọi trực tiếp với await

# KHÔNG dùng isawaitable() vì nó check instance, không check definition
# inspect.isawaitable(async_func) -> False (chưa gọi)
# inspect.isawaitable(async_func()) -> True (đã gọi, có coroutine)
```

`iscoroutinefunction()` check tại thời điểm wrap (1 lần). Đây là approach đúng cho factory pattern.

### asyncio.to_thread() chi tiết

```python
# asyncio.to_thread() chạy sync function trong default ThreadPoolExecutor
# Tương đương: loop.run_in_executor(None, func, *args)
# Python 3.9+ (nhưng SDK yêu cầu 3.10+ nên an toàn)

result = await asyncio.to_thread(calculate_tax, income=100, tax_rate=10, deductions=0)
```

Lưu ý:
- Thread-safe: function gốc phải thread-safe (không shared mutable state)
- GIL: I/O-bound functions được lợi, CPU-bound thì không (do GIL)
- Cancellation: `to_thread()` KHÔNG cancel được function đang chạy trong thread

### Auto-schema từ type hints: hạn chế

`wrap_function()` chỉ xử lý primitive types. Để hỗ trợ complex types cần mở rộng:

```python
# Hiện tại chỉ hỗ trợ:
PYTHON_TO_JSON_SCHEMA = {
    str: "string",
    int: "integer",
    float: "number",
    bool: "boolean",
}

# Để hỗ trợ thêm cần:
# - list[str] -> {"type": "array", "items": {"type": "string"}}
# - Optional[str] -> {"type": "string"} (+ bỏ khỏi required)
# - dict[str, int] -> {"type": "object", "additionalProperties": {"type": "integer"}}
# Đây là out-of-scope cho MAS-2, có thể mở rộng ở MAS-4+
```

### SdkMcpTool vs @tool decorator

`wrap_function()` tạo `SdkMcpTool` trực tiếp thay vì dùng `@tool` decorator. Hai cách tạo ra cùng type:

```python
# @tool decorator approach:
@tool("name", "desc", {"a": str})
async def my_tool(args): ...
# -> SdkMcpTool(name="name", description="desc", input_schema={"a": str}, handler=my_tool)

# Direct SdkMcpTool approach (factory dùng cách này):
SdkMcpTool(name="name", description="desc", input_schema={...}, handler=async_handler)
# -> Cùng type, cùng behavior khi dùng với create_sdk_mcp_server()
```

Factory dùng `SdkMcpTool` trực tiếp vì cần kiểm soát schema generation và handler wrapping, không phù hợp với decorator syntax.

### Khác biệt input_schema format

`@tool` decorator chấp nhận 2 formats:
1. **Simple dict:** `{"a": float, "b": float}` -- SDK auto-convert trong `create_sdk_mcp_server()`
2. **Full JSON Schema:** `{"type": "object", "properties": {...}}` -- dùng trực tiếp

`wrap_function()` luôn tạo full JSON Schema format (option 2) vì cần control chính xác `required` list và `description` cho mỗi property.

## 9. Rủi ro (Risks)

| Rủi ro | Xác suất | Tác động | Giảm thiểu |
|---|---|---|---|
| MAS-1 chưa hoàn thành (dependency) | N/A | Chặn hoàn toàn | Hoàn thành MAS-1 trước; nếu blocked, có thể đọc code MAS-1 nhưng không chạy |
| `get_type_hints()` fail với forward references | Thấp | Schema sai | Try/except fallback sang `{}`, tất cả params default `"string"` |
| Sync function có side effects (file I/O) | Trung bình | Race condition | Document rằng function gốc phải thread-safe |
| `to_thread()` không cancel khi timeout | Thấp | Thread leak | Thêm timeout wrapper nếu cần; chấp nhận risk cho demo |
| Claude chọn manual tool thay vì auto tool | Trung bình | Test không đủ | Prompt rõ ràng chỉ định tool nào dùng |
| 2 servers cùng tên tool | Thấp | Nhầm lẫn | Đặt tên khác nhau: `tax_calculator` vs `auto_tax_calculator` |
| Complex type hints (Optional, Union) gây crash | Trung bình | Schema generation fail | Fallback sang `"string"` cho unknown types; document limitation |
| Factory-wrapped tool trả về None | Thấp | Empty response | `str(None)` -> `"None"`, không crash nhưng output không hữu ích |
| `inspect.signature()` fail trên built-in functions | Thấp | Crash wrap_function | Chỉ support user-defined functions; document limitation |
