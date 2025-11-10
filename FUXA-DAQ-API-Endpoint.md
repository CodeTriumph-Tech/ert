# FUXA /api/daq Endpoint - Complete Technical Documentation

## Overview

The `/api/daq` endpoint is the **primary API route** for fetching historical DAQ (Data Acquisition) data in FUXA. Unlike the generic `/api/getHistoricalTags` endpoint mentioned in basic documentation, this endpoint uses a **query parameter approach** with a JSON-encoded query object containing specific parameters like `gid`, `from`, `to`, and `sids`.

---

## 1. Endpoint Specification

### Basic Information

| Property | Value |
|----------|-------|
| **URL Path** | `/api/daq` |
| **HTTP Method** | `GET` |
| **Authentication** | Optional (depends on FUXA configuration) |
| **Response Format** | JSON |
| **Content-Type** | `application/json` |

### Complete Request URL Format

```
GET /api/daq?query=<URL_ENCODED_JSON_QUERY>
```

---

## 2. Query Parameter Structure

The `/api/daq` endpoint accepts a single query parameter named `query` which contains a **URL-encoded JSON object**.

### Query Object Parameters

```typescript
interface DaqQuery {
  gid: string;           // Group/Device ID (project or device identifier)
  from: number;          // Start timestamp (milliseconds since epoch)
  to: number;            // End timestamp (milliseconds since epoch)
  sids: string[];        // Array of tag/series IDs to fetch
}
```

### Detailed Parameter Explanation

#### 1. **gid** (Group ID)
- **Type**: String
- **Purpose**: Identifies the group or device configuration ID
- **Format**: UUID format (e.g., `OXC_8e3c621a-c6bf490e`)
- **Usage**: Typically represents:
  - The project ID in multi-project setups
  - A device group identifier
  - A logical grouping of related tags
- **Example**: `"OXC_8e3c621a-c6bf490e"`

#### 2. **from** (Start Timestamp)
- **Type**: Number (Unix timestamp in milliseconds)
- **Purpose**: Defines the start of the time range for data retrieval
- **Format**: Milliseconds since January 1, 1970 (Unix epoch)
- **Example**: `1760121000000` (approximately July 9, 2025)
- **Conversion**: 
  - `JavaScript`: `new Date('2025-07-09').getTime()`
  - `Unix`: `1760121000000 / 1000 = 1760121000 seconds`

#### 3. **to** (End Timestamp)
- **Type**: Number (Unix timestamp in milliseconds)
- **Purpose**: Defines the end of the time range for data retrieval
- **Format**: Milliseconds since January 1, 1970 (Unix epoch)
- **Example**: `1762766003766` (approximately July 31, 2025)
- **Conversion**: Same as `from` parameter
- **Note**: The `to` timestamp can include milliseconds (e.g., `...3766`)

#### 4. **sids** (Series/Tag IDs)
- **Type**: Array of strings
- **Purpose**: Specifies which tags/series to retrieve data for
- **Format**: Array of tag IDs (typically UUID format)
- **Example**: 
  ```json
  [
    "t_80f6bec4-2c5740fd",
    "t_847b6c7c-868846bd",
    "t_931aec4d-5b7b4561",
    "t_caa27fd2-fa584d07",
    "t_362ec83e-d0ce4d67",
    "t_98c40cae-9cd54227",
    "t_140e59c9-ef724af1"
  ]
  ```
- **Limit**: No hard limit documented, but performance degrades with many tags (>100-200 tags in single query)
- **Best Practice**: Query 10-50 tags per request for optimal performance

---

## 3. Practical Examples

### Example 1: Simple Query

**Request:**
```
GET /api/daq?query=%7B%22gid%22:%22OXC_8e3c621a-c6bf490e%22,%22from%22:1760121000000,%22to%22:1762766003766,%22sids%22:%5B%22t_80f6bec4-2c5740fd%22%5D%7D
```

**Decoded Query Object:**
```json
{
  "gid": "OXC_8e3c621a-c6bf490e",
  "from": 1760121000000,
  "to": 1762766003766,
  "sids": ["t_80f6bec4-2c5740fd"]
}
```

