# The Complete Guide to Creating MCP Servers: Supercharge Your AI Workflows

The Model Context Protocol (MCP) is revolutionizing how AI assistants interact with external tools and data sources. Whether you're using Claude Desktop, Cline, Cursor, or GitHub Copilot, MCP servers can extend your AI's capabilities dramatically. In this comprehensive guide, we'll walk through everything you need to know to create your own MCP servers.

## What is MCP and Why Should You Care?

The Model Context Protocol (MCP) is an open standard that enables AI models to securely connect to external tools, databases, APIs, and services. Think of it as a universal translator that lets your AI assistant talk to virtually any system.

### Real-World Benefits

- **Database Integration**: Query your company's database directly from your AI chat
- **API Automation**: Have your AI make REST API calls and process responses
- **File System Access**: Let AI read, write, and manage files on your system
- **Cloud Service Control**: Manage AWS resources, GitHub repos, or Slack channels
- **Custom Business Logic**: Integrate proprietary tools and workflows

## Prerequisites

Before we dive in, make sure you have:

- **Node.js 18+** or **Python 3.9+** installed
- **Git** for version control
- An **MCP-compatible client** (Claude Desktop, Cline, Cursor, etc.)
- Basic programming knowledge in JavaScript/TypeScript or Python

## MCP Architecture Overview

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   AI Assistant  │    │   MCP Server    │    │  External API   │
│  (Claude, etc.) │◄──►│   (Your Code)   │◄──►│   or Service    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

**MCP Servers** expose three main types of capabilities:
1. **Tools**: Functions the AI can call
2. **Resources**: Data the AI can read
3. **Prompts**: Templates the AI can use

## Setting Up Your Development Environment

### For JavaScript/TypeScript Projects

```bash
# Create a new MCP server using the official template
npx @modelcontextprotocol/create-server my-awesome-server
cd my-awesome-server
npm install

# Install additional dependencies if needed
npm install axios dotenv
```

### For Python Projects

```bash
# Install MCP SDK
pip install mcp
# Or with uv (recommended)
uv add "mcp[cli]"

# Create project structure
mkdir my-mcp-server
cd my-mcp-server
```

## Creating Your First MCP Server

Let's build a practical example: a **Weather MCP Server** that fetches weather data.

### JavaScript/TypeScript Implementation

```typescript
#!/usr/bin/env node

import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';
import axios from 'axios';

// Server configuration
const server = new Server(
  {
    name: 'weather-server',
    version: '1.0.0',
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

// API configuration
const WEATHER_API_KEY = process.env.WEATHER_API_KEY;
const WEATHER_BASE_URL = 'https://api.openweathermap.org/data/2.5';

// Tool definitions
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: 'get_weather',
        description: 'Get current weather for a city',
        inputSchema: {
          type: 'object',
          properties: {
            city: {
              type: 'string',
              description: 'City name',
            },
            units: {
              type: 'string',
              enum: ['metric', 'imperial', 'standard'],
              description: 'Temperature units',
              default: 'metric',
            },
          },
          required: ['city'],
        },
      },
      {
        name: 'get_forecast',
        description: 'Get 5-day weather forecast for a city',
        inputSchema: {
          type: 'object',
          properties: {
            city: {
              type: 'string',
              description: 'City name',
            },
            days: {
              type: 'number',
              description: 'Number of days (1-5)',
              minimum: 1,
              maximum: 5,
              default: 3,
            },
          },
          required: ['city'],
        },
      },
    ],
  };
});

// Tool implementations
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  try {
    switch (name) {
      case 'get_weather': {
        const { city, units = 'metric' } = args as { city: string; units?: string };
        
        console.error(`[Weather] Fetching current weather for ${city}`);
        
        const response = await axios.get(`${WEATHER_BASE_URL}/weather`, {
          params: {
            q: city,
            appid: WEATHER_API_KEY,
            units,
          },
        });

        const data = response.data;
        
        return {
          content: [
            {
              type: 'text',
              text: JSON.stringify({
                city: data.name,
                country: data.sys.country,
                temperature: data.main.temp,
                feels_like: data.main.feels_like,
                humidity: data.main.humidity,
                description: data.weather[0].description,
                wind_speed: data.wind.speed,
                units: units,
              }, null, 2),
            },
          ],
        };
      }

      case 'get_forecast': {
        const { city, days = 3 } = args as { city: string; days?: number };
        
        console.error(`[Weather] Fetching ${days}-day forecast for ${city}`);
        
        const response = await axios.get(`${WEATHER_BASE_URL}/forecast`, {
          params: {
            q: city,
            appid: WEATHER_API_KEY,
            units: 'metric',
            cnt: days * 8, // 8 forecasts per day (3-hour intervals)
          },
        });

        const data = response.data;
        const dailyForecasts = data.list.filter((_: any, index: number) => index % 8 === 0);
        
        return {
          content: [
            {
              type: 'text',
              text: JSON.stringify({
                city: data.city.name,
                country: data.city.country,
                forecasts: dailyForecasts.slice(0, days).map((forecast: any) => ({
                  date: forecast.dt_txt,
                  temperature: forecast.main.temp,
                  description: forecast.weather[0].description,
                  humidity: forecast.main.humidity,
                })),
              }, null, 2),
            },
          ],
        };
      }

      default:
        throw new Error(`Unknown tool: ${name}`);
    }
  } catch (error) {
    console.error(`[Error] Tool ${name} failed:`, error);
    
    return {
      content: [
        {
          type: 'text',
          text: `Error: ${error instanceof Error ? error.message : String(error)}`,
        },
      ],
      isError: true,
    };
  }
});

// Start the server
async function main() {
  console.error('[Setup] Starting Weather MCP Server...');
  
  const transport = new StdioServerTransport();
  await server.connect(transport);
  
  console.error('[Setup] Weather MCP Server running on stdio');
}

if (require.main === module) {
  main().catch((error) => {
    console.error('[Fatal] Server failed to start:', error);
    process.exit(1);
  });
}
```

