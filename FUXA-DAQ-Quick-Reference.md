# FUXA /api/daq Endpoint - Quick Reference Guide

## Your Real-World Example

From your log entry:
```
GET /api/daq?query=%7B%22gid%22:%22OXC_8e3c621a-c6bf490e%22,%22from%22:1760121000000,%22to%22:1762766003766,%22sids%22:%5B%22t_80f6bec4-2c5740fd%22,%22t_847b6c7c-868846bd%22,%22t_931aec4d-5b7b4561%22,%22t_caa27fd2-fa584d07%22,%22t_362ec83e-d0ce4d67%22,%22t_98c40cae-9cd54227%22,%22t_140e59c9-ef724af1%22%5D%7D 
Response: 200 | Size: 9,178,928 bytes (~9.2 MB) | Time: 1224.607 ms
```

### Decoded Query Object
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

---

## Query Parameter Breakdown

### What Each Part Means

| Parameter | Your Value | Meaning |
|-----------|-----------|---------|
| **gid** | `OXC_8e3c621a-c6bf490e` | Project/Group ID - identifies which project's data to fetch |
| **from** | `1760121000000` | Start date: July 9, 2025 at 11:30 AM |
| **to** | `1762766003766` | End date: July 31, 2025 at 4:33 PM |
| **sids** | Array of 7 tags | Series IDs - the specific tags you want data for |
| **Time Range** | ~22 days | Duration of historical data query |

### JavaScript Example for Your Data

```javascript
const query = {
  gid: "OXC_8e3c621a-c6bf490e",
  from: 1760121000000,      // July 9, 2025
  to: 1762766003766,        // July 31, 2025
  sids: [
    "t_80f6bec4-2c5740fd",  // Tag 1
    "t_847b6c7c-868846bd",  // Tag 2
    "t_931aec4d-5b7b4561",  // Tag 3
    "t_caa27fd2-fa584d07",  // Tag 4
    "t_362ec83e-d0ce4d67",  // Tag 5
    "t_98c40cae-9cd54227",  // Tag 6
    "t_140e59c9-ef724af1"   // Tag 7
  ]
};

// Convert to URL-encoded query parameter
const encoded = encodeURIComponent(JSON.stringify(query));
const url = `/api/daq?query=${encoded}`;

// Fetch data
const response = await fetch(url);
const data = await response.json();
```

---

## How the API Works

### Request Flow

```
Client (Angular)
    ↓
Create query object: { gid, from, to, sids }
    ↓
Convert to JSON string: "{...}"
    ↓
URL-encode: "%7B%22gid%22..."
    ↓
Build URL: GET /api/daq?query=%7B...
    ↓
FUXA Backend
    ↓
Decode query parameter
    ↓
Query SQLite database:
    SELECT * FROM tag_history
    WHERE tag_id IN (sids)
    AND timestamp BETWEEN from AND to
    ↓
Return data as:
{
  "t_80f6bec4-2c5740fd": [
    { "x": 1760121000000, "y": 23.5 },
    { "x": 1760121060000, "y": 24.1 }
  ],
  "t_847b6c7c-868846bd": [
    { "x": 1760121000000, "y": 45.2 },
    { "x": 1760121060000, "y": 46.5 }
  ]
}
    ↓
Response to Client
    ↓
Render in Chart (Apache ECharts)
```

---

## Response Format Explanation

### Raw Response Structure
```json
{
  "t_80f6bec4-2c5740fd": [
    { "x": 1760121000000, "y": 23.5 },
    { "x": 1760121060000, "y": 24.1 },
    { "x": 1760121120000, "y": 23.8 }
  ],
  "t_847b6c7c-868846bd": [
    { "x": 1760121000000, "y": 45.2 },
    { "x": 1760121060000, "y": 46.5 }
  ]
}
```

### What Each Field Means
- **Key** (e.g., `t_80f6bec4-2c5740fd`): Tag ID from your `sids` array
- **x**: Timestamp in milliseconds (matches your `from` and `to` range)
- **y**: The actual value at that timestamp

### Converting Timestamps
```javascript
const timestamp = 1760121000000;
const date = new Date(timestamp);
console.log(date.toLocaleString());  // "7/9/2025, 11:30:00 AM"
```

---

## Performance Metrics from Your Example

| Metric | Value | Analysis |
|--------|-------|----------|
| **Response Size** | 9.2 MB | Large dataset (7 tags × 22 days) |
| **Response Time** | 1224.6 ms (~1.2 sec) | Very good performance |
| **Throughput** | ~7.5 MB/sec | Excellent backend efficiency |
| **Tags Queried** | 7 | Reasonable batch size |
| **Time Range** | 22 days | Typical for weekly/monthly reports |