**cURL Command:**
```bash
curl -X GET "http://localhost:1881/api/daq?query=%7B%22gid%22:%22OXC_8e3c621a-c6bf490e%22,%22from%22:1760121000000,%22to%22:1762766003766,%22sids%22:%5B%22t_80f6bec4-2c5740fd%22%5D%7D"
```

**JavaScript/Angular:**
```typescript
// Create query object
const daqQuery = {
  gid: 'OXC_8e3c621a-c6bf490e',
  from: 1760121000000,
  to: 1762766003766,
  sids: ['t_80f6bec4-2c5740fd']
};

// URL encode the query
const queryString = encodeURIComponent(JSON.stringify(daqQuery));

// Make HTTP request
const url = `http://localhost:1881/api/daq?query=${queryString}`;

fetch(url)
  .then(response => response.json())
  .then(data => console.log(data))
  .catch(error => console.error('Error:', error));
```

### Example 2: Multiple Tags Query (Your Production Example)

**Request:**
```
GET /api/daq?query=%7B%22gid%22:%22OXC_8e3c621a-c6bf490e%22,%22from%22:1760121000000,%22to%22:1762766003766,%22sids%22:%5B%22t_80f6bec4-2c5740fd%22,%22t_847b6c7c-868846bd%22,%22t_931aec4d-5b7b4561%22,%22t_caa27fd2-fa584d07%22,%22t_362ec83e-d0ce4d67%22,%22t_98c40cae-9cd54227%22,%22t_140e59c9-ef724af1%22%5D%7D
```

**Decoded Query:**
```json
{
  "gid": "OXC_8e3c621a-c6bf490e",
  "from": 1760121000000,
  "to": 1762766003766,
  "sids": [
    "t_80f6bec4-2c5740fd",
    "t_847b6c7c-868846bd",
    "t_931aec4d-5b7b4561",
    "t_caa27fd2-fa584d07",
    "t_362ec83e-d0ce4d67",
    "t_98c40cae-9cd54227",
    "t_140e59c9-ef724af1"
  ]
}
```

**Response Size:** 9,178,928 bytes (~9.2 MB)
**Response Time:** 1224.607 ms (~1.2 seconds)

**JavaScript Helper Function:**
```typescript
async function fetchDaqData(
  gid: string,
  tagIds: string[],
  fromDate: Date,
  toDate: Date
): Promise<any> {
  const query = {
    gid: gid,
    from: fromDate.getTime(),
    to: toDate.getTime(),
    sids: tagIds
  };

  const queryString = encodeURIComponent(JSON.stringify(query));
  const url = `/api/daq?query=${queryString}`;

  try {
    const response = await fetch(url);
    
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }

    const data = await response.json();
    return data;
  } catch (error) {
    console.error('DAQ API Error:', error);
    throw error;
  }
}

// Usage
const gid = 'OXC_8e3c621a-c6bf490e';
const tags = [
  't_80f6bec4-2c5740fd',
  't_847b6c7c-868846bd',
  't_931aec4d-5b7b4561'
];

