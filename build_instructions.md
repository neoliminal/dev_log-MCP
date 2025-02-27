# Step-by-Step Instructions: Building the Dev Log MCP Server

This guide provides detailed instructions for building the Dev Log MCP server from scratch. The server allows AI assistants to read from and write to a development log file, maintaining context across sessions.

## Prerequisites

- [Node.js](https://nodejs.org/) (v14 or later)
- [Visual Studio Code](https://code.visualstudio.com/)
- [Roo Cline extension](https://marketplace.visualstudio.com/items?itemName=rooveterinaryinc.roo-cline) for VS Code

## Installation Methods

You can choose between two installation methods:

### Method 1: Build from Scratch

Follow the steps below to build the MCP server from scratch.

### Method 2: Clone the Repository

If the repository is available on GitHub or another git hosting service, you can install it using git:

```bash
# Create a directory for the MCP server (if it doesn't exist)
mkdir -p /path/to/your/workspace

# Navigate to the directory
cd /path/to/your/workspace

# Clone the repository
git clone https://github.com/username/dev_log-MCP.git

# Navigate to the cloned repository
cd dev_log-MCP

# Install dependencies
npm install

# Build the project
npm run build
```

Then proceed to [Step 9: Configure the MCP Server in VS Code](#step-9-configure-the-mcp-server-in-vs-code).

## Step 1: Create Project Directory

```bash
# Create a directory for the project
mkdir dev_log-MCP
cd dev_log-MCP
```

## Step 2: Initialize Node.js Project

```bash
# Initialize a new Node.js project
npm init -y
```

## Step 3: Install Dependencies

```bash
# Install the MCP SDK
npm install @modelcontextprotocol/sdk

# Install TypeScript and Node.js type definitions as dev dependencies
npm install --save-dev typescript @types/node
```

## Step 4: Create TypeScript Configuration

Create a file named `tsconfig.json` with the following content:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "esModuleInterop": true,
    "outDir": "./build",
    "rootDir": "./src",
    "strict": true,
    "declaration": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "**/*.test.ts"]
}
```

## Step 5: Update package.json

Update your `package.json` file to include the following:

```json
{
  "name": "dev-log-mcp",
  "version": "0.1.0",
  "description": "VS Code MCP server for managing dev_log.md",
  "main": "build/index.js",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "start": "node build/index.js"
  }
}
```

## Step 6: Create Source Directory and Implementation File

```bash
# Create the source directory
mkdir -p src
```

Create a file named `src/index.ts` with the following content:

```typescript
#!/usr/bin/env node
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ErrorCode,
  ListToolsRequestSchema,
  McpError,
} from '@modelcontextprotocol/sdk/types.js';
import * as fs from 'fs/promises';
import * as path from 'path';
import { fileURLToPath } from 'url';

// Check if dev_log.md exists, create it if it doesn't
const checkAndCreateDevLog = async (workspacePath: string): Promise<string> => {
  // Determine if we're already in the dev_log-MCP directory
  const currentDir = path.basename(process.cwd());
  const devLogPath = currentDir === 'dev_log-MCP'
    ? path.join(process.cwd(), 'dev_log.md')
    : path.join(process.cwd(), 'dev_log-MCP', 'dev_log.md');
  
  console.error(`Using dev log path: ${devLogPath}`);
  try {
    await fs.access(devLogPath);
    return devLogPath;
  } catch (error) {
    // File doesn't exist, create it
    const initialContent = `# Development Log\n\nThis file contains development notes and progress updates.\n\n[${new Date().toISOString().replace('T', ' ').slice(0, 19)}] Initial dev log file created by MCP setup process.\n`;
    await fs.writeFile(devLogPath, initialContent, 'utf-8');
    return devLogPath;
  }
};

// Get the last N lines from a file
const getLastLines = async (filePath: string, lineCount: number = 20): Promise<string> => {
  const content = await fs.readFile(filePath, 'utf-8');
  const lines = content.split('\n');
  return lines.slice(-lineCount).join('\n');
};

