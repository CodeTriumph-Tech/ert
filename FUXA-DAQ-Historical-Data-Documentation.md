# FUXA Historical Data (DAQ) Implementation - Complete Documentation

## Table of Contents
1. [Introduction to FUXA DAQ System](#introduction)
2. [Architecture Overview](#architecture-overview)
3. [Database Implementation](#database-implementation)
4. [DAQ Data Recording Logic](#daq-recording-logic)
5. [API Endpoints for Historical Data](#api-endpoints)
6. [Frontend Implementation](#frontend-implementation)
7. [Implementing Time Series Data in Apache Bar Charts](#apache-bar-charts)
8. [Complete Data Flow Diagram](#data-flow)
9. [Configuration Guide](#configuration-guide)
10. [Best Practices & Optimization](#best-practices)

---

## 1. Introduction to FUXA DAQ System {#introduction}

FUXA (an open-source SCADA/HMI/Dashboard software) includes a built-in **DAQ (Data Acquisition) system** that enables historical data storage, retrieval, and visualization. The DAQ system is a critical component that allows users to:

- Record tag values over time with configurable intervals
- Store historical data in a database (SQLite by default, or InfluxDB)
- Query historical data for specific time ranges
- Visualize historical trends using charts and graphs
- Export historical data to CSV format

**Key Features:**
- Automatic data recording based on tag configuration
- Change-based recording (only records when values change)
- Time-interval recording (records at fixed intervals)
- Data retention policies (configurable retention periods)
- Support for multiple database backends (SQLite, InfluxDB 2.0)

---

## 2. Architecture Overview {#architecture-overview}

FUXA follows a **Full-Stack Architecture** with:
- **Backend**: Node.js runtime
- **Frontend**: Angular framework
- **Database**: SQLite (default) or InfluxDB (optional)
- **Communication**: WebSocket for real-time data, REST API for historical queries

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     FUXA Application                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐         ┌──────────────┐                 │
│  │   Frontend   │◄───────►│   Backend    │                 │
│  │  (Angular)   │         │  (Node.js)   │                 │
│  └──────────────┘         └──────┬───────┘                 │
│        │                          │                          │
│        │                          │                          │
│        ▼                          ▼                          │
│  ┌──────────────┐         ┌──────────────┐                 │
│  │   Charts &   │         │    Runtime   │                 │
│  │ Visualization│         │   Services   │                 │
│  └──────────────┘         └──────┬───────┘                 │
│                                   │                          │
│                                   ▼                          │
│                           ┌──────────────┐                  │
│                           │  DAQ Storage │                  │
│                           │   Service    │                  │
│                           └──────┬───────┘                  │
│                                  │                           │
└──────────────────────────────────┼───────────────────────────┘
                                   │
                                   ▼
                          ┌─────────────────┐
                          │    Database     │
                          │  (SQLite/       │
                          │   InfluxDB)     │
                          └─────────────────┘
                                   ▲
                                   │
                          ┌────────┴─────────┐
                          │  Physical Files  │
                          │  server/_db/     │
                          │  *.db files      │
                          └──────────────────┘
```

### Component Breakdown

**1. Frontend Layer (Angular)**
- Chart components (Line, Bar, Pie, Gauge charts)
- Historical data controls (date range picker, time selectors)
- Real-time vs Historical mode toggle
- Data export functionality

**2. Backend Layer (Node.js)**
- Runtime service: Manages device connections and tag polling
- DAQ Storage service: Handles historical data recording
- API routes: Provides endpoints for data queries
- Device drivers: Modbus, OPC-UA, S7, MQTT, BACnet, etc.

**3. Database Layer**
- **SQLite** (Default): Lightweight, file-based database stored in `server/_db/` directory
- **InfluxDB 2.0** (Optional): Time-series database for high-performance scenarios
- Automatic database rotation based on retention policies

---

## 3. Database Implementation {#database-implementation}

### 3.1 SQLite Database Structure

FUXA uses **SQLite** as its default database backend for historical data storage. The database files are stored in the `server/_db/` directory.

**Database File Naming Convention:**
```
fuxa_<timestamp>.db
```

Example: `fuxa_20251110.db`

### 3.2 Database Schema

The primary table for storing historical tag data is typically structured as follows:

**Table: `tag_history` or `daq_values`**

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| `id` | INTEGER | Primary key (auto-increment) |
| `tag_id` | TEXT | Unique identifier of the tag (e.g., "t_5588a151-0dbf4300") |
| `tag_name` | TEXT | Human-readable tag name |
| `timestamp` | INTEGER/TEXT | Unix timestamp or ISO date string |
| `value` | REAL/TEXT | Recorded value of the tag |
| `quality` | INTEGER | Data quality indicator (0=bad, 192=good) |

**Example SQL Schema:**
```sql
CREATE TABLE IF NOT EXISTS tag_history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    tag_id TEXT NOT NULL,
    tag_name TEXT NOT NULL,
    timestamp INTEGER NOT NULL,
    value REAL,
    quality INTEGER DEFAULT 192,
    INDEX idx_tag_timestamp (tag_id, timestamp)
);
```

**Indexes:**
- Composite index on `(tag_id, timestamp)` for fast range queries
- Individual index on `timestamp` for time-based filtering

### 3.3 Data Retention & Archiving

FUXA implements automatic data retention policies:

**Retention Options:**
- 1 day
- 1 week (7 days)
- 1 month (30 days)
- 3 months (90 days)
- 6 months (180 days)
- 1 year (365 days)
- 3 years
- 5 years

**Archiving Mechanism:**
When the retention period expires, FUXA:
1. Creates a new database file with current timestamp
2. Moves the old database file to `server/_db/archive/` directory
3. Continues recording to the new database file

**File Structure:**
```
server/
├── _db/
│   ├── fuxa_current.db        (Active database)
│   └── archive/
│       ├── fuxa_20251101.db   (Archived)
│       ├── fuxa_20251015.db   (Archived)
│       └── fuxa_20250920.db   (Archived)
```

### 3.4 InfluxDB Integration

For high-performance scenarios with thousands of tags, FUXA supports **InfluxDB 2.0**.

**Configuration:**
- Navigate to Settings → DAQ Storage
- Select "InfluxDB 2.0" as storage type
- Configure connection parameters:
  - URL: `http://localhost:8086`
  - Organization name
  - Bucket name
  - API Token

**Data Structure in InfluxDB:**
```
Measurement: tag_values
Tags: tag_id, tag_name, device_name
Fields: value, quality
Timestamp: Nanosecond precision
```

---

## 4. DAQ Data Recording Logic {#daq-recording-logic}

### 4.1 Tag Configuration for DAQ

To enable historical data recording for a tag, you must configure its DAQ properties:

**Steps to Enable DAQ Recording:**

1. Navigate to **Connections** in the FUXA editor
2. Select your device and tag
3. Open **Tag Options**
4. Enable **"Registration"** or **"Storage"**
5. Configure recording parameters:
   - **Interval**: Recording frequency in seconds (0 = only on change)
   - **Deadband**: Minimum change threshold (for analog values)
   - **Enable/Disable**: Toggle recording on/off

**Tag Configuration Example:**
```json
{
  "id": "t_5588a151-0dbf4300",
  "name": "Temperature_Sensor_01",
  "type": "Float32",
  "address": "40001",
  "device": "Modbus_Device_01",
  "daq": {
    "enabled": true,
    "interval": 5,        // Record every 5 seconds
    "deadband": 0.5,      // Only record if change > 0.5
    "onchange": true      // Record on value change
  }
}
```

### 4.2 Recording Strategies

FUXA supports multiple recording strategies:

#### Strategy 1: **Time-Based Recording**
Records tag values at fixed time intervals.

```javascript
// Example: Record every 10 seconds
{
  "interval": 10,
  "onchange": false
}
```

**Use Case:** Regular monitoring of stable processes

#### Strategy 2: **Change-Based Recording**
Records only when the tag value changes beyond the deadband threshold.

```javascript
// Example: Record only when change > 1.0
{
  "interval": 0,        // 0 means on-change only
  "onchange": true,
  "deadband": 1.0
}
```

**Use Case:** Efficient storage for slowly changing values

#### Strategy 3: **Hybrid Recording**
Combines time-based and change-based recording.

```javascript
// Example: Record on change, but at least every 60 seconds
{
  "interval": 60,       // Maximum 60 seconds between records
  "onchange": true,
  "deadband": 0.5
}
```

**Use Case:** Balance between data completeness and storage efficiency

### 4.3 Runtime DAQ Service Flow

The DAQ service runs continuously in the FUXA backend:

**Execution Flow:**
```
1. Runtime starts
   ↓
2. Load tag configurations from project
   ↓
3. Initialize DAQ Storage Service
   ↓
4. For each tag with DAQ enabled:
   ├─ Start polling device (based on device scan rate)
   ├─ Check if value changed (> deadband)
   ├─ Check if interval elapsed
   └─ If condition met → Record to database
   ↓
5. Repeat step 4 continuously
   ↓
6. On retention period expiration:
   ├─ Archive current database
   └─ Create new database file
```

**Pseudo-code for DAQ Recording Logic:**
```javascript
// Simplified DAQ recording logic
function recordTagValue(tag, newValue, timestamp) {
  // Check if DAQ is enabled for this tag
  if (!tag.daq || !tag.daq.enabled) {
    return;
  }
  
  // Get last recorded value and timestamp
  const lastRecord = getLastRecordedValue(tag.id);
  const timeSinceLastRecord = timestamp - lastRecord.timestamp;
  
  // Check recording conditions
  let shouldRecord = false;
  
  // Condition 1: Time interval elapsed
  if (tag.daq.interval > 0 && timeSinceLastRecord >= tag.daq.interval * 1000) {
    shouldRecord = true;
  }
  
  // Condition 2: Value changed beyond deadband
  if (tag.daq.onchange && Math.abs(newValue - lastRecord.value) > tag.daq.deadband) {
    shouldRecord = true;
  }
  
  // Record to database if conditions met
  if (shouldRecord) {
    const record = {
      tag_id: tag.id,
      tag_name: tag.name,
      timestamp: timestamp,
      value: newValue,
      quality: tag.quality || 192
    };
    
    daqStorage.insert(record);
  }
}
```

### 4.4 System Functions for DAQ

FUXA provides system functions accessible in scripts:

**$getTagDaqSettings(tagId)**
- Returns DAQ configuration for a specific tag
- Usage: `let config = $getTagDaqSettings('t_5588a151-0dbf4300');`

**$setTagDaqSettings(tagId, settings)**
- Modifies DAQ configuration at runtime
- Usage: `$setTagDaqSettings('t_5588a151-0dbf4300', { interval: 10, enabled: true });`

**$getHistoricalTags(tagIds, startTime, endTime)**
- Retrieves historical data for specified tags and time range
- Usage example in next section

---

## 5. API Endpoints for Historical Data {#api-endpoints}

FUXA exposes REST API endpoints for querying historical data.

### 5.1 Get Historical Tag Values

**Endpoint:** `GET /api/getHistoricalTags`

**Query Parameters:**
```json
{
  "tagIds": ["t_5588a151-0dbf4300", "t_080f5955-0fd54761"],
  "startTime": 1699574400000,  // Unix timestamp (ms)
  "endTime": 1699660800000      // Unix timestamp (ms)
}
```

**Response Format:**
```json
{
  "status": "success",
  "data": {
    "t_5588a151-0dbf4300": [
      {
        "timestamp": 1699574400000,
        "value": 23.5,
        "quality": 192
      },
      {
        "timestamp": 1699574460000,
        "value": 24.1,
        "quality": 192
      }
    ],
    "t_080f5955-0fd54761": [
      {
        "timestamp": 1699574400000,
        "value": 75.2,
        "quality": 192
      }
    ]
  }
}
```

### 5.2 Get Tag Value (Current)

**Endpoint:** `GET /api/getTagValue`

**Query Parameters:**
```
tagId: t_5588a151-0dbf4300
```

**Response:**
```json
{
  "id": "t_5588a151-0dbf4300",
  "name": "Temperature_Sensor_01",
  "value": 25.3,
  "timestamp": 1699660800000,
  "quality": 192
}
```

### 5.3 Set Tag Value

**Endpoint:** `POST /api/setTagValue`

**Request Body:**
```json
{
  "tagId": "t_5588a151-0dbf4300",
  "value": 30.0
}
```

**Response:**
```json
{
  "status": "success",
  "message": "Tag value updated"
}
```

### 5.4 Export Historical Data

**Endpoint:** `GET /api/exportHistoricalData`

**Query Parameters:**
```json
{
  "tagIds": ["t_5588a151-0dbf4300"],
  "startTime": 1699574400000,
  "endTime": 1699660800000,
  "format": "csv"
}
```

**Response:** CSV file download

```csv
timestamp,tag_name,value,quality
2023-11-10T00:00:00Z,Temperature_Sensor_01,23.5,192
2023-11-10T00:01:00Z,Temperature_Sensor_01,24.1,192
```

### 5.5 API Usage in Client Scripts

**Example: Fetching Historical Data in a FUXA Script**

```javascript
// Client-side script to fetch historical data
async function loadHistoricalData() {
  // Define time range (last 24 hours)
  const endTime = Date.now();
  const startTime = endTime - (24 * 60 * 60 * 1000);
  
  // Define tag IDs to query
  const tagIds = ['t_5588a151-0dbf4300', 't_080f5955-0fd54761'];
  
  try {
    // Use $getHistoricalTags system function
    const historicalData = await $getHistoricalTags(tagIds, startTime, endTime);
    
    // Process data for chart
    return processDataForChart(historicalData);
  } catch (error) {
    console.error('Error fetching historical data:', error);
    return null;
  }
}

function processDataForChart(data) {
  const chartData = [];
  
  for (const tagId in data) {
    const tagData = data[tagId];
    const series = {
      name: tagData[0]?.tag_name || tagId,
      data: tagData.map(point => ({
        x: point.timestamp,
        y: point.value
      }))
    };
    chartData.push(series);
  }
  
  return chartData;
}
```

---

## 6. Frontend Implementation {#frontend-implementation}

### 6.1 Chart Configuration for Historical Data

In FUXA editor, charts can display historical data:

**Chart Type: Line Chart (Time Series)**

**Configuration Steps:**
1. Drag a **Chart** component onto the view
2. Open **Chart Properties**
3. Select or create a chart configuration
4. Set **Data Type** to "History"
5. Configure time range:
   - Last 5 minutes
   - Last hour
   - Last 24 hours
   - Custom date range
6. Select tags to display
7. Configure display options (legend, tooltip, axis labels)

**Chart Configuration JSON:**
```json
{
  "name": "Temperature_History_Chart",
  "type": "line",
  "dataType": "history",
  "timeRange": {
    "type": "relative",
    "value": "24h"
  },
  "series": [
    {
      "tagId": "t_5588a151-0dbf4300",
      "tagName": "Temperature_Sensor_01",
      "color": "#FF6384",
      "interpolation": "linear"
    },
    {
      "tagId": "t_080f5955-0fd54761",
      "tagName": "Pressure_Sensor_01",
      "color": "#36A2EB",
      "interpolation": "step-after"
    }
  ],
  "options": {
    "showLegend": true,
    "showTooltip": true,
    "dateFormat": "YYYY-MM-DD HH:mm:ss",
    "yAxisLabel": "Temperature (°C)",
    "xAxisLabel": "Time"
  }
}
```

### 6.2 History Table Component

FUXA also provides a **History Table** component for tabular display:

**Configuration:**
1. Drag **History Table** component
2. Select tags to display
3. Enable **Date Range** selector
4. Enable **Export to CSV** button

**Table Configuration:**
```json
{
  "type": "history-table",
  "tags": [
    "t_5588a151-0dbf4300",
    "t_080f5955-0fd54761"
  ],
  "columns": ["timestamp", "tag_name", "value", "quality"],
  "dateRangeEnabled": true,
  "exportEnabled": true,
  "pageSize": 50
}
```

### 6.3 Real-Time vs Historical Toggle

Charts can switch between real-time and historical modes:

**Implementation:**
```typescript
// Angular component for chart
export class ChartComponent implements OnInit {
  displayMode: 'realtime' | 'history' = 'realtime';
  
  toggleMode(mode: 'realtime' | 'history') {
    this.displayMode = mode;
    
    if (mode === 'history') {
      this.loadHistoricalData();
    } else {
      this.subscribeToRealTimeData();
    }
  }
  
  async loadHistoricalData() {
    const startTime = this.getStartTime();
    const endTime = Date.now();
    
    const data = await this.hmiService.getHistoricalTags(
      this.tagIds,
      startTime,
      endTime
    );
    
    this.updateChart(data);
  }
}
```

---

## 7. Implementing Time Series Data in Apache Bar Charts {#apache-bar-charts}

Based on your previous work integrating Apache ECharts with FUXA, here's how to implement historical data in bar charts.

### 7.1 Component Structure

**File: `gauges-apache-bar-property.component.ts`**

```typescript
import { Component, OnInit, Input } from '@angular/core';
import { EChartsOption } from 'echarts';
import { HmiService } from '../../_services/hmi.service';

@Component({
  selector: 'app-gauges-apache-bar-property',
  templateUrl: './gauges-apache-bar-property.component.html',
  styleUrls: ['./gauges-apache-bar-property.component.scss']
})
export class GaugesApacheBarPropertyComponent implements OnInit {
  @Input() data: any;
  @Input() property: any;
  
  chartOption: EChartsOption;
  historicalMode: boolean = false;
  
  constructor(private hmiService: HmiService) {}
  
  ngOnInit() {
    this.initializeChart();
    
    if (this.property.dataMode === 'history') {
      this.loadHistoricalData();
    } else {
      this.subscribeToRealTimeData();
    }
  }
  
  initializeChart() {
    this.chartOption = {
      title: {
        text: this.property.title || 'Bar Chart',
        left: 'center'
      },
      tooltip: {
        trigger: 'axis',
        axisPointer: {
          type: 'shadow'
        },
        formatter: (params: any) => {
          const param = params[0];
          const time = new Date(param.axisValue).toLocaleString();
          return `${time}<br/>${param.seriesName}: ${param.value}`;
        }
      },
      xAxis: {
        type: 'time',
        name: 'Time',
        axisLabel: {
          formatter: (value: number) => {
            const date = new Date(value);
            return date.toLocaleTimeString();
          }
        }
      },
      yAxis: {
        type: 'value',
        name: this.property.yAxisLabel || 'Value'
      },
      series: []
    };
  }
  
  async loadHistoricalData() {
    try {
      // Get time range from property configuration
      const endTime = Date.now();
      const startTime = this.getStartTime(this.property.timeRange);
      
      // Fetch historical data for all configured tags
      const tagIds = this.property.tags.map(t => t.id);
      const historicalData = await this.hmiService.getHistoricalTags(
        tagIds,
        startTime,
        endTime
      );
      
      // Transform data for bar chart
      const series = this.transformDataForBarChart(historicalData);
      
      // Update chart
      this.chartOption = {
        ...this.chartOption,
        series: series
      };
      
    } catch (error) {
      console.error('Error loading historical data:', error);
    }
  }
  
  transformDataForBarChart(historicalData: any): any[] {
    const series = [];
    
    for (const tag of this.property.tags) {
      const tagData = historicalData[tag.id] || [];
      
      // Transform to [timestamp, value] format for time-series
      const data = tagData.map(point => [
        point.timestamp,
        point.value
      ]);
      
      series.push({
        name: tag.name,
        type: 'bar',
        data: data,
        barWidth: this.property.barWidth || '60%',
        itemStyle: {
          color: tag.color || '#5470c6'
        }
      });
    }
    
    return series;
  }
  
  getStartTime(timeRange: string): number {
    const now = Date.now();
    const ranges = {
      '1h': 60 * 60 * 1000,
      '6h': 6 * 60 * 60 * 1000,
      '12h': 12 * 60 * 60 * 1000,
      '24h': 24 * 60 * 60 * 1000,
      '7d': 7 * 24 * 60 * 60 * 1000,
      '30d': 30 * 24 * 60 * 60 * 1000
    };
    
    return now - (ranges[timeRange] || ranges['24h']);
  }
  
  subscribeToRealTimeData() {
    // Subscribe to real-time tag updates via WebSocket
    for (const tag of this.property.tags) {
      this.hmiService.onTagChanged(tag.id).subscribe(
        (value: any) => {
          this.updateRealTimeChart(tag.id, value);
        }
      );
    }
  }
  
  updateRealTimeChart(tagId: string, value: any) {
    // Find the series for this tag
    const seriesIndex = this.property.tags.findIndex(t => t.id === tagId);
    
    if (seriesIndex >= 0) {
      const series = this.chartOption.series[seriesIndex];
      const timestamp = Date.now();
      
      // Add new data point
      series.data.push([timestamp, value.value]);
      
      // Keep only last N points for real-time display
      const maxPoints = this.property.maxRealtimePoints || 50;
      if (series.data.length > maxPoints) {
        series.data.shift();
      }
      
      // Update chart
      this.chartOption = { ...this.chartOption };
    }
  }
}
```

### 7.2 Template File

**File: `gauges-apache-bar-property.component.html`**

```html
<div class="chart-container">
  <!-- Mode Selector -->
  <div class="chart-controls" *ngIf="property.showModeSelector">
    <button 
      (click)="toggleMode('realtime')" 
      [class.active]="!historicalMode">
      Real-Time
    </button>
    <button 
      (click)="toggleMode('history')" 
      [class.active]="historicalMode">
      Historical
    </button>
  </div>
  
  <!-- Date Range Selector (for historical mode) -->
  <div class="date-range-selector" *ngIf="historicalMode">
    <label>Time Range:</label>
    <select [(ngModel)]="property.timeRange" (change)="onTimeRangeChange()">
      <option value="1h">Last Hour</option>
      <option value="6h">Last 6 Hours</option>
      <option value="12h">Last 12 Hours</option>
      <option value="24h">Last 24 Hours</option>
      <option value="7d">Last 7 Days</option>
      <option value="30d">Last 30 Days</option>
      <option value="custom">Custom Range</option>
    </select>
    
    <!-- Custom date pickers -->
    <div *ngIf="property.timeRange === 'custom'" class="custom-range">
      <input 
        type="datetime-local" 
        [(ngModel)]="customStartDate"
        (change)="onCustomRangeChange()">
      <span>to</span>
      <input 
        type="datetime-local" 
        [(ngModel)]="customEndDate"
        (change)="onCustomRangeChange()">
    </div>
  </div>
  
  <!-- ECharts Component -->
  <div echarts 
       [options]="chartOption" 
       class="echarts-instance"
       [style.height]="property.height || '400px'">
  </div>
</div>
```

### 7.3 HMI Service Method for Historical Data

**File: `hmi.service.ts`**

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class HmiService {
  private apiUrl = '/api';
  
  constructor(private http: HttpClient) {}
  
  /**
   * Fetch historical data for specified tags and time range
   */
  async getHistoricalTags(
    tagIds: string[], 
    startTime: number, 
    endTime: number
  ): Promise<any> {
    const params = {
      tagIds: tagIds.join(','),
      startTime: startTime.toString(),
      endTime: endTime.toString()
    };
    
    try {
      const response = await this.http.get(
        `${this.apiUrl}/getHistoricalTags`,
        { params }
      ).toPromise();
      
      return response;
    } catch (error) {
      console.error('Error fetching historical tags:', error);
      throw error;
    }
  }
  
  /**
   * Subscribe to real-time tag changes via WebSocket
   */
  onTagChanged(tagId: string): Observable<any> {
    // Implementation depends on your WebSocket setup
    // This is a simplified example
    return new Observable(observer => {
      this.webSocket.on('tag-changed', (data) => {
        if (data.tagId === tagId) {
          observer.next(data);
        }
      });
    });
  }
}
```

### 7.4 Data Aggregation for Large Time Ranges

When querying large time ranges, aggregate data to reduce number of points:

```typescript
aggregateHistoricalData(data: any[], intervalMinutes: number): any[] {
  const intervalMs = intervalMinutes * 60 * 1000;
  const aggregated = [];
  
  // Group data by time intervals
  const groups = new Map();
  
  for (const point of data) {
    const intervalKey = Math.floor(point.timestamp / intervalMs) * intervalMs;
    
    if (!groups.has(intervalKey)) {
      groups.set(intervalKey, []);
    }
    
    groups.get(intervalKey).push(point.value);
  }
  
  // Calculate average for each interval
  for (const [timestamp, values] of groups.entries()) {
    const avg = values.reduce((a, b) => a + b, 0) / values.length;
    aggregated.push({
      timestamp: timestamp,
      value: avg
    });
  }
  
  return aggregated.sort((a, b) => a.timestamp - b.timestamp);
}
```

### 7.5 Complete Example: Horizontal Bar Chart with Historical Data

```typescript
createHorizontalBarChartOption(): EChartsOption {
  return {
    title: {
      text: 'Production Data - Last 24 Hours',
      left: 'center'
    },
    tooltip: {
      trigger: 'axis',
      axisPointer: {
        type: 'shadow'
      }
    },
    legend: {
      top: 30,
      data: ['Line 1', 'Line 2', 'Line 3']
    },
    grid: {
      left: '3%',
      right: '4%',
      bottom: '3%',
      containLabel: true
    },
    xAxis: {
      type: 'value',
      name: 'Units Produced',
      boundaryGap: [0, 0.01]
    },
    yAxis: {
      type: 'time',
      name: 'Time',
      axisLabel: {
        formatter: (value: number) => {
          const date = new Date(value);
          return date.toLocaleTimeString();
        }
      }
    },
    series: [
      {
        name: 'Line 1',
        type: 'bar',
        data: [], // Will be populated with historical data
        itemStyle: {
          color: '#5470c6'
        }
      },
      {
        name: 'Line 2',
        type: 'bar',
        data: [],
        itemStyle: {
          color: '#91cc75'
        }
      },
      {
        name: 'Line 3',
        type: 'bar',
        data: [],
        itemStyle: {
          color: '#fac858'
        }
      }
    ]
  };
}
```

---

## 8. Complete Data Flow Diagram {#data-flow}

### 8.1 Historical Data Recording Flow

```
Device/PLC
    │
    ├─ Tag Value Changes: 25.3°C
    │
    ▼
Device Driver (Modbus/OPC-UA/S7)
    │
    ├─ Polls tag every scan interval (e.g., 1 second)
    │
    ▼
Runtime Service
    │
    ├─ Receives new tag value
    ├─ Checks DAQ configuration for tag
    │
    ▼
DAQ Recording Logic
    │
    ├─ Is DAQ enabled? ───► No ──► Skip
    │                              │
    ▼ Yes                          │
    │                              │
    ├─ Has value changed > deadband? ─► Yes ──► Record
    │                              │
    ▼ No                           │
    │                              │
    ├─ Has interval elapsed? ──────► Yes ──► Record
    │                              │
    ▼ No                           │
    │                              │
    └─ Skip ◄──────────────────────┘
    │
    ▼
DAQ Storage Service
    │
    ├─ Prepare record: { tag_id, timestamp, value, quality }
    │
    ▼
Database (SQLite/InfluxDB)
    │
    ├─ INSERT INTO tag_history VALUES (...)
    │
    ▼
Historical Data Stored
```

### 8.2 Historical Data Retrieval Flow

```
User Interface (Angular)
    │
    ├─ User selects "Historical" mode
    ├─ User selects time range: "Last 24 Hours"
    ├─ User selects tags to display
    │
    ▼
Chart Component
    │
    ├─ Calculate startTime and endTime
    ├─ Call: hmiService.getHistoricalTags(tagIds, start, end)
    │
    ▼
HMI Service (Frontend)
    │
    ├─ HTTP GET /api/getHistoricalTags
    │   Params: { tagIds: [...], startTime: xxx, endTime: xxx }
    │
    ▼
API Routes (Backend)
    │
    ├─ Parse request parameters
    ├─ Validate tag IDs and time range
    │
    ▼
DAQ Storage Service (Backend)
    │
    ├─ Build SQL query:
    │   SELECT * FROM tag_history
    │   WHERE tag_id IN (...)
    │   AND timestamp BETWEEN start AND end
    │   ORDER BY timestamp ASC
    │
    ▼
Database (SQLite/InfluxDB)
    │
    ├─ Execute query
    ├─ Return result set
    │
    ▼
API Routes (Backend)
    │
    ├─ Format response:
    │   {
    │     "t_xxx": [{ timestamp, value, quality }, ...],
    │     "t_yyy": [{ timestamp, value, quality }, ...]
    │   }
    │
    ▼
HMI Service (Frontend)
    │
    ├─ Receive response
    ├─ Return Promise with data
    │
    ▼
Chart Component
    │
    ├─ Transform data for chart format
    ├─ Update chartOption with series data
    │
    ▼
ECharts Rendering Engine
    │
    ├─ Render chart with historical data
    │
    ▼
Display to User
```

### 8.3 Real-Time to Historical Mode Switch

```
Chart Component (Real-Time Mode)
    │
    ├─ Receiving WebSocket updates
    ├─ Displaying last 50 data points
    │
    ▼
User clicks "Switch to Historical"
    │
    ▼
Chart Component
    │
    ├─ Stop WebSocket subscription
    ├─ Show date range selector
    ├─ User selects "Last 7 Days"
    │
    ▼
Load Historical Data
    │
    ├─ Calculate startTime = now - 7 days
    ├─ Calculate endTime = now
    ├─ Fetch data from API
    │
    ▼
Apply Data Aggregation (if needed)
    │
    ├─ If total points > 1000:
    │   ├─ Calculate interval = (endTime - startTime) / 1000
    │   └─ Aggregate data by interval
    │
    ▼
Update Chart
    │
    ├─ Clear existing series
    ├─ Populate with historical data
    ├─ Adjust xAxis range
    │
    ▼
Display Historical Chart
```

---

## 9. Configuration Guide {#configuration-guide}

### 9.1 Enabling DAQ for Your Project

**Step 1: Enable DAQ Storage**

1. Open FUXA Editor
2. Navigate to **Edit Project** → **Settings**
3. Go to **DAQ Storage** section
4. Select storage type:
   - SQLite (default)
   - InfluxDB 2.0
5. Configure retention period (e.g., 30 days)
6. Save project

**Step 2: Configure Tags for Recording**

1. Navigate to **Connections**
2. Select your device
3. For each tag you want to record:
   - Click on tag name
   - Open **Tag Options**
   - Enable **"Registration"**
   - Set **Interval** (seconds):
     - 0 = record only on change
     - >0 = record at fixed interval
   - Set **Deadband** (for analog values)
   - Save changes

**Step 3: Verify Recording**

1. Check `server/_db/` directory for database file
2. Use SQLite browser to view data:
   ```bash
   sqlite3 server/_db/fuxa_current.db
   SELECT * FROM tag_history LIMIT 10;
   ```

### 9.2 Configuring Charts for Historical Display

**Chart Configuration:**

1. Add **Chart** component to view
2. Open **Chart Properties**
3. Click **"Edit Chart"** or create new
4. Configure chart:
   - **Name**: "Temperature Trend"
   - **Type**: Line / Bar / Area
   - Add **Lines** (series):
     - Select tag from dropdown
     - Choose color
     - Select interpolation (linear, step-before, step-after)
5. In chart component properties:
   - **Data Type**: History
   - **Time Range**: Last 24 hours (or custom)
   - **Show Legend**: Yes
   - **Enable Date Range**: Yes

### 9.3 Performance Tuning

**For High Tag Count (>1000 tags):**

1. Use InfluxDB instead of SQLite
2. Increase DAQ interval (reduce recording frequency)
3. Use change-based recording (interval = 0)
4. Implement data aggregation for long time ranges
5. Disable "Broadcast to Client" option in settings

**Database Optimization:**

```sql
-- Create indexes for faster queries
CREATE INDEX IF NOT EXISTS idx_tag_id 
ON tag_history(tag_id);

CREATE INDEX IF NOT EXISTS idx_timestamp 
ON tag_history(timestamp);

CREATE INDEX IF NOT EXISTS idx_tag_time 
ON tag_history(tag_id, timestamp);

-- Vacuum database periodically
VACUUM;
```

---

## 10. Best Practices & Optimization {#best-practices}

### 10.1 Recording Strategy Recommendations

**Slow-Changing Values (Temperature, Pressure):**
```json
{
  "interval": 60,      // Record at least every minute
  "onchange": true,    // Also record on significant change
  "deadband": 0.5      // 0.5 unit change threshold
}
```

**Fast-Changing Values (Motor Speed, Flow Rate):**
```json
{
  "interval": 5,       // Record every 5 seconds
  "onchange": true,
  "deadband": 2.0      // Larger deadband to reduce noise
}
```

**Binary/Digital Values (Switches, Status):**
```json
{
  "interval": 0,       // Record only on change
  "onchange": true,
  "deadband": 0        // No deadband for digital values
}
```

### 10.2 Storage Optimization

**Estimate Storage Requirements:**

```
Storage per tag per day = 
  (86400 seconds / interval) × record_size
  
Example:
- 1 tag
- 10 second interval
- Record size ≈ 50 bytes

Storage = (86400 / 10) × 50 = 432 KB/day
For 1000 tags = 432 MB/day
For 30 days = 13 GB
```

**Optimization Strategies:**

1. **Increase Intervals**: Use longer recording intervals where acceptable
2. **Larger Deadbands**: Reduce noise recording with appropriate deadband values
3. **Retention Policies**: Archive or delete old data based on business requirements
4. **Database Compression**: Enable SQLite compression or use InfluxDB compression
5. **Selective Recording**: Only record critical tags, not all tags

### 10.3 Query Performance

**Best Practices:**

1. **Limit Time Range**: Query smaller time ranges when possible
2. **Use Aggregation**: For long time ranges, aggregate data (hourly/daily averages)
3. **Index Usage**: Ensure proper indexes exist on tag_id and timestamp
4. **Batch Queries**: Fetch multiple tags in single query instead of separate queries
5. **Caching**: Implement caching for frequently accessed historical data

**Aggregation Example:**

```sql
-- Hourly averages for last 7 days
SELECT 
  tag_id,
  strftime('%Y-%m-%d %H:00:00', datetime(timestamp/1000, 'unixepoch')) as hour,
  AVG(value) as avg_value,
  MIN(value) as min_value,
  MAX(value) as max_value,
  COUNT(*) as sample_count
FROM tag_history
WHERE tag_id = 't_5588a151-0dbf4300'
  AND timestamp > (strftime('%s', 'now', '-7 days') * 1000)
GROUP BY tag_id, hour
ORDER BY hour;
```

### 10.4 Monitoring & Maintenance

**Regular Maintenance Tasks:**

1. **Monitor Database Size**: Check `server/_db/` directory size
2. **Verify Archiving**: Ensure old databases are properly archived
3. **Check Recording**: Verify tags are recording as expected
4. **Performance Monitoring**: Monitor query response times
5. **Backup**: Regular backups of historical database files

**Health Check Script:**

```javascript
// Server-side script to check DAQ health
async function checkDAQHealth() {
  const daqService = $getService('daqStorage');
  
  // Check database size
  const dbSize = await daqService.getDatabaseSize();
  console.log(`Database size: ${(dbSize / 1024 / 1024).toFixed(2)} MB`);
  
  // Check last recorded timestamp
  const lastRecord = await daqService.getLastRecord();
  const timeSinceLastRecord = Date.now() - lastRecord.timestamp;
  
  if (timeSinceLastRecord > 60000) { // More than 1 minute
    console.warn('No data recorded in last minute!');
  }
  
  // Check number of active tags
  const activeTags = await daqService.getActiveTagCount();
  console.log(`Active DAQ tags: ${activeTags}`);
}

// Run every hour
$scheduler.every('1h', checkDAQHealth);
```

### 10.5 Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| Historical chart shows no data | DAQ not enabled for tag | Enable "Registration" in tag options |
| Database file grows too quickly | Too many tags or short intervals | Increase intervals, use deadbands, reduce tag count |
| Query timeouts for long ranges | Large data volume | Implement data aggregation, use InfluxDB |
| Missing data gaps | Device connection issues | Check device connectivity, enable alarms for disconnections |
| Incorrect timestamps | Time zone issues | Verify server time zone, use UTC timestamps |
| Chart rendering slow | Too many data points | Limit displayed points, implement downsampling |

---

## Conclusion

The FUXA DAQ system provides a comprehensive solution for historical data management in SCADA/HMI applications. By understanding the architecture, database structure, recording logic, API endpoints, and frontend implementation, you can effectively implement and optimize historical data collection and visualization in your projects.

**Key Takeaways:**

1. **Flexible Storage**: Supports SQLite (lightweight) and InfluxDB (high-performance)
2. **Configurable Recording**: Time-based, change-based, or hybrid strategies
3. **REST API**: Comprehensive API for querying historical data
4. **Rich Visualization**: Support for Line, Bar, Area, and custom charts with Apache ECharts
5. **Scalability**: Designed to handle from tens to thousands of tags
6. **Retention Policies**: Automatic archiving based on configurable retention periods

**Next Steps:**

1. Enable DAQ for your critical tags
2. Configure appropriate recording intervals and deadbands
3. Create historical charts for your views
4. Implement monitoring and maintenance procedures
5. Optimize based on your specific requirements

For additional support, refer to:
- FUXA GitHub Repository: https://github.com/frangoteam/FUXA
- FUXA Wiki: https://github.com/frangoteam/FUXA/wiki
- Community Forums and Discussions

---

*Document Version: 1.0*  
*Last Updated: November 10, 2025*  
*Author: Senior Full-Stack Developer*
