# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Kubernetes MCP Server** - a Model Context Protocol server written in Go that provides tools for interacting with Kubernetes clusters and Helm through a standardized interface. It uses `mark3labs/mcp-go` for MCP protocol implementation.

**Module:** `github.com/reza-gholizade/k8s-mcp-server`
**Go Version:** 1.25.0

## Essential Commands

### Building and Running

```bash
# Install dependencies
go mod download

# Build the binary
go build -o k8s-mcp-server main.go

# Run in different modes
./k8s-mcp-server --mode stdio                    # Standard I/O (for CLI/VS Code MCP extension)
./k8s-mcp-server --mode sse                      # Server-Sent Events (default, port 8080)
./k8s-mcp-server --mode streamable-http          # Streamable HTTP (MCP spec compliant)

# Enable read-only mode (disables write operations)
./k8s-mcp-server --read-only

# Disable specific tool categories
./k8s-mcp-server --no-k8s                        # Only Helm tools
./k8s-mcp-server --no-helm                       # Only Kubernetes tools
```

### Docker

```bash
# Build Docker image
docker build -t k8s-mcp-server:latest .

# Run with kubeconfig mounted
docker run -p 8080:8080 -v ~/.kube/config:/home/appuser/.kube/config:ro k8s-mcp-server:latest

# Run in read-only mode
docker run -p 8080:8080 -v ~/.kube/config:/home/appuser/.kube/config:ro k8s-mcp-server:latest --read-only

# Use Docker Compose
docker compose up -d
```

### Environment Variables

- `SERVER_MODE`: Transport mode (stdio, sse, streamable-http)
- `SERVER_PORT`: Port for HTTP modes (default: 8080)
- `KUBECONFIG`: Path to kubeconfig file
- `KUBECONFIG_DATA`: Full kubeconfig content (as alternative to file)
- `KUBERNETES_SERVER`: API server URL
- `KUBERNETES_TOKEN`: Bearer token for authentication
- `KUBERNETES_CA_CERT` / `KUBERNETES_CA_CERT_PATH`: CA certificate
- `KUBERNETES_INSECURE`: Skip TLS verification

## High-Level Architecture

