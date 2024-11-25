# MCP Python SDK
[![PyPI][pypi-badge]][pypi-url]
[![MIT licensed][mit-badge]][mit-url]
[![Python Version][python-badge]][python-url]
[![Documentation][docs-badge]][docs-url]
[![Specification][spec-badge]][spec-url]
[![GitHub Discussions][discussions-badge]][discussions-url]

[pypi-badge]: https://img.shields.io/pypi/v/mcp.svg
[pypi-url]: https://pypi.org/project/mcp/
[mit-badge]: https://img.shields.io/pypi/l/mcp.svg
[mit-url]: https://github.com/modelcontextprotocol/python-sdk/blob/main/LICENSE
[python-badge]: https://img.shields.io/pypi/pyversions/mcp.svg
[python-url]: https://www.python.org/downloads/
[docs-badge]: https://img.shields.io/badge/docs-modelcontextprotocol.io-blue.svg
[docs-url]: https://modelcontextprotocol.io
[spec-badge]: https://img.shields.io/badge/spec-spec.modelcontextprotocol.io-blue.svg
[spec-url]: https://spec.modelcontextprotocol.io
[discussions-badge]: https://img.shields.io/github/discussions/modelcontextprotocol/python-sdk
[discussions-url]: https://github.com/modelcontextprotocol/python-sdk/discussions