### Python Implementation

```python
#!/usr/bin/env python3

import asyncio
import json
import os
import sys
from typing import Any, Dict, List

import httpx
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import (
    CallToolRequest,
    CallToolResult,
    ListToolsRequest,
    ListToolsResult,
    TextContent,
    Tool,
)

# Server configuration
app = Server("weather-server")

# API configuration
WEATHER_API_KEY = os.getenv("WEATHER_API_KEY")
WEATHER_BASE_URL = "https://api.openweathermap.org/data/2.5"

@app.list_tools()
async def list_tools() -> List[Tool]:
    """List available tools."""
    return [
        Tool(
            name="get_weather",
            description="Get current weather for a city",
            inputSchema={
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "City name"
                    },
                    "units": {
                        "type": "string",
                        "enum": ["metric", "imperial", "standard"],
                        "description": "Temperature units",
                        "default": "metric"
                    }
                },
                "required": ["city"]
            }
        ),
        Tool(
            name="get_forecast",
            description="Get 5-day weather forecast for a city",
            inputSchema={
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "City name"
                    },
                    "days": {
                        "type": "number",
                        "description": "Number of days (1-5)",
                        "minimum": 1,
                        "maximum": 5,
                        "default": 3
                    }
                },
                "required": ["city"]
            }
        )
    ]

@app.call_tool()
async def call_tool(name: str, arguments: Dict[str, Any]) -> CallToolResult:
    """Handle tool calls."""
    
    try:
        if name == "get_weather":
            city = arguments.get("city")
            units = arguments.get("units", "metric")
            
            print(f"[Weather] Fetching current weather for {city}", file=sys.stderr)
            
            async with httpx.AsyncClient() as client:
                response = await client.get(
                    f"{WEATHER_BASE_URL}/weather",
                    params={
                        "q": city,
                        "appid": WEATHER_API_KEY,
                        "units": units
                    }
                )
                response.raise_for_status()
                data = response.json()
            
            result = {
                "city": data["name"],
                "country": data["sys"]["country"],
                "temperature": data["main"]["temp"],
                "feels_like": data["main"]["feels_like"],
                "humidity": data["main"]["humidity"],
                "description": data["weather"][0]["description"],
                "wind_speed": data["wind"]["speed"],
                "units": units
            }
            
            return CallToolResult(
                content=[TextContent(type="text", text=json.dumps(result, indent=2))]
            )
            
        elif name == "get_forecast":
            city = arguments.get("city")
            days = arguments.get("days", 3)
            
            print(f"[Weather] Fetching {days}-day forecast for {city}", file=sys.stderr)
            
            async with httpx.AsyncClient() as client:
                response = await client.get(
                    f"{WEATHER_BASE_URL}/forecast",
                    params={
                        "q": city,
                        "appid": WEATHER_API_KEY,
                        "units": "metric",
                        "cnt": days * 8  # 8 forecasts per day
                    }
                )
                response.raise_for_status()
                data = response.json()
            
            # Get daily forecasts (every 8th entry for daily data)
            daily_forecasts = data["list"][::8][:days]
            
            result = {
                "city": data["city"]["name"],
                "country": data["city"]["country"],
                "forecasts": [
                    {
                        "date": forecast["dt_txt"],
                        "temperature": forecast["main"]["temp"],
                        "description": forecast["weather"][0]["description"],
                        "humidity": forecast["main"]["humidity"]
                    }
                    for forecast in daily_forecasts
                ]
            }
            
            return CallToolResult(
                content=[TextContent(type="text", text=json.dumps(result, indent=2))]
            )
            
        else:
            raise ValueError(f"Unknown tool: {name}")
            
    except Exception as error:
        print(f"[Error] Tool {name} failed: {error}", file=sys.stderr)
        
        return CallToolResult(
            content=[TextContent(type="text", text=f"Error: {str(error)}")],
            isError=True
        )

async def main():
    """Run the server."""
    print("[Setup] Starting Weather MCP Server...", file=sys.stderr)
    
    async with stdio_server() as (read_stream, write_stream):
        await app.run(read_stream, write_stream, app.create_initialization_options())

if __name__ == "__main__":
    asyncio.run(main())
```

