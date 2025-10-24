# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Essential Commands

### Building and Running

```bash
# Install dependencies
go mod download

# Build the binary
go build -o k8s-mcp-server main.go

# Run in different modes
./k8s-mcp-server --mode stdio                    # Standard I/O mode (for CLI integrations)
./k8s-mcp-server --mode sse                      # Server-Sent Events mode (default, port 8080)
./k8s-mcp-server --mode streamable-http          # Streamable HTTP mode (MCP spec compliant)
./k8s-mcp-server --mode sse --port 9090          # Custom port

# Enable read-only mode (disables write operations)
./k8s-mcp-server --read-only

# Disable specific tool categories
./k8s-mcp-server --no-k8s                        # Only Helm tools
./k8s-mcp-server --no-helm                       # Only Kubernetes tools
./k8s-mcp-server --read-only --no-helm           # Read-only K8s tools only
```

### Docker

```bash
# Build Docker image
docker build -t k8s-mcp-server:latest .

# Run with kubeconfig mounted (SSE mode, default)
docker run -p 8080:8080 -v ~/.kube/config:/home/appuser/.kube/config:ro k8s-mcp-server:latest

# Run in stdio mode
docker run -i --rm -v ~/.kube/config:/home/appuser/.kube/config:ro k8s-mcp-server:latest --mode stdio

# Run in read-only mode
docker run -p 8080:8080 -v ~/.kube/config:/home/appuser/.kube/config:ro k8s-mcp-server:latest --read-only

# Use Docker Compose
docker compose up -d
docker compose logs -f k8s-mcp-server
```

### Environment Variables

- `SERVER_MODE`: Transport mode (stdio, sse, streamable-http)
- `SERVER_PORT`: Port for SSE/streamable-http modes (default: 8080)
- `KUBECONFIG`: Path to kubeconfig file
- `KUBERNETES_CONTEXT`: Specific cluster context to use

## Architecture Overview

This project implements a Model Context Protocol (MCP) server for Kubernetes and Helm operations using a layered architecture:

### Layer Structure

```
MCP Server (mark3labs/mcp-go)
    ↓
Tools Layer (tools/)
    - Defines MCP tool schemas (input/output specifications)
    - k8s.go: Kubernetes tool definitions
    - helm.go: Helm tool definitions
    ↓
Handlers Layer (handlers/)
    - Implements business logic for each tool
    - Maps MCP requests to client operations
    - k8s.go: Kubernetes operation handlers
    - helm.go: Helm operation handlers
    ↓
Client Layer (pkg/)
    - Abstracts Kubernetes/Helm API interactions
    - pkg/k8s/: Dynamic Kubernetes client with GVR caching
    - pkg/helm/: Helm v3 action client wrapper
```

### Key Components

**MCP Server (`main.go`)**
- Initializes and registers tools based on flags
- Supports three transport modes: stdio (CLI), SSE (web), streamable-http (MCP spec)
- Conditionally registers tools based on `--read-only`, `--no-k8s`, `--no-helm` flags
- Uses `mark3labs/mcp-go` library for MCP protocol implementation

**Kubernetes Client (`pkg/k8s/client.go`)**
- Uses `dynamic.Interface` for arbitrary resource types
- Implements GVR (GroupVersionResource) caching for performance
- Provides CRUD operations for any Kubernetes resource
- Supports metrics collection via `metrics-server`

**Helm Client (`pkg/helm/client.go`)**
- Wraps Helm v3 action API
- Supports OCI and traditional chart repositories
- Manages release lifecycle (install, upgrade, rollback, uninstall)

## Key Architectural Patterns

### Tool Registration Pattern

All tools follow a 3-step registration process defined in `main.go`:

1. **Tool Definition** (`tools/`)
   ```go
   func GetAPIResourcesTool() mcp.Tool {
       return mcp.NewTool("getAPIResources",
           mcp.WithDescription("..."),
           mcp.WithBoolean("includeNamespaceScoped", ...),
           mcp.WithBoolean("includeClusterScoped", ...))
   }
   ```

2. **Handler Implementation** (`handlers/`)
   ```go
   func GetAPIResources(client *k8s.Client) func(context.Context, mcp.CallToolRequest) (*mcp.CallToolResult, error) {
       return func(ctx context.Context, request mcp.CallToolRequest) (*mcp.CallToolResult, error) {
           // Extract args, call client, serialize response
       }
   }
   ```

3. **Registration** (`main.go`)
   ```go
   s.AddTool(tools.GetAPIResourcesTool(), handlers.GetAPIResources(client))
   ```

### GVR Caching

The Kubernetes client caches GroupVersionResource mappings to avoid repeated discovery API calls:
- First lookup uses discovery client to resolve Kind → GVR
- Subsequent lookups use in-memory cache with RWMutex protection
- Enables efficient dynamic resource operations

### Dynamic Client Usage

The `pkg/k8s/client.go` uses Kubernetes dynamic client to:
- Handle arbitrary resource types without static type definitions
- Support YAML and JSON manifests
- Enable create/update/delete operations for any resource kind
- Query resources with label and field selectors

### Transport Modes

Three transport modes for different integration scenarios:

- **stdio**: Synchronous request/response via stdin/stdout (used by VS Code MCP extension)
- **sse**: Server-Sent Events over HTTP (long-lived connections for web apps)
- **streamable-http**: Stateless HTTP with streaming responses per MCP specification

## Development Workflow

### Adding a New Tool

1. Define the tool in `tools/k8s.go` or `tools/helm.go`:
   - Use `mcp.NewTool()` with descriptive name
   - Define input parameters with types and descriptions
   - Mark required parameters with `mcp.Required()`

2. Implement the handler in `handlers/k8s.go` or `handlers/helm.go`:
   - Extract and validate arguments
   - Call the appropriate client method
   - Serialize response to JSON
   - Return `mcp.NewToolResultText(jsonString)`

3. Register in `main.go`:
   - Add to the appropriate section (K8s or Helm)
   - Respect read-only flag for write operations

### Configuration Priority

Command-line flags override environment variables:
- Flags: `--mode`, `--port`, `--read-only`, `--no-k8s`, `--no-helm`
- Environment: `SERVER_MODE`, `SERVER_PORT`
- Defaults: SSE mode on port 8080

### Docker Security

The Dockerfile implements security best practices:
- Multi-stage build to minimize image size
- Runs as non-root user (`appuser` UID 1001)
- Minimal Alpine base with only ca-certificates and curl
- Health check endpoint on root path
- Kubeconfig mounted read-only to `/home/appuser/.kube/config`

### VS Code Integration

The server integrates with VS Code via the MCP extension in stdio mode. Configuration goes in VS Code `settings.json`:

```json
{
  "mcp.mcpServers": {
    "k8s-mcp-server": {
      "command": "k8s-mcp-server",
      "args": ["--mode", "stdio", "--read-only"],
      "env": {
        "KUBECONFIG": "${env:HOME}/.kube/config"
      }
    }
  }
}
```