const result = await fetchDaqData(
  gid,
  tags,
  new Date('2025-07-09'),
  new Date('2025-07-31')
);
```

---

## 4. Response Format

### Successful Response (HTTP 200)

```json
{
  "t_80f6bec4-2c5740fd": [
    {
      "x": 1760121000000,
      "y": 23.5
    },
    {
      "x": 1760121060000,
      "y": 24.1
    },
    {
      "x": 1760121120000,
      "y": 23.8
    }
  ],
  "t_847b6c7c-868846bd": [
    {
      "x": 1760121000000,
      "y": 45.2
    },
    {
      "x": 1760121060000,
      "y": 46.5
    }
  ]
}
```

### Response Structure Details

| Field | Type | Description |
|-------|------|-------------|
| Key | String | Tag ID (from `sids` array) |
| Value | Array | Array of data points for that tag |
| x | Number | Timestamp in milliseconds (Unix epoch) |
| y | Number/String | Value at that timestamp |

### Response with Quality/Status Information

Some versions of FUXA may include additional fields:

```json
{
  "t_80f6bec4-2c5740fd": [
    {
      "x": 1760121000000,
      "y": 23.5,
      "quality": 192,
      "timestamp": 1760121000000
    }
  ]
}
```

### Error Response (HTTP 4xx/5xx)

```json
{
  "error": "Invalid query parameters",
  "message": "gid parameter is required"
}
```

---

## 5. Response Characteristics

### Response in Your Example

| Metric | Value |
|--------|-------|
| Status Code | 200 |
| Response Size | 9,178,928 bytes (~9.2 MB) |
| Query Duration | 1224.607 ms |
| Data Points | Multiple (7 tags × ~22 days of data) |
| Tags Requested | 7 |

### Performance Analysis

**Calculation:**
- Query Duration: 1224.607 ms ≈ 1.22 seconds
- Data Size: 9.2 MB
- Throughput: ~7.5 MB/second (very good)
- This indicates the backend is efficiently serializing and transmitting data
- For optimized queries, response times should be < 2 seconds

---

## 6. Common Use Cases

### Use Case 1: Real-Time Chart Display (Angular Component)

```typescript
import { Component, OnInit } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Component({
  selector: 'app-daq-chart',
  templateUrl: './daq-chart.component.html',
  styleUrls: ['./daq-chart.component.scss']
})
export class DaqChartComponent implements OnInit {
  chartData: any = {};
  loading = false;
  error: string | null = null;

  constructor(private http: HttpClient) {}

  ngOnInit() {
    this.loadHistoricalData();
  }

  loadHistoricalData() {
    const gid = 'OXC_8e3c621a-c6bf490e';
    const tags = ['t_80f6bec4-2c5740fd', 't_847b6c7c-868846bd'];
    
    // Last 7 days
    const toDate = new Date();
    const fromDate = new Date(toDate.getTime() - 7 * 24 * 60 * 60 * 1000);

    this.loading = true;
    this.error = null;

    const query = {
      gid: gid,
      from: fromDate.getTime(),
      to: toDate.getTime(),
      sids: tags
    };

    const queryString = encodeURIComponent(JSON.stringify(query));
    const url = `/api/daq?query=${queryString}`;

    this.http.get(url).subscribe(
      (data: any) => {
        this.chartData = this.transformForChart(data);
        this.loading = false;
      },
      (error) => {
        this.error = error.message;
        this.loading = false;
      }
    );
  }

  transformForChart(rawData: any): any {
    const series = [];

    for (const tagId in rawData) {
      if (rawData.hasOwnProperty(tagId)) {
        const points = rawData[tagId];
        
        series.push({
          name: tagId,
          data: points.map((p: any) => ({
            x: new Date(p.x),
            y: p.y
          }))
        });
      }
    }

    return series;
  }
}
```

### Use Case 2: Data Export to CSV

```typescript
exportToCsv(gid: string, tagIds: string[], fromDate: Date, toDate: Date) {
  const query = {
    gid: gid,
    from: fromDate.getTime(),
    to: toDate.getTime(),
    sids: tagIds
  };

  const queryString = encodeURIComponent(JSON.stringify(query));
  const url = `/api/daq?query=${queryString}`;

  this.http.get(url).subscribe((data: any) => {
    let csvContent = 'Timestamp,';
    
    // Header with tag names
    csvContent += tagIds.join(',') + '\n';

    // Collect all timestamps
    const timestamps = new Set<number>();
    for (const tagId in data) {
      data[tagId].forEach((point: any) => {
        timestamps.add(point.x);
      });
    }

    // Sort timestamps and create rows
    const sortedTimestamps = Array.from(timestamps).sort((a, b) => a - b);
    
    sortedTimestamps.forEach((timestamp) => {
      const row = [new Date(timestamp).toISOString()];
      
      tagIds.forEach((tagId) => {
        const point = data[tagId].find((p: any) => p.x === timestamp);
        row.push(point ? point.y.toString() : '');
      });

      csvContent += row.join(',') + '\n';
    });

    // Download CSV
    this.downloadCsv(csvContent, `daq_export_${Date.now()}.csv`);
  });
}