## Building and Testing

### Build Your Server

For TypeScript projects:
```bash
# Compile TypeScript to JavaScript
npm run build

# Test the server locally
node build/index.js
```

For Python projects:
```bash
# Make the script executable
chmod +x server.py

# Test the server
python server.py
```

### Environment Configuration

Create a `.env` file for your API keys:
```bash
WEATHER_API_KEY=your_openweathermap_api_key_here
```

## MCP Client Configuration

Now you need to configure your AI client to use your MCP server.

### Cline Configuration

Add to your `cline_mcp_settings.json`:

```json
{
  "mcpServers": {
    "weather-server": {
      "command": "node",
      "args": ["path/to/your/build/index.js"],
      "env": {
        "WEATHER_API_KEY": "your_api_key_here"
      },
      "disabled": false
    }
  }
}
```

### Cursor Configuration

Add to your VS Code `settings.json` or `.vscode/mcp.json`:

```json
{
  "mcp": {
    "servers": {
      "weather-server": {
        "type": "stdio",
        "command": "node",
        "args": ["./build/index.js"],
        "env": {
          "WEATHER_API_KEY": "your_api_key_here"
        }
      }
    }
  }
}
```

### Claude Desktop Configuration

Add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "weather-server": {
      "command": "node",
      "args": ["path/to/build/index.js"],
      "env": {
        "WEATHER_API_KEY": "your_api_key_here"
      }
    }
  }
}
```

## Testing Your MCP Server

### Manual Testing

Test your server with sample inputs:
```bash
# For Node.js
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}' | node build/index.js

# For Python
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}' | python server.py
```

### Integration Testing

1. **Start your AI client** (Cline, Cursor, Claude Desktop)
2. **Verify server connection** in the MCP servers list
3. **Test each tool** with realistic inputs:
   - "What's the weather in New York?"
   - "Get me a 5-day forecast for London"
   - "Check the current temperature in Tokyo"

## Advanced MCP Features

### Adding Resources

Resources let your AI read data:

```typescript
// TypeScript example
server.setRequestHandler(ListResourcesRequestSchema, async () => {
  return {
    resources: [
      {
        uri: 'weather://config',
        name: 'Weather Configuration',
        mimeType: 'application/json',
      },
    ],
  };
});

server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const { uri } = request.params;
  
  if (uri === 'weather://config') {
    return {
      contents: [
        {
          uri,
          mimeType: 'application/json',
          text: JSON.stringify({
            supportedUnits: ['metric', 'imperial', 'standard'],
            apiVersion: '2.5',
            rateLimit: '1000 calls/day',
          }),
        },
      ],
    };
  }
  
  throw new Error(`Resource not found: ${uri}`);
});
```

### Adding Prompts

Prompts provide templates:

```typescript
server.setRequestHandler(ListPromptsRequestSchema, async () => {
  return {
    prompts: [
      {
        name: 'weather_report',
        description: 'Generate a detailed weather report',
        arguments: [
          {
            name: 'city',
            description: 'City to get weather for',
            required: true,
          },
        ],
      },
    ],
  };
});
```

### Error Handling Best Practices

```typescript
try {
  // Your tool logic here
  const result = await apiCall();
  
  return {
    content: [
      {
        type: 'text',
        text: JSON.stringify(result, null, 2),
      },
    ],
  };
} catch (error) {
  console.error(`[Error] ${toolName} failed:`, error);
  
  // Return user-friendly error
  if (error.response?.status === 401) {
    return {
      content: [
        {
          type: 'text',
          text: 'Error: Invalid API key. Please check your configuration.',
        },
      ],
      isError: true,
    };
  }
  
  return {
    content: [
      {
        type: 'text',
        text: `Error: ${error.message || 'Unknown error occurred'}`,
      },
    ],
    isError: true,
  };
}
```

## Security Best Practices

### Environment Variables
- **Never hardcode** API keys or secrets
- **Use environment variables** for all sensitive data
- **Validate inputs** before making external calls

### Rate Limiting
```typescript
import { RateLimiter } from 'limiter';