// Append content to the file with timestamp
const appendToFile = async (filePath: string, text: string): Promise<void> => {
  const timestamp = new Date().toISOString().replace('T', ' ').slice(0, 19);
  const formattedEntry = `\n[${timestamp}] ${text}`;
  await fs.appendFile(filePath, formattedEntry, 'utf-8');
};

class DevLogMcpServer {
  private server: Server;
  private workspacePath: string;
  private devLogPath: string | null = null;

  constructor() {
    this.server = new Server(
      {
        name: 'dev-log',
        version: '0.1.0',
      },
      {
        capabilities: {
          tools: {},
        },
      }
    );

    // Get the workspace path (this will be set properly during initialization)
    this.workspacePath = process.cwd();

    this.setupToolHandlers();
    
    // Error handling
    this.server.onerror = (error: unknown) => {
      const errorMessage = error instanceof Error ? error.message : String(error);
      console.error('[MCP Error]', errorMessage);
      if (error instanceof Error && error.stack) {
        console.error('[MCP Error Stack]', error.stack);
      }
    };
    process.on('SIGINT', async () => {
      await this.server.close();
      process.exit(0);
    });
  }

  private async initialize(): Promise<void> {
    try {
      console.error(`Current working directory: ${this.workspacePath}`);
      console.error(`Parent directory: ${path.resolve(this.workspacePath, '..')}`);
      this.devLogPath = await checkAndCreateDevLog(this.workspacePath);
      console.error(`Dev log path: ${this.devLogPath}`);
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : String(error);
      console.error('Failed to initialize dev log:', errorMessage);
      throw error;
    }
  }

  private setupToolHandlers() {
    this.server.setRequestHandler(ListToolsRequestSchema, async () => ({
      tools: [
        {
          name: 'tail_dev_log',
          description: 'Get the last N lines from the dev_log.md file',
          inputSchema: {
            type: 'object',
            properties: {
              lines: {
                type: 'number',
                description: 'Number of lines to return (default: 20)',
                minimum: 1,
                maximum: 1000,
              },
            },
          },
        },
        {
          name: 'write_to_dev_log',
          description: 'Append text to the dev_log.md file with timestamp',
          inputSchema: {
            type: 'object',
            properties: {
              text: {
                type: 'string',
                description: 'Text content to append to the log',
              },
            },
            required: ['text'],
          },
        },
      ],
    }));

    this.server.setRequestHandler(CallToolRequestSchema, async (request: any) => {
      if (!this.devLogPath) {
        await this.initialize();
      }

      if (request.params.name === 'tail_dev_log') {
        const lines = request.params.arguments?.lines || 20;
        
        if (typeof lines !== 'number' || lines < 1) {
          throw new McpError(
            ErrorCode.InvalidParams,
            'Lines parameter must be a positive number'
          );
        }

        try {
          const lastLines = await getLastLines(this.devLogPath!, lines);
          return {
            content: [
              {
                type: 'text',
                text: lastLines,
              },
            ],
          };
        } catch (error) {
          const errorMessage = error instanceof Error ? error.message : String(error);
          return {
            content: [
              {
                type: 'text',
                text: `Error reading dev_log.md: ${errorMessage}`,
              },
            ],
            isError: true,
          };
        }
      }
      
      if (request.params.name === 'write_to_dev_log') {
        const text = request.params.arguments?.text;
        
        if (!text || typeof text !== 'string') {
          throw new McpError(
            ErrorCode.InvalidParams,
            'Text parameter is required and must be a string'
          );
        }

        try {
          await appendToFile(this.devLogPath!, text);
          return {
            content: [
              {
                type: 'text',
                text: `Entry added to dev_log.md`,
              },
            ],
          };
        } catch (error) {
          const errorMessage = error instanceof Error ? error.message : String(error);
          return {
            content: [
              {
                type: 'text',
                text: `Error writing to dev_log.md: ${errorMessage}`,
              },
            ],
            isError: true,
          };
        }
      }

      throw new McpError(
        ErrorCode.MethodNotFound,
        `Unknown tool: ${request.params.name}`
      );
    });
  }