The project follows a **layered architecture** with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────┐
│                    main.go                               │
│  - MCP Server Initialization                            │
│  - Tool Registration (conditional based on flags)       │
│  - Transport Mode Selection (stdio/sse/streamable-http) │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                    tools/                                │
│  - k8s.go: Kubernetes tool schemas                      │
│  - helm.go: Helm tool schemas                           │
│  - Defines MCP tool definitions with parameters         │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                  handlers/                               │
│  - k8s.go: Kubernetes operation handlers                │
│  - helm.go: Helm operation handlers                     │
│  - Maps MCP requests to client operations               │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                    pkg/                                  │
│  - k8s/client.go: Kubernetes dynamic client              │
│  - helm/client.go: Helm v3 action client wrapper        │
│  - Abstracts API interactions                           │
└─────────────────────────────────────────────────────────┘
```

### Key Architectural Patterns

**Tool Registration Pattern (3-step process):**
1. Define tool schema in `tools/` (k8s.go or helm.go)
2. Implement handler in `handlers/` (k8s.go or helm.go)
3. Register in `main.go`

**Dynamic Client Usage:**
- Kubernetes client uses `dynamic.Interface` for arbitrary resource types
- No static type definitions needed
- Supports any Kubernetes resource via GVR (GroupVersionResource)

**GVR Caching:**
- Caches GroupVersionResource mappings to avoid repeated discovery API calls
- Thread-safe with RWMutex protection
- Improves performance for repeated operations

**Transport Modes:**
- **stdio**: For CLI integrations (VS Code MCP extension)
- **sse**: Server-Sent Events for web applications
- **streamable-http**: Stateless HTTP per MCP specification

## Directory Structure

- `main.go` - Entry point, MCP server initialization and tool registration
- `tools/` - MCP tool schema definitions
  - `k8s.go` - Kubernetes tool definitions (9 tools)
  - `helm.go` - Helm tool definitions (6 tools)
- `handlers/` - Business logic for tool handlers
  - `k8s.go` - Kubernetes operation handlers
  - `helm.go` - Helm operation handlers
- `pkg/` - Client implementations
  - `k8s/client.go` - Dynamic Kubernetes client with GVR caching (~751 lines)
  - `helm/client.go` - Helm v3 action client wrapper (~374 lines)
- `scripts/` - VS Code installation scripts
- `.github/workflows/` - CI/CD pipelines

## Kubernetes Authentication Priority

The server supports multiple authentication methods, tried in this order:

1. **KUBECONFIG_DATA** - Full kubeconfig content as environment variable
2. **KUBERNETES_SERVER + KUBERNETES_TOKEN** - API server URL and bearer token
3. **In-cluster** - Service account token (when running in a pod)
4. **KUBECONFIG** - File path to kubeconfig (defaults to ~/.kube/config)

## Available Tools

### Kubernetes Tools (read-only)
- `getAPIResources` - List all API resources in cluster
- `listResources` - List resources by type with filters
- `getResource` - Get specific resource details
- `describeResource` - Describe resource (kubectl describe style)
- `getPodsLogs` - Retrieve pod logs
- `getNodeMetrics` - Get node resource usage
- `getPodMetrics` - Get pod CPU/memory metrics
- `getEvents` - List cluster events
- `getIngresses` - Retrieve ingress resources

### Kubernetes Tools (write operations, disabled in read-only mode)
- `createOrUpdateResourceJSON` - Create/update from JSON
- `createOrUpdateResourceYAML` - Create/update from YAML
- `deleteResource` - Delete a resource
- `rolloutRestart` - Trigger rolling restart

### Helm Tools (read-only)
- `helmList` - List releases
- `helmGet` - Get release details
- `helmHistory` - Get release history
- `helmRepoList` - List repositories

### Helm Tools (write operations, disabled in read-only mode)
- `helmInstall` - Install chart
- `helmUpgrade` - Upgrade release
- `helmUninstall` - Uninstall release
- `helmRollback` - Rollback release
- `helmRepoAdd` - Add repository

## Adding a New Tool

1. **Define the tool** in `tools/k8s.go` or `tools/helm.go`:
   ```go
   func MyNewTool() mcp.Tool {
       return mcp.NewTool("myNewTool",
           mcp.WithDescription("..."),
           mcp.WithString("param", mcp.Required(), mcp.Description("...")))
   }
   ```

2. **Implement the handler** in `handlers/k8s.go` or `handlers/helm.go`:
   ```go
   func MyNewTool(client *k8s.Client) func(context.Context, mcp.CallToolRequest) (*mcp.CallToolResult, error) {
       return func(ctx context.Context, request mcp.CallToolRequest) (*mcp.CallToolResult, error) {
           // Extract args, call client, serialize response to JSON
           return mcp.NewToolResultText(jsonString), nil
       }
   }
   ```

3. **Register in `main.go`**:
   - Add to the appropriate section (K8s or Helm)
   - Respect `--read-only` flag for write operations
   - Respect `--no-k8s` or `--no-helm` flags

## Key Dependencies

- `github.com/mark3labs/mcp-go` v0.43.2 - MCP protocol implementation
- `helm.sh/helm/v3` v3.19.5 - Helm client library
- `k8s.io/client-go` v0.35.0 - Kubernetes client libraries
- `k8s.io/metrics` v0.35.0 - Metrics API for pod/node metrics
- `sigs.k8s.io/yaml` v1.6.0 - YAML handling

## Security Notes

- Docker container runs as non-root user (`appuser` UID 1001)
- Read-only mode available for safe operations
- Kubeconfig should be mounted read-only in Docker
- Uses minimal Alpine base image

## Configuration Priority

Command-line flags override environment variables:
- Flags: `--mode`, `--port`, `--read-only`, `--no-k8s`, `--no-helm`
- Environment: `SERVER_MODE`, `SERVER_PORT`
- Defaults: SSE mode on port 8080

## VS Code Integration

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