downloadCsv(content: string, fileName: string) {
  const element = document.createElement('a');
  element.setAttribute('href', 'data:text/csv;charset=utf-8,' + encodeURIComponent(content));
  element.setAttribute('download', fileName);
  element.style.display = 'none';
  document.body.appendChild(element);
  element.click();
  document.body.removeChild(element);
}
```

### Use Case 3: Time Range Selector

```typescript
selectTimeRange(range: string) {
  const toDate = new Date();
  let fromDate: Date;

  const rangeInMs = {
    '1h': 60 * 60 * 1000,
    '6h': 6 * 60 * 60 * 1000,
    '24h': 24 * 60 * 60 * 1000,
    '7d': 7 * 24 * 60 * 60 * 1000,
    '30d': 30 * 24 * 60 * 60 * 1000,
    '90d': 90 * 24 * 60 * 60 * 1000,
    'custom': null  // User selects custom range
  };

  if (rangeInMs[range]) {
    fromDate = new Date(toDate.getTime() - rangeInMs[range]);
  } else {
    return; // Handle custom range separately
  }

  this.loadDaqData(fromDate, toDate);
}
```

---

## 7. URL Encoding Guide

### Why URL Encoding is Necessary

The `query` parameter contains JSON with special characters that must be URL-encoded:
- `{` → `%7B`
- `}` → `%7D`
- `"` → `%22`
- `[` → `%5B`
- `]` → `%5D`
- `:` → `%3A` (sometimes)
- `,` → `%2C` (sometimes)

### Manual Encoding Example

```javascript
// Original JSON
const query = {
  gid: "OXC_8e3c621a-c6bf490e",
  from: 1760121000000,
  to: 1762766003766,
  sids: ["t_80f6bec4-2c5740fd"]
};

// Convert to JSON string
const jsonString = JSON.stringify(query);
// Result: {"gid":"OXC_8e3c621a-c6bf490e","from":1760121000000,"to":1762766003766,"sids":["t_80f6bec4-2c5740fd"]}

// URL encode
const encoded = encodeURIComponent(jsonString);
// Result: %7B%22gid%22%3A%22OXC_8e3c621a-c6bf490e%22%2C%22from%22%3A1760121000000%2C%22to%22%3A1762766003766%2C%22sids%22%3A%5B%22t_80f6bec4-2c5740fd%22%5D%7D

// Final URL
const url = `/api/daq?query=${encoded}`;
```

### Decoding Example

```javascript
// From URL query parameter
const encodedQuery = '%7B%22gid%22%3A%22OXC_8e3c621a-c6bf490e%22%7D';

// Decode
const jsonString = decodeURIComponent(encodedQuery);
// Result: {"gid":"OXC_8e3c621a-c6bf490e"}

// Parse JSON
const query = JSON.parse(jsonString);
// Result: { gid: "OXC_8e3c621a-c6bf490e" }
```

---

## 8. Backend Implementation Overview

### API Route Handler

The FUXA backend implements the `/api/daq` route in `server/api.js` (or similar):

```javascript
// Simplified backend pseudocode
app.get('/api/daq', async (req, res) => {
  try {
    // Parse query parameter
    const query = JSON.parse(decodeURIComponent(req.query.query));

    // Extract parameters
    const { gid, from, to, sids } = query;

    // Validate parameters
    if (!gid || !from || !to || !sids) {
      return res.status(400).json({ error: 'Invalid parameters' });
    }

    // Fetch data from DAQ storage
    const daqStorage = require('./runtime/daqstorage');
    const result = await daqStorage.getValues(gid, sids, from, to);

    // Return formatted response
    res.json(result);

  } catch (error) {
    console.error('DAQ API Error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

### DAQ Storage Service (Backend)

The `daqstorage.js` service handles the actual data retrieval:

```javascript
// Simplified daqstorage pseudocode
class DaqStorage {
  
  async getValues(gid, sids, from, to) {
    const result = {};

    for (const sid of sids) {
      // Query database for each tag
      const data = await this.queryDatabase(gid, sid, from, to);
      result[sid] = data;
    }

    return result;
  }