  async run() {
    try {
      console.error('Starting MCP server initialization...');
      await this.initialize();
      console.error('Initialization complete, creating transport...');
      const transport = new StdioServerTransport();
      console.error('Transport created, connecting to server...');
      await this.server.connect(transport);
      console.error('Dev Log MCP server running on stdio');
      
      // Log server status periodically
      setInterval(() => {
        console.error(`[${new Date().toISOString()}] MCP server still running`);
      }, 10000);
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : String(error);
      console.error('Failed to start server:', errorMessage);
      if (error instanceof Error && error.stack) {
        console.error('Stack trace:', error.stack);
      }
      process.exit(1);
    }
  }
}

const server = new DevLogMcpServer();
server.run().catch(console.error);
```

## Step 7: Build the Project

```bash
# Build the TypeScript project
npm run build
```

## Step 8: Create a Test Script (Optional)

Create a file named `test-dev-log.js` with the following content:

```javascript
// Simple script to test reading and writing to the dev_log.md file
const fs = require('fs');
const path = require('path');

// Path to the dev_log.md file
const devLogPath = path.join(__dirname, 'dev_log.md');
console.log('Dev log path:', devLogPath);
console.log('File exists:', fs.existsSync(devLogPath));

// Function to read the last N lines from the file
function readLastLines(filePath, lineCount = 20) {
  try {
    const content = fs.readFileSync(filePath, 'utf-8');
    const lines = content.split('\n');
    const lastLines = lines.slice(-lineCount).join('\n');
    console.log('Last lines from dev_log.md:');
    console.log(lastLines);
    return lastLines;
  } catch (error) {
    console.error('Error reading dev_log.md:', error.message);
    return null;
  }
}

// Function to append content to the file with timestamp
function appendToFile(filePath, text) {
  try {
    const timestamp = new Date().toISOString().replace('T', ' ').slice(0, 19);
    const formattedEntry = `\n[${timestamp}] ${text}`;
    fs.appendFileSync(filePath, formattedEntry, 'utf-8');
    console.log('Entry added to dev_log.md:', formattedEntry);
    return true;
  } catch (error) {
    console.error('Error writing to dev_log.md:', error.message);
    return false;
  }
}

// Test reading the last 20 lines
readLastLines(devLogPath);

// Test appending a new entry
appendToFile(devLogPath, 'Test entry from test-dev-log.js script');