---

## Using in Apache ECharts Bar Chart

### Simple Implementation

```typescript
// 1. Build your query
const query = {
  gid: "OXC_8e3c621a-c6bf490e",
  from: new Date(Date.now() - 24*60*60*1000).getTime(),  // Last 24 hours
  to: Date.now(),
  sids: ["t_80f6bec4-2c5740fd", "t_847b6c7c-868846bd"]
};

// 2. Fetch data
const encoded = encodeURIComponent(JSON.stringify(query));
const response = await fetch(`/api/daq?query=${encoded}`);
const data = await response.json();

// 3. Transform for ECharts
const series = [];
for (const tagId in data) {
  series.push({
    name: tagId,
    type: 'bar',
    data: data[tagId].map(p => [p.x, p.y])
  });
}

// 4. Configure chart
const option = {
  xAxis: { type: 'time' },
  yAxis: { type: 'value' },
  series: series,
  tooltip: { trigger: 'axis' },
  dataZoom: [{ type: 'slider' }]
};

// 5. Render
echarts.setOption(option);
```

---

## Common Use Cases

### Use Case 1: Display Last 24 Hours

```javascript
const now = Date.now();
const yesterday = now - (24 * 60 * 60 * 1000);

const query = {
  gid: "OXC_8e3c621a-c6bf490e",
  from: yesterday,
  to: now,
  sids: ["t_80f6bec4-2c5740fd"]
};
```

### Use Case 2: Display Last 7 Days

```javascript
const now = Date.now();
const sevenDaysAgo = now - (7 * 24 * 60 * 60 * 1000);

const query = {
  gid: "OXC_8e3c621a-c6bf490e",
  from: sevenDaysAgo,
  to: now,
  sids: ["t_80f6bec4-2c5740fd", "t_847b6c7c-868846bd"]
};
```

### Use Case 3: Query Specific Date Range

```javascript
const query = {
  gid: "OXC_8e3c621a-c6bf490e",
  from: new Date("2025-07-09").getTime(),
  to: new Date("2025-07-31").getTime(),
  sids: [
    "t_80f6bec4-2c5740fd",
    "t_847b6c7c-868846bd",
    "t_931aec4d-5b7b4561"
  ]
};
```

---

## Implementation Checklist

- [ ] Create `hmi.service.ts` with `getHistoricalData()` method
- [ ] Create bar chart component with ECharts
- [ ] Add time range selector (1h, 6h, 24h, 7d, 30d, custom)
- [ ] Implement query parameter encoding using `encodeURIComponent()`
- [ ] Handle API response and transform to chart format
- [ ] Add error handling for failed requests
- [ ] Add loading states
- [ ] Test with multiple tags (batch queries)
- [ ] Implement CSV export functionality
- [ ] Add chart refresh button
- [ ] Test performance with large time ranges
- [ ] Implement data caching for repeated queries

---

## Troubleshooting

### Issue: Getting empty results
**Solution:** Check that:
1. Tags exist and have DAQ recording enabled
2. Time range includes data (tags may not have recorded for that period)
3. `gid` parameter matches your project ID

### Issue: Slow response times for large ranges
**Solution:** 
1. Reduce time range
2. Query fewer tags at once (batch into 50-tag chunks)
3. Implement data aggregation (hourly/daily averages)
4. Use chunked loading (fetch 7 days at a time)

### Issue: URL too long error
**Solution:**
1. Reduce number of tags per query (50 max recommended)
2. Reduce time range
3. Use multiple sequential requests instead of one large query

---

## Key Takeaways

1. **Endpoint**: `/api/daq` uses query parameters
2. **Query Format**: Single parameter named `query` with URL-encoded JSON
3. **Required Fields**: `gid`, `from`, `to`, `sids`
4. **Response Format**: `{tagId: [{x: timestamp, y: value}]}`
5. **Performance**: Excellent for multi-MB queries (~7.5 MB/sec)
6. **Best Practice**: Batch 10-50 tags per request for optimal performance

---

## Related Documentation Files

This is Part 3 of FUXA DAQ documentation:
1. `FUXA-DAQ-Historical-Data-Documentation.md` - Complete DAQ system overview
2. `FUXA-DAQ-API-Endpoint.md` - Deep technical dive into /api/daq endpoint
3. `FUXA-DAQ-ECharts-Integration.md` - Production-ready Angular integration
4. **THIS FILE** - Quick reference guide

---

*Quick Reference Version: 1.0*  
*Focus: /api/daq Query Parameters & Examples*  
*Last Updated: November 10, 2025*
