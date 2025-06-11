# SQL API Comparison Results

## Test Query
```sql
SELECT '2026-01-01'::timestamptz
```

## 1. Query API (sql_service/query)

### Request
```http
POST http://localhost:8080/v1/instances/metadb/databases/postgres:query
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "statement": "SELECT '2026-01-01'::timestamptz",
  "dataSourceId": "2d054f02-1dfb-488d-9e2a-5521358ed9b6",
  "limit": 1000
}
```

### Response
```json
{
  "results": [
    {
      "columnNames": ["timestamptz"],
      "columnTypeNames": ["TIMESTAMPTZ"],
      "rows": [
        {
          "values": [
            {
              "timestampTzValue": {
                "googleTimestamp": "2025-12-31T16:00:00Z",
                "zone": "CST",
                "offset": 28800,
                "accuracy": 6
              }
            }
          ]
        }
      ],
      "rowsCount": "1",
      "masked": [false],
      "sensitive": [false],
      "latency": "0.005491334s",
      "statement": "WITH result AS (\nSELECT '2026-01-01'::timestamptz\n) SELECT * FROM result LIMIT 1000;",
      "allowExport": true
    }
  ]
}
```

### Key Observations
- Successfully executed with READ_ONLY data source
- Returns timestamptz value as `2025-12-31T16:00:00Z` (UTC) with CST zone info
- Query was wrapped with CTE: `WITH result AS (...) SELECT * FROM result LIMIT 1000;`
- Execution time: ~5.5ms
- Export allowed

## 2. AdminExecute API (sql_service/adminExecute)

### Request
```http
GET http://localhost:8080/v1:adminExecute?name=instances/metadb/databases/postgres&statement=SELECT%20'2026-01-01'::timestamptz&limit=1000
Authorization: Bearer {{token}}
```

### Response
- **Status**: 200 OK
- **Issue**: Streaming endpoint returns empty response when tested with regular HTTP clients
- **Note**: This endpoint requires WebSocket/gRPC streaming client for proper communication

### Expected Response (based on proto definition)
Would return similar `QueryResult` structure but through streaming protocol:
```json
{
  "results": [
    {
      "columnNames": ["timestamptz"],
      "columnTypeNames": ["TIMESTAMPTZ"],
      "rows": [...],
      // Similar structure to Query API
    }
  ]
}
```

## Key Differences Between APIs

| Feature | Query API | AdminExecute API |
|---------|-----------|------------------|
| **HTTP Method** | POST | GET (streaming) |
| **Request Format** | JSON body | Query parameters |
| **Protocol** | Standard HTTP | gRPC streaming/WebSocket |
| **Data Source** | Must specify (READ_ONLY works) | Uses admin connection |
| **Permissions** | `bb.databases.get` | `bb.sql.admin` |
| **Connection** | Per-request | Persistent (session-based) |
| **Use Case** | Read queries | Admin operations |
| **Query Wrapping** | Adds CTE wrapper | Executes as-is |

## Testing Limitations

1. **httpyac/curl**: Cannot properly handle the streaming response from adminExecute
2. **Proper testing requires**:
   - grpcurl for command-line gRPC testing
   - WebSocket client for browser-based testing
   - Native gRPC client implementation

## Conclusion

Both APIs would return the same timestamp value (`2026-01-01` as `2025-12-31T16:00:00Z` in UTC), but through different protocols:
- **Query API**: Standard HTTP REST endpoint, easy to test, suitable for read operations
- **AdminExecute API**: Streaming gRPC endpoint, requires special clients, provides admin privileges and persistent connections