Python implementation of the [Model Context Protocol](https://modelcontextprotocol.io) (MCP), providing both client and server capabilities for integrating with LLM surfaces.

## Overview

The Model Context Protocol allows applications to provide context for LLMs in a standardized way, separating the concerns of providing context from the actual LLM interaction. This Python SDK implements the full MCP specification, making it easy to:

- Build MCP clients that can connect to any MCP server
- Create MCP servers that expose resources, prompts and tools
- Use standard transports like stdio and SSE
- Handle all MCP protocol messages and lifecycle events

## Installation

We recommend the use of [uv](https://docs.astral.sh/uv/) to manage your Python projects:

```bash
uv add mcp
```

Alternatively, add mcp to your `requirements.txt`:
```
pip install mcp
# or add to requirements.txt
pip install -r requirements.txt
```

## Overview
MCP servers provide focused functionality like resources, tools, prompts, and other capabilities that can be reused across many client applications. These servers are designed to be easy to build, highly composable, and modular.

### Key design principles
- Servers are extremely easy to build with clear, simple interfaces
- Multiple servers can be composed seamlessly through a shared protocol
- Each server operates in isolation and cannot access conversation context
- Features can be added progressively through capability negotiation

### Server provided primitives
- [Prompts](https://modelcontextprotocol.io/docs/concepts/prompts): Templatable text
- [Resources](https://modelcontextprotocol.io/docs/concepts/resources): File-like attachments
- [Tools](https://modelcontextprotocol.io/docs/concepts/tools): Functions that models can call
- Utilities:
  - Completion: Auto-completion provider for prompt arguments or resource URI templates
  - Logging: Logging to the client
  - Pagination*: Pagination for long results

### Client provided primitives
 - [Sampling](https://modelcontextprotocol.io/docs/concepts/sampling): Allow servers to sample using client models
 - Roots: Information about locations to operate on (e.g., directories)

Connections between clients and servers are established through transports like **stdio** or **SSE** (Note that most clients support stdio, but not SSE at the moment). The transport layer handles message framing, delivery, and error handling.

## Quick Start

### Creating a Server

MCP servers follow a decorator approach to register handlers for MCP primitives like resources, prompts, and tools. The goal is to provide a simple interface for exposing capabilities to LLM clients.

```python
# /// script
# dependencies = [
#   "mcp"
# ]
# ///
from mcp.server import Server, NotificationOptions
from mcp.server.models import InitializationOptions
import mcp.server.stdio
import mcp.types as types

# Create a server instance
server = Server("example-server")

# Add prompt capabilities
@server.list_prompts()
async def handle_list_prompts() -> list[types.Prompt]:
    return [
        types.Prompt(
            name="example-prompt",
            description="An example prompt template",
            arguments=[
                types.PromptArgument(
                    name="arg1",
                    description="Example argument",
                    required=True
                )
            ]
        )
    ]

@server.get_prompt()
async def handle_get_prompt(
    name: str,
    arguments: dict[str, str] | None
) -> types.GetPromptResult:
    if name != "example-prompt":
        raise ValueError(f"Unknown prompt: {name}")

    return types.GetPromptResult(
        description="Example prompt",
        messages=[
            types.PromptMessage(
                role="user",
                content=types.TextContent(
                    type="text",
                    text="Example prompt text"
                )
            )
        ]
    )

async def run():
    # Run the server as STDIO
    async with mcp.server.stdio.stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            InitializationOptions(
                server_name="example",
                server_version="0.1.0",
                capabilities=server.get_capabilities(
                    notification_options=NotificationOptions(),
                    experimental_capabilities={},
                )
            )
        )

if __name__ == "__main__":
    import asyncio
    asyncio.run(run())
```

### Creating a Client

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

# Create server parameters for stdio connection
server_params = StdioServerParameters(
    command="path/to/server",
    args=[], # Optional command line arguments
    env=None # Optional environment variables
)

async with stdio_client(server_params) as (read, write):
    async with ClientSession(read, write) as session:
        # Initialize the connection
        await session.initialize()

        # List available resources
        resources = await session.list_resources()

        # List available prompts
        prompts = await session.list_prompts()

        # List available tools
        tools = await session.list_tools()

        # Read a resource
        resource = await session.read_resource("file://some/path")

        # Call a tool
        result = await session.call_tool("tool-name", arguments={"arg1": "value"})

        # Get a prompt
        prompt = await session.get_prompt("prompt-name", arguments={"arg1": "value"})
```

## Primitives

The MCP Python SDK provides decorators that map to the core protocol primitives. Each primitive follows a different interaction pattern based on how it is controlled and used:

| Primitive | Control               | Description                                         | Example Use                  |
|-----------|-----------------------|-----------------------------------------------------|------------------------------|
| Prompts   | User-controlled       | Interactive templates invoked by user choice        | Slash commands, menu options |
| Resources | Application-controlled| Contextual data managed by the client application   | File contents, API responses |
| Tools     | Model-controlled      | Functions exposed to the LLM to take actions        | API calls, data updates      |

### User-Controlled Primitives

**Prompts** are designed to be explicitly selected by users for their interactions with LLMs.

| Decorator                | Description                            |
|--------------------------|----------------------------------------|
| `@server.list_prompts()` | List available prompt templates        |
| `@server.get_prompt()`   | Get a specific prompt with arguments   |

### Application-Controlled Primitives

**Resources** are controlled by the client application, which decides how and when they should be used based on its own logic.

| Decorator                      | Description                           |
|--------------------------------|---------------------------------------|
| `@server.list_resources()`     | List available resources              |
| `@server.read_resource()`      | Read a specific resource's content    |
| `@server.subscribe_resource()` | Subscribe to resource updates         |

### Model-Controlled Primitives

**Tools** are exposed to LLMs to enable automated actions, with user approval.

| Decorator              | Description                        |
|------------------------|------------------------------------|
| `@server.list_tools()` | List available tools               |
| `@server.call_tool()`  | Execute a tool with arguments      |

### Server Management

Additional decorators for server functionality:

| Decorator                     | Description                    |
|-------------------------------|--------------------------------|
| `@server.set_logging_level()` | Update server logging level    |

### Capabilities

MCP servers declare capabilities during initialization. These map to specific decorators:

| Capability  | Feature Flag                 | Decorators                                                      | Description                        |
|-------------|------------------------------|-----------------------------------------------------------------|-------------------------------------|
| `prompts`   | `listChanged`                | `@list_prompts`<br/>`@get_prompt`                               | Prompt template management          |
| `resources` | `subscribe`<br/>`listChanged`| `@list_resources`<br/>`@read_resource`<br/>`@subscribe_resource`| Resource exposure and updates       |
| `tools`     | `listChanged`                | `@list_tools`<br/>`@call_tool`                                  | Tool discovery and execution        |
| `logging`   | -                            | `@set_logging_level`                                            | Server logging configuration        |
| `completion`| -                            | `@complete_argument`                                            | Argument completion suggestions     |

Capabilities are negotiated during connection initialization. Servers only need to implement the decorators for capabilities they support.

## Client Interaction

The MCP Python SDK enables servers to interact with clients through request context and session management. This allows servers to perform operations like LLM sampling and progress tracking.

### Request Context

The Request Context provides access to the current request and client session. It can be accessed through `server.request_context` and enables:

- Sampling from the client's LLM
- Sending progress updates
- Logging messages
- Accessing request metadata

Example using request context for LLM sampling:

```python
@server.call_tool()
async def handle_call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    # Access the current request context
    context = server.request_context

    # Use the session to sample from the client's LLM
    result = await context.session.create_message(
        messages=[
            types.SamplingMessage(
                role="user",
                content=types.TextContent(
                    type="text",
                    text="Analyze this data: " + json.dumps(arguments)
                )
            )
        ],
        max_tokens=100
    )

    return [types.TextContent(type="text", text=result.content.text)]
```

Using request context for progress updates:

```python
@server.call_tool()
async def handle_call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    context = server.request_context

    if progress_token := context.meta.progressToken:
        # Send progress notifications
        await context.session.send_progress_notification(
            progress_token=progress_token,
            progress=0.5,
            total=1.0
        )

    # Perform operation...

    if progress_token:
        await context.session.send_progress_notification(
            progress_token=progress_token,
            progress=1.0,
            total=1.0
        )

    return [types.TextContent(type="text", text="Operation complete")]
```

The request context is automatically set for each request and provides a safe way to access the current client session and request metadata.

## Documentation

- [Model Context Protocol documentation](https://modelcontextprotocol.io)
- [Model Context Protocol specification](https://spec.modelcontextprotocol.io)
- [Officially supported servers](https://github.com/modelcontextprotocol/servers)

## Contributing

We are passionate about supporting contributors of all levels of experience and would love to see you get involved in the project. See the [contributing guide](CONTRIBUTING.md) to get started.

## License

This project is licensed under the MIT License - see the LICENSE file for details.
