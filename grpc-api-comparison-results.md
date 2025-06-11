# Complete SQL API Comparison Results (HTTP + gRPC)

## Test Query

```sql
SELECT '2026-01-01'::timestamptz
```

## 1. Query API - HTTP REST

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

## 2. Query API - gRPC

### Request
```http
GRPC http://localhost:8080/bytebase.v1.SQLService/Query
Authorization: Bearer {{token}}

{
  "name": "instances/metadb/databases/postgres",
  "statement": "SELECT '2026-01-01'::timestamptz",
  "data_source_id": "2d054f02-1dfb-488d-9e2a-5521358ed9b6",
  "limit": 1000
}
```

### Response
```json
{
  "results": [
    {
      "column_names": ["timestamptz"],
      "column_type_names": ["TIMESTAMPTZ"],
      "rows": [
        {
          "values": [
            {
              "timestamp_tz_value": {
                "google_timestamp": {
                  "seconds": {
                    "low": 1767196800,
                    "high": 0,
                    "unsigned": false
                  }
                },
                "zone": "CST",
                "offset": 28800,
                "accuracy": 6
              }
            }
          ]
        }
      ],
      "masked": [false],
      "sensitive": [false],
      "latency": {
        "nanos": 798000
      },
      "statement": "WITH result AS (\nSELECT '2026-01-01'::timestamptz\n) SELECT * FROM result LIMIT 1000;",
      "rows_count": {
        "low": 1,
        "high": 0,
        "unsigned": false
      },
      "allow_export": true
    }
  ]
}
```

## 3. AdminExecute API - HTTP REST

### Request
```http
GET http://localhost:8080/v1:adminExecute?name=instances/metadb/databases/postgres&statement=SELECT%20'2026-01-01'::timestamptz&limit=1000
Authorization: Bearer {{token}}
```

### Response
- **Status**: 200 OK
- **Issue**: Streaming endpoint returns empty response with standard HTTP clients
- **Note**: Requires WebSocket/gRPC streaming client

## 4. AdminExecute API - gRPC

### Request
```http
GRPC http://localhost:8080/bytebase.v1.SQLService/AdminExecute
Authorization: Bearer {{token}}

{
  "name": "instances/metadb/databases/postgres",
  "statement": "SELECT '2026-01-01'::timestamptz",
  "limit": 1000
}
```

### Response
```json
{
  "results": [
    {
      "column_names": ["timestamptz"],
      "column_type_names": ["TIMESTAMPTZ"],
      "rows": [
        {
          "values": [
            {
              "timestamp_tz_value": {
                "google_timestamp": {
                  "seconds": {
                    "low": 1767196800,
                    "high": 0,
                    "unsigned": false
                  }
                },
                "zone": "CST",
                "offset": 28800,
                "accuracy": 6
              }
            }
          ]
        }
      ],
      "latency": {
        "nanos": 624458
      },
      "statement": "SELECT '2026-01-01'::timestamptz",
      "rows_count": {
        "low": 1,
        "high": 0,
        "unsigned": false
      }
    }
  ]
}
```

## Key Observations

### 1. Timestamp Value Consistency
All APIs return the same timestamp value:
- **Human readable**: `2026-01-01` CST
- **UTC representation**: `2025-12-31T16:00:00Z` 
- **Unix timestamp**: 1767196800 seconds
- **Timezone offset**: 28800 seconds (8 hours for CST)

### 2. Response Format Differences

| Feature | HTTP REST | gRPC |
|---------|-----------|------|
| **Field naming** | camelCase (e.g., `columnNames`) | snake_case (e.g., `column_names`) |
| **Timestamp format** | ISO 8601 string | Protobuf timestamp object |
| **Latency format** | String (e.g., "0.005491334s") | Nanos object |
| **Row count** | String | Long object |

### 3. Query Execution Differences

| Feature | Query API | AdminExecute API |
|---------|-----------|------------------|
| **Query wrapping** | Adds `WITH result AS (...) SELECT * FROM result LIMIT X;` | Executes as-is |
| **Execution time** | ~5.5ms (HTTP), ~0.8ms (gRPC) | ~0.6ms (gRPC) |
| **Export permission** | Explicitly returned | Not returned |
| **Masked/Sensitive fields** | Returned | Not returned in AdminExecute |

### 4. Protocol Support

| API | HTTP REST | gRPC | Streaming |
|-----|-----------|------|-----------|
| **Query** | ✅ Full support | ✅ Full support | ❌ Not streaming |
| **AdminExecute** | ⚠️ Limited (empty response) | ✅ Full support | ✅ Bidirectional streaming |

## Conclusion

1. **httpyac gRPC mode successfully handles both APIs** when using the proper gRPC protocol
2. **AdminExecute via HTTP REST requires special streaming clients** (WebSocket, Server-Sent Events)
3. **Both APIs return identical timestamp values** but with different response formats
4. **AdminExecute is faster** (~0.6ms vs ~0.8-5.5ms) and executes queries without modification
5. **Query API adds safety wrapper** and provides more metadata (export permission, masked/sensitive flags)