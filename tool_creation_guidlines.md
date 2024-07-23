# Tool Creation Guidelines

When designing custom functions to be used as tools in this orchestrator program, you should follow these guidelines:

1. **Input Parameters:**
   - Design your functions to accept input parameters that match the "input_schema" defined in the tool description.
   - Use type hints for clarity and to match the schema types (e.g., `str`, `int`, `float`, `list`, `dict`).

2. **Return Values:**
   - Return data that can be easily serialized to JSON (strings, numbers, lists, dictionaries).
   - Avoid returning complex objects or data types that can't be easily represented in JSON.

3. **Error Handling:**
   - Implement proper error handling within the function.
   - Raise specific exceptions for different error conditions.

4. **Documentation:**
   - Include clear docstrings explaining the function's purpose, parameters, and return value.

Here's a template for designing these functions:

```python
from typing import List, Dict, Any

def my_custom_tool(param1: str, param2: int, param3: List[str]) -> Dict[str, Any]:
    """
    Description of what this tool does.

    Args:
        param1 (str): Description of param1
        param2 (int): Description of param2
        param3 (List[str]): Description of param3

    Returns:
        Dict[str, Any]: Description of the return value

    Raises:
        ValueError: Description of when this error is raised
    """
    try:
        # Your function logic here
        result = {
            "output1": "some result",
            "output2": 42,
            "output3": ["item1", "item2"]
        }
        return result
    except SomeSpecificError as e:
        raise ValueError(f"An error occurred: {str(e)}")

# Example tool definition
my_custom_tool_definition = {
    "name": "my_custom_tool",
    "description": "Description of what this tool does",
    "input_schema": {
        "type": "object",
        "properties": {
            "param1": {"type": "string", "description": "Description of param1"},
            "param2": {"type": "integer", "description": "Description of param2"},
            "param3": {
                "type": "array",
                "items": {"type": "string"},
                "description": "Description of param3"
            }
        },
        "required": ["param1", "param2", "param3"]
    }
}
```

When integrating these functions into your orchestrator:

1. Add the tool definition to your `tools` list.
2. Update the `process_tool_call` function to handle the new tool:

```python
def process_tool_call(tool_name: str, tool_input: dict) -> Any:
    if tool_name == "my_custom_tool":
        return my_custom_tool(**tool_input)
    # ... other tool handlers ...
    else:
        raise ValueError(f"Unknown tool: {tool_name}")
```

By following this structure, you can create small, specific programs (functions) that:
1. Accept well-defined inputs
2. Perform specific actions
3. Return specific outputs

This approach makes your tools modular, reusable, and easy to integrate into the larger orchestrator program. It also allows TAO Agent to understand how to use these tools effectively based on their descriptions and input schemas.