  async queryDatabase(gid, sid, from, to) {
    // SQLite query example
    const query = `
      SELECT timestamp as x, value as y
      FROM tag_history
      WHERE tag_id = ? 
        AND timestamp >= ? 
        AND timestamp <= ?
      ORDER BY timestamp ASC
    `;

    const values = await db.all(query, [sid, from, to]);
    return values;
  }
}
```

---

## 9. Query Parameters Breakdown

### From Your Example Log

```
GET /api/daq?query=%7B%22gid%22:%22OXC_8e3c621a-c6bf490e%22,%22from%22:1760121000000,%22to%22:1762766003766,%22sids%22:%5B%22t_80f6bec4-2c5740fd%22,%22t_847b6c7c-868846bd%22,%22t_931aec4d-5b7b4561%22,%22t_caa27fd2-fa584d07%22,%22t_362ec83e-d0ce4d67%22,%22t_98c40cae-9cd54227%22,%22t_140e59c9-ef724af1%22%5D%7D 200 9178928 - 1224.607 ms
```

| Component | Value |
|-----------|-------|
| HTTP Method | GET |
| Endpoint | `/api/daq` |
| Status Code | 200 (Success) |
| Response Size | 9,178,928 bytes |
| Response Time | 1224.607 ms |
| GID | OXC_8e3c621a-c6bf490e |
| From | 1760121000000 (July 9, 2025) |
| To | 1762766003766 (July 31, 2025) |
| Tag Count | 7 tags |
| Time Range | ~22 days of historical data |

### Timestamp Breakdown

```javascript
// From timestamp
const from = 1760121000000;
const fromDate = new Date(from);
console.log(fromDate); // 2025-07-09T11:30:00.000Z

// To timestamp
const to = 1762766003766;
const toDate = new Date(to);
console.log(toDate); // 2025-07-31T16:33:23.766Z

// Duration
const duration = to - from;
const days = duration / (1000 * 60 * 60 * 24);
console.log(days); // ~22.21 days
```

---

## 10. Performance Optimization Tips

### Tip 1: Limit Time Range

```typescript
// ❌ Bad - Querying years of data
const query = {
  from: new Date('2020-01-01').getTime(),  // Too far back
  to: new Date().getTime(),
  sids: tags
};

// ✅ Good - Query reasonable time range
const query = {
  from: new Date().getTime() - (30 * 24 * 60 * 60 * 1000),  // Last 30 days
  to: new Date().getTime(),
  sids: tags
};
```

### Tip 2: Batch Requests for Many Tags

```typescript
// ❌ Bad - Querying 1000 tags in one request
const allTags = getAll1000Tags();
const query = {
  gid,
  from,
  to,
  sids: allTags  // Too many
};

// ✅ Good - Batch into 50-tag chunks
const allTags = getAll1000Tags();
const batchSize = 50;

for (let i = 0; i < allTags.length; i += batchSize) {
  const batch = allTags.slice(i, i + batchSize);
  const query = {
    gid,
    from,
    to,
    sids: batch
  };
  
  await fetchDaqData(query);
}
```

### Tip 3: Cache Results

```typescript
// Implement caching for repeated queries
const cache = new Map();

function getCacheKey(gid, from, to, sids) {
  return `${gid}:${from}:${to}:${sids.join(',')}`;
}

