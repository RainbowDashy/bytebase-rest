# REST API Testing Directory

This directory is for testing Bytebase APIs.

## Project Structure

- Bytebase source code is located at `~/bytebase` (also accessible as `../bytebase/` or `./bytebase/` from this directory)
- This directory (`~/rest/`) contains HTTP test files for API testing
- `./bytebase/` directory contains symlinked full Bytebase source code for Claude's analysis

## Technology Stack

- Bytebase uses protobuf and gRPC
- Uses grpc-web for web clients
- Generates HTTP gateway to allow HTTP requests alongside gRPC
- API definitions and implementations can be found in the `~/bytebase` or `../bytebase/` directory

## API Structure

- **Base URL**: Uses `{{url}}` variable from `login.env` (http://localhost:8080)
- **Authentication**: Bearer token via `Authorization: Bearer {{token}}`
- **API Version**: All endpoints use `/v1/` prefix
- **Content-Type**: `application/json`

## Update Pattern

Bytebase follows a consistent PATCH pattern for updates:

- Uses `updateMask` with `paths` array to specify which fields to update
- Resource names follow format: `{resource-type}/{resource-id}`
- Example: `instances/my-instance`, `environments/prod`

## Key Resources

- **Instances**: `/v1/instances/{instance-id}`
- **Projects**: `/v1/projects/{project}`
- **Environments**: `/v1/environments/{environment}`
- **Issues**: `/v1/projects/{project}/issues/{issue}`
- **Task runs**: `/v1/projects/{project}/rollouts/{rollout}/stages/{stage}/tasks/{task}/taskRuns/{taskRun}`

## Protobuf Definitions

- v1 API services: `./bytebase/proto/v1/v1/*.proto`
- Store definitions: `./bytebase/proto/store/store/*.proto`
- Generated Go code: `./bytebase/proto/generated-go/v1/*.pb.go`
- HTTP gateway generates REST endpoints from gRPC/protobuf definitions

## Source Code Access

- Full Bytebase source code available at `./bytebase/`
- Backend API implementations: `./bytebase/backend/api/v1/*_service.go`
- Store layer: `./bytebase/backend/store/*.go`
- HTTP gateway generated code: `./bytebase/proto/generated-go/v1/*.pb.gw.go`

## Update Mask Paths

When using PATCH requests with `updateMask` parameters, find the correct field paths in the Go implementation:

- **Setting updates**: Check `./bytebase/backend/api/v1/setting_service.go` in the `UpdateSetting` method
- **Other resources**: Check respective `*_service.go` files for `Update*` methods
- **Pattern**: Look for `request.UpdateMask.Paths` loops and `case` statements for valid field paths
- **Example**: `value.workspace_profile_setting_value.database_change_mode` for workspace database change mode

The Go code contains the authoritative field paths that the server accepts, not the proto definitions.

## HTTP Gateway Generation Rules

**Critical Pattern**: Proto `google.api.http` options determine HTTP mapping:

```proto
rpc UpdateInstance(UpdateInstanceRequest) returns (Instance) {
  option (google.api.http) = {
    patch: "/v1/{instance.name=instances/*}"
    body: "instance"
  };
}
```

**Translation Rules:**

- **HTTP Method**: `patch:` → HTTP PATCH
- **URL Path**: `"/v1/{instance.name=instances/*}"` extracts ID from path
- **Request Body**: `body: "instance"` → `Instance` message becomes JSON body directly
- **Query Parameters**: Fields NOT in body (like `update_mask`) become query params
- **Field Naming**: `update_mask` → `updateMask` (camelCase conversion)

**Key Insight**: When `body: "field_name"` is specified, that field becomes the direct request body content, NOT wrapped in the parent request message.

**Example Mapping:**
```proto
message UpdateInstanceRequest {
  Instance instance = 1;           // → becomes request body
  FieldMask update_mask = 2;       // → becomes ?updateMask= query param
}
```

**Common Body Options:**
- `body: "*"` = entire request message becomes body
- `body: "field_name"` = specific field becomes body
- No body specified = all fields become query parameters

## Usage

- `.http` files in this directory contain REST API test requests
- Environment variables and secrets are stored in `.env` and `mysecrets`

## Testing Requests

Use httpyac CLI to test HTTP requests:

```bash
# Test a specific request by name
httpyac send ~/rest/instance.http --env ~/rest/.env --name "Update instance environment" --verbose

# Test a specific request by line number
httpyac send ~/rest/instance.http --env ~/rest/.env --line 1

# Test all requests in a file
httpyac send ~/rest/instance.http --env ~/rest/.env --all

# General pattern
httpyac send <file.http> --env <env-file> --name "<request-name>" --verbose
httpyac send <file.http> --env <env-file> --line <line-number>
```

**Note**: Always use httpyac to test requests after writing them to verify correct API structure and authentication.

## gRPC Testing with httpyac

httpyac supports native gRPC testing, which is essential for Bytebase's streaming endpoints and direct gRPC APIs:

### Basic gRPC Syntax

```http
GRPC {{url}}/[fully.qualified.ServiceName]/[MethodName]
Authorization: Bearer {{token}}

{
  "field1": "value1",
  "field2": "value2"
}
```

### Key Points for gRPC Testing

1. **Service Names**: Use exact service names from reflection (e.g., `bytebase.v1.SQLService`)
2. **Field Naming**: Use snake_case for fields in gRPC (e.g., `data_source_id` not `dataSourceId`)
3. **Streaming Support**: httpyac handles bidirectional streaming endpoints like AdminExecute
4. **Service Discovery**: Run with `--verbose` to see available services via reflection

### Examples

```http
# Standard Request/Response
GRPC {{url}}/bytebase.v1.SQLService/Query
Authorization: Bearer {{token}}

{
  "name": "instances/metadb/databases/postgres",
  "statement": "SELECT NOW()",
  "data_source_id": "2d054f02-1dfb-488d-9e2a-5521358ed9b6",
  "limit": 1000
}

###

# Streaming Endpoint
GRPC {{url}}/bytebase.v1.SQLService/AdminExecute  
Authorization: Bearer {{token}}

{
  "name": "instances/metadb/databases/postgres",
  "statement": "SELECT NOW()",
  "limit": 1000
}
```

### When to Use gRPC Mode

- **Streaming endpoints**: Required for AdminExecute and other streaming APIs
- **Better performance**: Direct gRPC without HTTP gateway overhead
- **Type validation**: Ensures correct protobuf types
- **Debugging**: When HTTP gateway behaves unexpectedly

### Response Format Differences

| Feature | HTTP REST | gRPC |
|---------|-----------|------|
| Field naming | camelCase | snake_case |
| Numbers | JSON numbers/strings | Protobuf objects |
| Timestamps | ISO 8601 strings | Nested objects |
| Duration | String format | Nanos field |

## Token Management

If you encounter "access token expired" error:

1. Send the login request to refresh the token:

   ```bash
   httpyac send ~/rest/login.http --env ~/rest/.env --line 1
   ```

2. Copy the new token from the response and update it in `.env` file:

   ```env
   token=<new-token-value>
   ```

The login endpoint provides fresh authentication tokens that should be stored in `.env` for subsequent API requests.

**Note for Claude**: Always refresh tokens automatically when they expire by:

1. Running the login request
2. Updating the token in `.env` file  
3. Retrying the original request