const limiter = new RateLimiter({
  tokensPerInterval: 100,
  interval: 'hour'
});

// In your tool handler
const remainingRequests = await limiter.removeTokens(1);
if (remainingRequests < 0) {
  throw new Error('Rate limit exceeded. Please try again later.');
}
```

### Input Validation
```typescript
function validateCity(city: string): void {
  if (!city || typeof city !== 'string') {
    throw new Error('City name is required and must be a string');
  }
  
  if (city.length > 100) {
    throw new Error('City name is too long');
  }
  
  // Basic sanitization
  if (!/^[a-zA-Z\s-']+$/.test(city)) {
    throw new Error('City name contains invalid characters');
  }
}
```

## Publishing and Distribution

### Package Your Server

Create a `package.json` with proper metadata:

```json
{
  "name": "weather-mcp-server",
  "version": "1.0.0",
  "description": "MCP server for weather data",
  "main": "build/index.js",
  "bin": {
    "weather-mcp-server": "build/index.js"
  },
  "scripts": {
    "build": "tsc",
    "start": "node build/index.js"
  },
  "keywords": ["mcp", "weather", "ai", "assistant"],
  "author": "Your Name",
  "license": "MIT"
}
```

### NPM Distribution

```bash
# Build and publish
npm run build
npm publish

# Users can then install with:
npm install -g weather-mcp-server
```

### GitHub Distribution

```bash
# Tag and release
git tag v1.0.0
git push origin v1.0.0

# Create GitHub release with built artifacts
```

## Real-World MCP Server Ideas

### Database Integrations
- **PostgreSQL MCP**: Query databases with natural language
- **MongoDB MCP**: Document database operations
- **Redis MCP**: Cache management and real-time data

### Cloud Services
- **AWS MCP**: Manage EC2, S3, Lambda functions
- **Azure MCP**: Control Azure resources
- **GCP MCP**: Google Cloud operations

### Development Tools
- **Git MCP**: Repository management and code operations
- **Docker MCP**: Container management
- **Kubernetes MCP**: Cluster operations

### Business Applications
- **Salesforce MCP**: CRM data access and manipulation
- **Jira MCP**: Issue tracking and project management
- **Slack MCP**: Team communication automation

### Data Sources
- **RSS MCP**: News and content aggregation
- **Calendar MCP**: Schedule management
- **Email MCP**: Email operations and analytics

## Troubleshooting Common Issues

### Server Not Starting
```bash
# Check for syntax errors
node --check build/index.js

# Run with verbose logging
DEBUG=* node build/index.js
```

### Client Connection Issues
- **Verify file paths** in configuration
- **Check environment variables** are set correctly
- **Ensure server has execute permissions**

### Tool Call Failures
- **Add comprehensive logging** to identify issues
- **Validate all inputs** before processing
- **Handle API errors gracefully**

### Performance Problems
- **Implement caching** for frequently accessed data
- **Add request timeouts** to prevent hanging
- **Monitor memory usage** for long-running servers

## Next Steps and Advanced Topics

### Building Complex Workflows
- **Chain multiple tools** together
- **Implement state management** across calls
- **Create conditional logic** based on context

### Integration Patterns
- **Database connections** with pooling
- **Message queue integration** for async operations
- **Webhook handling** for real-time updates

### Monitoring and Observability
- **Add structured logging** with correlation IDs
- **Implement health checks** and status endpoints
- **Track usage metrics** and performance

## Conclusion

MCP servers open up incredible possibilities for extending AI assistants with real-world capabilities. Start with simple tools like our weather example, then gradually build more complex integrations as you become comfortable with the protocol.

The key to successful MCP servers is:
- **Clear, focused functionality**
- **Robust error handling**
- **Comprehensive testing**
- **Good documentation**



---

**Resources:**
- [MCP Specification](https://modelcontextprotocol.io/)
- [Official MCP SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [Community Examples](https://github.com/topics/model-context-protocol)
- [Anthropic's MCP Guide](https://www.anthropic.com/news/model-context-protocol)

*Have you built an interesting MCP server? Share your experience in the comments below!*