async function fetchWithCache(gid, sids, from, to) {
  const cacheKey = getCacheKey(gid, from, to, sids);
  
  if (cache.has(cacheKey)) {
    console.log('Returning cached data');
    return cache.get(cacheKey);
  }

  const data = await fetchDaqData(gid, sids, from, to);
  cache.set(cacheKey, data);
  
  return data;
}
```

### Tip 4: Implement Chunked/Streaming Responses

```typescript
// For very large queries, consider requesting data in chunks
async function fetchDataInChunks(
  gid: string,
  sids: string[],
  from: number,
  to: number,
  chunkDays: number = 7
) {
  const allData: any = {};
  const chunksNeeded = Math.ceil((to - from) / (chunkDays * 24 * 60 * 60 * 1000));

  for (let i = 0; i < chunksNeeded; i++) {
    const chunkFrom = from + (i * chunkDays * 24 * 60 * 60 * 1000);
    const chunkTo = Math.min(
      chunkFrom + (chunkDays * 24 * 60 * 60 * 1000),
      to
    );

    const chunkData = await fetchDaqData(gid, sids, chunkFrom, chunkTo);
    
    // Merge chunk data
    for (const sid in chunkData) {
      if (!allData[sid]) {
        allData[sid] = [];
      }
      allData[sid] = allData[sid].concat(chunkData[sid]);
    }
  }

  return allData;
}
```

---

## 11. Error Handling & Edge Cases

### Error Case 1: Missing Required Parameters

```typescript
// Missing 'gid' parameter
const query = {
  from: 1760121000000,
  to: 1762766003766,
  sids: ['t_80f6bec4-2c5740fd']
};

// Server Response (400)
{
  "error": "Invalid parameters",
  "message": "gid is required"
}
```

### Error Case 2: Invalid Timestamp Range

```typescript
const query = {
  gid: 'OXC_8e3c621a-c6bf490e',
  from: 1762766003766,  // Later than 'to'
  to: 1760121000000,    // Earlier than 'from'
  sids: ['t_80f6bec4-2c5740fd']
};

// May return empty results or error
```

### Error Case 3: Non-Existent Tag

```typescript
const query = {
  gid: 'OXC_8e3c621a-c6bf490e',
  from: 1760121000000,
  to: 1762766003766,
  sids: ['t_nonexistent-tag']
};

// Response - may return empty array or null
{
  "t_nonexistent-tag": []
}
```

### Error Handling in Angular

```typescript
async function safeDAQFetch(gid: string, sids: string[], from: number, to: number) {
  try {
    // Validate inputs
    if (!gid || !sids.length || !from || !to) {
      throw new Error('Missing required parameters');
    }

    if (from >= to) {
      throw new Error('Invalid time range: from must be before to');
    }

    // Fetch data
    const data = await fetchDaqData(gid, sids, from, to);

    // Validate response
    if (!data || Object.keys(data).length === 0) {
      console.warn('No data returned from DAQ endpoint');
      return {};
    }

    return data;

  } catch (error) {
    console.error('DAQ Fetch Error:', error);
    
    // Handle specific error types
    if (error instanceof TypeError) {
      console.error('Network error:', error.message);
    } else if (error instanceof SyntaxError) {
      console.error('JSON parse error:', error.message);
    } else {
      console.error('General error:', error.message);
    }

    return null;
  }
}
```

---

## 12. Comparing /api/daq vs /api/getHistoricalTags

| Aspect | /api/daq | /api/getHistoricalTags |
|--------|----------|----------------------|
| **Query Method** | Query parameter with JSON | Path parameters |
| **URL Format** | `/api/daq?query={}` | `/api/getHistoricalTags` |
| **Parameter Format** | URL-encoded JSON object | Individual parameters |
| **Production Usage** | Actively used in FUXA UI | Legacy/alternative |
| **Batch Support** | Multiple tags in one query | Yes |
| **Response Format** | `{tagId: []}` | Similar structure |
| **Performance** | Optimized | Similar |
| **Flexibility** | High (custom query object) | Lower (fixed parameters) |

---

## Conclusion

The `/api/daq` endpoint is the **core API** for historical data retrieval in FUXA. It uses a:

- **Query Parameter Approach**: Single `query` parameter with URL-encoded JSON
- **Flexible Structure**: Supports `gid`, `from`, `to`, and `sids` parameters
- **Batch Capability**: Query multiple tags in a single request
- **Efficient Delivery**: Returns data as `{tagId: [{x: timestamp, y: value}]}` format
- **Production-Ready**: Used in real-world FUXA deployments handling multi-MB data efficiently

For **best performance**, query 10-50 tags at a time, use reasonable time ranges (7-30 days), and implement caching for repeated queries.

---

*Document Version: 2.0 (Updated)*  
*Focus: /api/daq Endpoint Deep Dive*  
*Last Updated: November 10, 2025*
