# Bytebase REST API Testing Guide

This directory contains HTTP test files for testing Bytebase APIs using httpyac.

## Quick Start

```bash
# Test a specific request
httpyac send ~/rest/instance.http --env ~/rest/.env --name "List all instances"

# Test by line number
httpyac send ~/rest/instance.http --env ~/rest/.env --line 1

# Test all requests in a file
httpyac send ~/rest/instance.http --env ~/rest/.env --all
```

## Project Structure

```text
~/rest/                    # This directory - API test files
├── .env                   # Environment variables (url, token)
├── *.http                 # HTTP/gRPC test files
├── CLAUDE.md             # This file - project documentation
└── bytebase/             # Symlink to Bytebase source code

~/bytebase/               # Bytebase source code
├── proto/v1/v1/*.proto   # API protobuf definitions
├── backend/api/v1/*      # API implementations
└── backend/store/*       # Database layer
```

## Technology Stack

- **Protocol**: gRPC with HTTP gateway
- **API Version**: v1
- **Authentication**: Bearer token
- **Content-Type**: application/json (HTTP), application/grpc (gRPC)

## API Endpoints

### Base Configuration

- **URL**: `{{url}}` = `http://localhost:8080`
- **Auth**: `Authorization: Bearer {{token}}`
- **Version**: All endpoints prefixed with `/v1/`

### Key Resources

| Resource | Endpoint Pattern | Example |
|----------|-----------------|---------|
| Instances | `/v1/instances/{instance-id}` | `instances/metadb` |
| Projects | `/v1/projects/{project}` | `projects/db333` |
| Databases | `/v1/instances/{instance}/databases/{database}` | `instances/metadb/databases/postgres` |
| Environments | `/v1/environments/{environment}` | `environments/prod` |
| Issues | `/v1/projects/{project}/issues/{issue}` | `projects/db333/issues/101` |
| Settings | `/v1/settings/{setting}` | `settings/bb.workspace.profile` |

## Common Patterns

### PATCH Updates

```http
PATCH {{url}}/v1/instances/sample-instance?updateMask=environment
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "name": "instances/sample-instance",
  "environment": "environments/prod"
}
```

**Key Points:**

- Always include resource `name` in body
- Use `updateMask` query parameter (not in body)
- Field paths use dots: `value.workspace_profile_setting_value.database_change_mode`

### Finding Update Mask Paths

1. Check the service implementation file (e.g., `setting_service.go`)
2. Look for `UpdateXXX` method
3. Find `request.UpdateMask.Paths` loop
4. Valid paths are in the `case` statements

## Testing Modes

### HTTP REST Testing

```http
### Get instance details
GET {{url}}/v1/instances/metadb
Authorization: Bearer {{token}}

### Update instance
PATCH {{url}}/v1/instances/metadb?updateMask=environment
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "name": "instances/metadb",
  "environment": "environments/prod"
}
```

### gRPC Testing

```http
### Query via gRPC
GRPC {{url}}/bytebase.v1.SQLService/Query
Authorization: Bearer {{token}}

{
  "name": "instances/metadb/databases/postgres",
  "statement": "SELECT NOW()",
  "data_source_id": "2d054f02-1dfb-488d-9e2a-5521358ed9b6",
  "limit": 1000
}

### Streaming endpoint via gRPC
GRPC {{url}}/bytebase.v1.SQLService/AdminExecute
Authorization: Bearer {{token}}

{
  "name": "instances/metadb/databases/postgres",
  "statement": "SELECT NOW()",
  "limit": 1000
}
```

### When to Use Each Mode

| Use Case | HTTP REST | gRPC |
|----------|-----------|------|
| Simple CRUD operations | Preferred | Works |
| Streaming endpoints | Limited | Required |
| Performance testing | Gateway overhead | Direct |
| Debugging proto issues | JSON conversion | Native |

## Authentication and Token Management

### Automatic Token Refresh (for Claude)

When encountering "access token expired":

1. **Run login request**:

   ```bash
   httpyac send ~/rest/login.http --env ~/rest/.env --line 1
   ```