// Read the last 20 lines again to verify the new entry
readLastLines(devLogPath);
```

## Step 9: Configure the MCP Server in VS Code

1. Open VS Code
2. Open the MCP settings file:
   - Windows: `%APPDATA%\Code\User\globalStorage\rooveterinaryinc.roo-cline\settings\cline_mcp_settings.json`
   - macOS: `~/Library/Application Support/Code/User/globalStorage/rooveterinaryinc.roo-cline/settings/cline_mcp_settings.json`
   - Linux: `~/.config/Code/User/globalStorage/rooveterinaryinc.roo-cline/settings/cline_mcp_settings.json`

3. Add the following configuration to the `mcpServers` object:

```json
{
  "mcpServers": {
    "dev-log": {
      "command": "C:/Program Files/nodejs/node.exe",
      "args": ["C:/path/to/your/dev_log-MCP/build/index.js"],
      "disabled": false,
      "alwaysAllow": [
        "tail_dev_log",
        "write_to_dev_log"
      ]
    }
  }
}
```

> **Important Notes:** 
> - Replace `C:/path/to/your/dev_log-MCP` with the absolute path to your project directory. 
> - Make sure to use forward slashes (/) in the path, even on Windows.
> - Using the full path to node.exe (`C:/Program Files/nodejs/node.exe`) is more reliable than just using `node`.
> - The `alwaysAllow` setting determines which tools can be used without prompting the user for permission each time.

## Step 10: Restart VS Code

After configuring the MCP server, restart VS Code to apply the changes.

## Step 11: Test the MCP Server

1. Open a new VS Code window
2. Create a new file and save it
3. Open the Roo Cline extension
4. Test reading from the dev log:

```
<use_mcp_tool>
<server_name>dev-log</server_name>
<tool_name>tail_dev_log</tool_name>
<arguments>
{
  "lines": 20
}
</arguments>
</use_mcp_tool>
```

5. Test writing to the dev log:

```
<use_mcp_tool>
<server_name>dev-log</server_name>
<tool_name>write_to_dev_log</tool_name>
<arguments>
{
  "text": "Test entry from Roo Cline"
}
</arguments>
</use_mcp_tool>
```

## Security Considerations

### alwaysAllow Setting

The `alwaysAllow` setting in the MCP configuration determines which tools can be used without prompting the user for permission each time. By default, this is set to an empty array, which means the user will be prompted for permission every time a tool is used.

For convenience, you may want to add the tools to the `alwaysAllow` array:

```json
"alwaysAllow": [
  "tail_dev_log",
  "write_to_dev_log"
]
```

However, be aware that this means the tools can be used without user permission, which may be a security concern in some environments.

## Troubleshooting

### MCP Server Not Connecting

If the MCP server is not connecting to the Roo Cline extension, check the following:

1. Ensure the path in the MCP settings file is correct and uses forward slashes
2. Check that the server has been built successfully (`npm run build`)
3. Restart VS Code
4. Check the VS Code Developer Tools console for any error messages (Help > Toggle Developer Tools)

### File Access Issues

If the server is having trouble accessing or writing to the dev_log.md file:

1. Check file permissions
2. Ensure the path is correct in the `checkAndCreateDevLog` function
3. Try running the test script (`node test-dev-log.js`) to verify file access

## Customization

### Changing the Dev Log File Path

By default, the MCP server uses the current working directory for the dev_log.md file. To use a different path:

1. Open the `src/index.ts` file
2. Locate the `checkAndCreateDevLog` function
3. Modify the path in this line:
   ```typescript
   const devLogPath = currentDir === 'dev_log-MCP'
     ? path.join(process.cwd(), 'dev_log.md')
     : path.join(process.cwd(), 'dev_log-MCP', 'dev_log.md');
   ```
4. Replace it with your desired path
5. Rebuild the project: `npm run build`
6. Restart VS Code to apply the changes

### Adding More Tools

You can extend the MCP server with additional tools by modifying the `setupToolHandlers` method in the `DevLogMcpServer` class. Follow these steps:

1. Add a new tool definition in the `ListToolsRequestSchema` handler
2. Implement the tool's functionality in the `CallToolRequestSchema` handler
3. Rebuild the project and restart VS Code

Example of adding a new tool to search the dev log:

```typescript
// In the ListToolsRequestSchema handler, add:
{
  name: 'search_dev_log',
  description: 'Search for text in the dev_log.md file',
  inputSchema: {
    type: 'object',
    properties: {
      query: {
        type: 'string',
        description: 'Text to search for',
      },
    },
    required: ['query'],
  },
},

// In the CallToolRequestSchema handler, add:
if (request.params.name === 'search_dev_log') {
  const query = request.params.arguments?.query;
  
  if (!query || typeof query !== 'string') {
    throw new McpError(
      ErrorCode.InvalidParams,
      'Query parameter is required and must be a string'
    );
  }

  try {
    const content = await fs.readFile(this.devLogPath!, 'utf-8');
    const lines = content.split('\n');
    const matchingLines = lines.filter(line => 
      line.toLowerCase().includes(query.toLowerCase())
    );
    
    return {
      content: [
        {
          type: 'text',
          text: matchingLines.join('\n') || 'No matches found',
        },
      ],
    };
  } catch (error) {
    const errorMessage = error instanceof Error ? error.message : String(error);
    return {
      content: [
        {
          type: 'text',
          text: `Error searching dev_log.md: ${errorMessage}`,
        },
      ],
      isError: true,
    };
  }
}