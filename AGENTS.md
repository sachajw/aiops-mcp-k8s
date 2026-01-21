# AGENTS.md

This file guides agentic coding assistants working in this Kubernetes MCP server repository.

## Build Commands

```bash
# Install dependencies
go mod download

# Build binary
go build -o k8s-mcp-server main.go

# Run in different modes
./k8s-mcp-server --mode stdio                    # Standard I/O (CLI/VS Code MCP)
./k8s-mcp-server --mode sse --port 8080         # Server-Sent Events (default)
./k8s-mcp-server --mode streamable-http --port 8080  # MCP spec compliant
./k8s-mcp-server --read-only                     # Disable write operations
```

## Testing

This repository currently has no test suite. Tests should be added using Go's testing framework (`_test.go` files).

## Code Style Guidelines

### Imports
- Standard library imports first, grouped with blank lines
- External package imports second, grouped with blank lines
- Local package imports third (github.com/reza-gholizade/k8s-mcp-server/...)
- Use alias imports only for disambiguation (e.g., `corev1 "k8s.io/api/core/v1"`)

### Naming Conventions
- **Packages**: lowercase, single word (main, tools, handlers, pkg/k8s, pkg/helm)
- **Exported functions/types**: PascalCase (`GetAPIResources`, `Client`)
- **Unexported functions/types**: camelCase (`getStringArg`, `customRESTClientGetter`)
- **Functions returning tools/handlers**: PascalCase with descriptive names (e.g., `ListResourcesTool`)
- **Constants**: PascalCase

### Comments
- Package comment on first line describing purpose
- Function comments immediately above definition
- Use `//` style for all comments (not `/* */`)
- Multi-line comments use consecutive `//` lines

### Error Handling
- Always check errors immediately
- Wrap errors with `fmt.Errorf("%s: %w", context, err)` using `%w` verb
- Return errors directly from handlers; let caller handle serialization
- Return `nil, error` on failure

### Handler Pattern
Tool handlers follow this signature:
```go
func HandlerName(client *k8s.Client) func(context.Context, mcp.CallToolRequest) (*mcp.CallToolResult, error)
```
1. Return closure accepting `ctx` and `request`
2. Extract args using helper functions: `getStringArg`, `getBoolArg`, `getRequiredStringArg`
3. Call client method with context
4. Serialize response to JSON
5. Return `mcp.NewToolResultText(jsonString), nil` or error

### Tool Registration Pattern
Three-step process:
1. **Define tool** in `tools/k8s.go` or `tools/helm.go`
2. **Implement handler** in `handlers/k8s.go` or `handlers/helm.go`
3. **Register** in `main.go` with appropriate flags (respect `--read-only`, `--no-k8s`, `--no-helm`)

### Type Safety
- Use explicit types for struct fields
- Prefer typed constants over magic values
- Use context.Context as first parameter for client calls
- Always type-assert args: `val, ok := args[key].(string)`

### Formatting
- Use `gofmt` for standard Go formatting (though no linter is configured)
- Tabs for indentation (Go standard)
- No trailing whitespace
- One blank line between functions/types