2. **Extract token from response** and update `.env`:

   ```env
   token=<new-token-value>
   ```

3. **Retry the original request**

### Manual Token Refresh

```bash
# Get new token
curl -X POST http://localhost:8080/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"demo@bytebase.com","password":"1024"}'

# Update .env file
echo "token=<new-token>" > ~/rest/.env
```

## Advanced Topics

### HTTP Gateway Translation Rules

Proto options determine HTTP mapping:

```proto
rpc UpdateInstance(UpdateInstanceRequest) returns (Instance) {
  option (google.api.http) = {
    patch: "/v1/{instance.name=instances/*}"
    body: "instance"
  };
}
```

**Translation:**

- `patch:` → HTTP PATCH method
- `{instance.name=instances/*}` → Extract ID from URL
- `body: "instance"` → Instance field becomes request body
- Other fields → Query parameters
- `snake_case` → `camelCase` in JSON

### Response Format Differences

| Feature | HTTP REST | gRPC |
|---------|-----------|------|
| Field naming | camelCase | snake_case |
| Empty fields | Omitted | Default values |
| Timestamps | ISO 8601 strings | `{seconds, nanos}` |
| Duration | "1.234s" | `{nanos: 1234000000}` |
| Int64 | String or number | `{low, high, unsigned}` |

### Common Data Source IDs

When using SQL Query API, you need a data source ID:

1. **Get instance details**:

   ```bash
   curl -X GET "{{url}}/v1/instances/metadb" -H "Authorization: Bearer {{token}}"
   ```

2. **Find data source**:
   - Type: `ADMIN` (read-write, not queryable via Query API)
   - Type: `READ_ONLY` (read-only, queryable)

3. **Use the READ_ONLY data source ID in queries**

## Useful Commands

### httpyac Tips

```bash
# See all available gRPC services
httpyac send file.http --env .env --verbose | grep "grpc reflection"

# Test with specific environment
httpyac send file.http --env production.env

# Dry run (see what would be sent)
httpyac send file.http --env .env --dry-run

# Output to file
httpyac send file.http --env .env --output response.json
```

### Debugging

```bash
# Check if Bytebase is running
curl http://localhost:8080/v1/actuator/health

# List available services (gRPC reflection)
grpcurl -plaintext localhost:8080 list

# Describe a service
grpcurl -plaintext localhost:8080 describe bytebase.v1.SQLService
```

## Finding Information

### Proto Definitions

- Service definitions: `./bytebase/proto/v1/v1/*.proto`
- Request/Response messages: Same proto files
- Store types: `./bytebase/proto/store/*.proto`

### Implementation

- API handlers: `./bytebase/backend/api/v1/*_service.go`
- Business logic: Handler methods in service files
- Database queries: `./bytebase/backend/store/*.go`

### Generated Code

- HTTP gateway: `./bytebase/proto/generated-go/v1/*.pb.gw.go`
- gRPC stubs: `./bytebase/proto/generated-go/v1/*.pb.go`

## Testing Checklist

When creating new API tests:

- [ ] Use correct resource name format
- [ ] Include authentication header
- [ ] For PATCH: Use updateMask query param
- [ ] For gRPC: Use snake_case field names
- [ ] Test error cases (404, 403, 400)
- [ ] Verify response structure
- [ ] Document complex requests with comments

## Common Issues

### "data source id is required"

- Query API requires `dataSourceId` field
- Use READ_ONLY type data source
- Get ID from instance details

### "access token expired"

- Tokens expire after ~24 hours
- Refresh using login endpoint
- Update .env file

### Empty response from streaming endpoint

- Use gRPC mode for streaming endpoints
- HTTP gateway may not handle streaming properly
- Example: AdminExecute requires gRPC

### "Service not found" in gRPC

- Use full service name: `bytebase.v1.ServiceName`
- Check available services with `--verbose`
- Case sensitive

## Notes for Claude

1. **Always check file existence** before reading
2. **Refresh tokens automatically** when they expire
3. **Use gRPC mode** for streaming endpoints
4. **Read source code** to find correct update paths
5. **Test requests** after writing them
6. **Update this file** when discovering new patterns