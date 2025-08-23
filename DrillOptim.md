# Drilling Optimization System - Technical Implementation Guide

## System Overview

This system implements real-time drilling parameter optimization based on field-validated PID control protocols. It monitors WITS0 data streams, compares against optimal operating curves, and generates actionable insights.

```mermaid
graph TB
    subgraph "Rig Site"
        WITS[WITS0 Data Stream<br/>1Hz Sampling]
        AD[Autodriller<br/>PID Controller]
    end
    
    subgraph "Data Processing Pipeline"
        VAL[Data Validator<br/>Unit Conversion<br/>Quality Checks]
        CONF[Conformance Engine<br/>Protocol Comparison]
        INS[Insights Generator<br/>Real-time Analysis]
    end
    
    subgraph "Storage & API"
        TS[(Time Series DB<br/>InfluxDB/TimescaleDB)]
        API[REST API<br/>WebSocket Stream]
    end
    
    subgraph "User Interface"
        DASH[Driller Dashboard<br/>Real-time KPIs]
        ALERT[Alert System<br/>SMS/Email]
    end
    
    WITS -->|TCP/Serial| VAL
    VAL --> CONF
    CONF --> INS
    INS --> TS
    TS --> API
    API --> DASH
    INS --> ALERT
    ALERT -.->|Parameter Adjustment| AD
```

## Core Components

### 1. PID Configuration Protocol

Based on field tests with **187.27 m/h ROP achieved** using optimal settings:

```mermaid
stateDiagram-v2
    [*] --> DepthCheck: New Depth Data
    
    DepthCheck --> FastDrilling: Rapid Formation
    DepthCheck --> Transition: Intercalations
    
    FastDrilling --> SetPID_Fast: Apply Fast PID
    Transition --> SetPID_Stable: Apply Stable PID
    
    SetPID_Fast --> Monitor
    SetPID_Stable --> Monitor
    
    Monitor --> Alert: Out of Curve
    Monitor --> Continue: In Curve
    
    Alert --> [*]
    Continue --> [*]
    
    note right of SetPID_Fast
        WOB: FG=2.0, TC=0.2
        DP: FG=1.0, TC=0.2
        Torque: FG=1.0
        ROP Limit: >300 m/h
    end note
    
    note right of SetPID_Stable
        WOB: FG=0.5, TC=0.2
        DP: FG=1.0, TC=0.2
        Torque: FG=1.0
        ROP Limit: 120% avg
    end note
```

### 2. Data Flow Architecture

```mermaid
sequenceDiagram
    participant W as WITS0 Feed
    participant V as Validator
    participant C as Conformance
    participant I as Insights
    participant D as Database
    participant U as UI/Dashboard
    
    loop Every 1 second
        W->>V: Raw WITS0 Records
        Note over V: Validate & Convert Units
        V->>C: Clean Data
        Note over C: Compare vs Protocol Curve
        C->>I: Conformance Metrics
        Note over I: Generate Insights
        I->>D: Store Results
        I->>U: Push Updates
        
        alt Conformance < 70%
            I-->>U: Trigger Alert
            U-->>W: Adjust Parameters
        end
    end
```

## Implementation Details

### WITS0 Data Structure

**Required WITS0 Records:**

| Record | Parameter | Unit | Field | Description | Valid Range |
|--------|-----------|------|-------|-------------|-------------|
| 01 | DBTM | m | 08-12 | Depth Bit | 0-10000 |
| 01 | ROPA | m/h | 13-17 | Rate of Penetration Avg | 0-500 |
| 01 | WOBA | klbs | 18-22 | Weight on Bit Avg | 0-50 |
| 01 | RPMA | rpm | 28-32 | Rotary Speed Avg | 0-200 |
| 01 | TQA | ft-lbs | 33-37 | Torque Avg | 0-20000 |
| 01 | SPPA | psi | 38-42 | Standpipe Pressure | 0-6000 |
| 12 | MFOP | gpm | 18-22 | Mud Flow Out % | 0-150 |

### Operating Envelope Definition

```python
# config/drilling_zones.yaml
zones:
  fast_drilling:
    depth_ranges:
      - [0, 500]      # Surface
      - [800, 1800]   # Rapid formation
    pid_settings:
      wob_gain_factor: 2.0
      wob_time_constant: 0.2
      wob_closure_limit: 2.0
    targets:
      rop_min: 150
      rop_max: 300
      wob_max: 18
      rpm: 100
      torque_max: 15000
    tolerances:
      wob_error: 0.33  # 33% acceptable
      torque_std: 605   # ft-lbs
      
  formation_transition:
    depth_ranges:
      - [500, 800]
      - [1800, 2500]
    pid_settings:
      wob_gain_factor: 0.5
      wob_time_constant: 0.2
      wob_closure_limit: 0.0
    targets:
      rop_min: 80
      rop_max: 150
      wob_max: 15
      rpm: 100
      torque_max: 14000
    tolerances:
      wob_error: 0.59  # 59% acceptable
      torque_std: 629   # ft-lbs
```

### Core Algorithms

#### Mechanical Specific Energy (MSE) Calculation

```python
def calculate_mse(wob_klbs: float, rpm: float, rop_mh: float, 
                  torque_ftlbs: float, bit_diameter_in: float = 8.5) -> float:
    """
    MSE = (WOB/Ab) + (120*π*RPM*T)/(Ab*ROP)
    
    Field Validated Ranges (from tests):
    - Normal: 2000-5000 psi
    - Warning: 5000-8000 psi  
    - Critical: >8000 psi
    """
    # Convert units
    wob_lbf = wob_klbs * 1000
    rop_fthr = rop_mh * 3.28084
    
    # Bit area
    bit_area = math.pi * (bit_diameter_in ** 2) / 4
    
    # MSE components
    mse_weight = wob_lbf / bit_area
    mse_rotation = (120 * math.pi * rpm * torque_ftlbs) / (bit_area * rop_fthr)
    
    return mse_weight + mse_rotation
```

#### Conformance Score Algorithm

```python
def calculate_conformance_score(metrics: Dict) -> float:
    """
    Weighted scoring based on field test results
    
    Weights determined from protocol testing:
    - ROP achievement: 35%
    - WOB control: 25%
    - Torque stability: 20%
    - MSE efficiency: 20%
    """
    score = 0.0
    
    # ROP Performance (35%)
    rop_efficiency = metrics['rop_actual'] / metrics['rop_target_max']
    score += min(35, rop_efficiency * 35)
    
    # WOB Control (25%)
    wob_error = metrics['wob_error_pct']
    expected_error = 33 if metrics['zone'] == 'fast_drilling' else 59
    wob_score = max(0, 1 - (wob_error / expected_error)) * 25
    score += wob_score
    
    # Torque Stability (20%)
    torque_std = metrics['torque_std']
    expected_std = 605 if metrics['zone'] == 'fast_drilling' else 629
    torque_score = max(0, 1 - (torque_std / expected_std)) * 20
    score += torque_score
    
    # MSE Efficiency (20%)
    mse = metrics['mse']
    if mse < 5000:
        score += 20
    elif mse < 8000:
        score += 10
    
    return min(100, max(0, score))
```

### Database Schema

```mermaid
erDiagram
    WITS_RAW ||--o{ CONFORMANCE : generates
    CONFORMANCE ||--o{ INSIGHTS : triggers
    CONFORMANCE ||--|| DRILLING_ZONES : references
    
    WITS_RAW {
        timestamp datetime PK
        depth_m float
        rop_mh float
        wob_klbs float
        rpm float
        torque_ftlbs float
        sppa_psi float
        mflow_gpm float
    }
    
    CONFORMANCE {
        id uuid PK
        timestamp datetime
        depth_m float
        zone_type string
        conformance_score float
        rop_efficiency float
        wob_error_pct float
        mse_psi float
        in_curve boolean
    }
    
    INSIGHTS {
        id uuid PK
        timestamp datetime
        type string
        severity string
        message text
        metrics json
        acknowledged boolean
    }
    
    DRILLING_ZONES {
        depth_start float PK
        depth_end float
        zone_type string
        pid_config json
        targets json
    }
```

### API Endpoints

```yaml
# REST API Specification
endpoints:
  # Real-time data
  GET /api/v1/conformance/current:
    description: Current conformance status
    response:
      depth: 1250.5
      zone: "fast_drilling"
      conformance_score: 85.2
      in_curve: true
      metrics:
        rop_actual: 165.5
        rop_target: 180.0
        wob_error_pct: 28.5
        mse: 4250
        
  # Historical analysis  
  GET /api/v1/conformance/history:
    parameters:
      start_depth: 1000
      end_depth: 1500
      resolution: "1m"  # 1 minute aggregation
    response:
      - timestamp: "2024-01-15T10:00:00Z"
        avg_conformance: 82.3
        rop_avg: 162.4
        wob_avg: 12.5
        
  # Insights
  GET /api/v1/insights/active:
    response:
      - id: "uuid-123"
        type: "ROP_OPTIMIZATION"
        severity: "MEDIUM"
        message: "ROP 25% below target"
        metrics:
          current_rop: 120
          target_rop: 160
          
  # WebSocket stream
  WS /api/v1/stream:
    message_format:
      type: "conformance_update"
      data:
        timestamp: "2024-01-15T10:00:00Z"
        conformance_score: 85.2
        rop: 165.5
        wob: 12.8
```

### Real-time Processing Pipeline

```python
# src/pipeline/wits_processor.py
import asyncio
from typing import Dict, List
import numpy as np
from collections import deque

class WITSProcessor:
    def __init__(self, buffer_size: int = 300):
        """
        Buffer size = 300 seconds (5 minutes at 1Hz)
        Based on field tests showing optimal response at 5-min windows
        """
        self.buffer = deque(maxlen=buffer_size)
        self.zone_detector = ZoneDetector()
        self.conformance_calculator = ConformanceCalculator()
        
    async def process_record(self, wits_record: Dict) -> Dict:
        """
        Process single WITS0 record
        """
        # 1. Validate data
        if not self._validate_record(wits_record):
            return {"status": "invalid", "reason": "data_validation_failed"}
            
        # 2. Detect zone
        zone = self.zone_detector.get_zone(wits_record['depth'])
        
        # 3. Calculate instantaneous metrics
        metrics = {
            'timestamp': wits_record['timestamp'],
            'depth': wits_record['depth'],
            'zone': zone,
            'rop': wits_record['rop'],
            'wob': wits_record['wob'],
            'rpm': wits_record['rpm'],
            'torque': wits_record['torque'],
            'mse': calculate_mse(
                wits_record['wob'],
                wits_record['rpm'],
                wits_record['rop'],
                wits_record['torque']
            )
        }
        
        # 4. Add to buffer for trending
        self.buffer.append(metrics)
        
        # 5. Calculate conformance
        conformance = self.conformance_calculator.calculate(
            metrics, 
            self._get_targets(zone)
        )
        
        # 6. Detect anomalies
        anomalies = self._detect_anomalies()
        
        return {
            'status': 'success',
            'metrics': metrics,
            'conformance': conformance,
            'anomalies': anomalies
        }
    
    def _validate_record(self, record: Dict) -> bool:
        """
        Validate WITS0 record based on field ranges
        """
        validations = [
            0 <= record.get('depth', -1) <= 10000,
            0 <= record.get('rop', -1) <= 500,
            0 <= record.get('wob', -1) <= 50,
            0 <= record.get('rpm', -1) <= 200,
            0 <= record.get('torque', -1) <= 20000
        ]
        return all(validations)
    
    def _detect_anomalies(self) -> List[Dict]:
        """
        Detect drilling dysfunctions
        """
        anomalies = []
        
        if len(self.buffer) < 30:  # Need 30 seconds minimum
            return anomalies
            
        recent_data = list(self.buffer)[-30:]
        
        # Stick-slip detection (torque oscillation)
        torque_values = [d['torque'] for d in recent_data]
        torque_std = np.std(torque_values)
        if torque_std > 1000:  # From field tests
            anomalies.append({
                'type': 'STICK_SLIP',
                'severity': 'HIGH',
                'value': torque_std
            })
            
        # Sudden ROP drop
        rop_values = [d['rop'] for d in recent_data]
        if len(rop_values) > 10:
            rop_drop = (np.mean(rop_values[-10:-5]) - np.mean(rop_values[-5:])) / np.mean(rop_values[-10:-5])
            if rop_drop > 0.3:  # 30% drop
                anomalies.append({
                    'type': 'ROP_DROP',
                    'severity': 'MEDIUM',
                    'value': rop_drop * 100
                })
                
        return anomalies
```

### Performance Benchmarks

Based on field test data from PO-1233 and PO-1212:

```mermaid
graph LR
    subgraph "Fast Drilling Zone"
        FD_TARGET[Target:<br/>ROP: 300 m/h<br/>WOB: 18 klbs]
        FD_ACHIEVED[Achieved:<br/>ROP: 187.27 m/h<br/>WOB: 12.71 klbs<br/>62% Efficiency]
        FD_TARGET --> FD_ACHIEVED
    end
    
    subgraph "Transition Zone"
        TZ_TARGET[Target:<br/>ROP: 150 m/h<br/>WOB: 15 klbs]
        TZ_ACHIEVED[Achieved:<br/>ROP: 130.60 m/h<br/>WOB: 7.37 klbs<br/>87% Efficiency]
        TZ_TARGET --> TZ_ACHIEVED
    end
```

### Insight Generation Rules

```python
# src/insights/rules.py
class InsightRules:
    """
    Rule-based insight generation from field-validated thresholds
    """
    
    RULES = {
        'ROP_SUBOPTIMAL': {
            'condition': lambda m: m['rop_efficiency'] < 0.7,
            'severity': lambda m: 'HIGH' if m['rop_efficiency'] < 0.5 else 'MEDIUM',
            'message': lambda m: (
                f"ROP at {m['rop_actual']:.1f} m/h is {(1-m['rop_efficiency'])*100:.0f}% "
                f"below target {m['rop_target']:.1f} m/h. "
                f"{'Increase WOB gain to 2.0' if m['zone']=='fast_drilling' else 'Check formation change'}"
            )
        },
        
        'WOB_CONTROL_POOR': {
            'condition': lambda m: m['wob_error_pct'] > (40 if m['zone']=='fast_drilling' else 65),
            'severity': 'HIGH',
            'message': lambda m: (
                f"WOB error {m['wob_error_pct']:.1f}% exceeds acceptable range. "
                f"Verify autodriller PID settings: FG={2.0 if m['zone']=='fast_drilling' else 0.5}, TC=0.2"
            )
        },
        
        'MSE_HIGH': {
            'condition': lambda m: m['mse'] > 8000,
            'severity': 'HIGH',
            'message': lambda m: (
                f"MSE at {m['mse']:.0f} psi indicates poor drilling efficiency. "
                f"Consider: bit condition, formation change, or parameter optimization"
            )
        },
        
        'TORQUE_OSCILLATION': {
            'condition': lambda m: m['torque_std'] > 1000,
            'severity': 'MEDIUM',
            'message': lambda m: (
                f"Torque oscillation {m['torque_std']:.0f} ft-lbs indicates possible stick-slip. "
                f"Reduce WOB time constant to 0.2 for faster response"
            )
        }
    }
    
    @classmethod
    def evaluate(cls, metrics: Dict) -> List[Dict]:
        """
        Evaluate all rules against current metrics
        """
        insights = []
        
        for rule_name, rule in cls.RULES.items():
            if rule['condition'](metrics):
                insights.append({
                    'type': rule_name,
                    'severity': rule['severity'](metrics) if callable(rule['severity']) else rule['severity'],
                    'message': rule['message'](metrics),
                    'timestamp': metrics['timestamp'],
                    'depth': metrics['depth']
                })
                
        return insights
```

### Deployment Configuration

```yaml
# docker-compose.yml
version: '3.8'

services:
  wits-receiver:
    image: drilling-opt/wits-receiver:latest
    ports:
      - "5000:5000"  # WITS0 TCP port
    environment:
      WITS_FORMAT: "WITS0"
      BUFFER_SIZE: 300
    volumes:
      - ./config:/app/config
      
  processor:
    image: drilling-opt/processor:latest
    depends_on:
      - wits-receiver
      - timescaledb
    environment:
      PROCESSING_INTERVAL: 1
      ZONE_CONFIG: /app/config/drilling_zones.yaml
      
  timescaledb:
    image: timescale/timescaledb:latest-pg14
    environment:
      POSTGRES_DB: drilling_opt
      POSTGRES_USER: drilling
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - timescale-data:/var/lib/postgresql/data
      
  api:
    image: drilling-opt/api:latest
    ports:
      - "8080:8080"
    depends_on:
      - timescaledb
      
  dashboard:
    image: drilling-opt/dashboard:latest
    ports:
      - "3000:3000"
    depends_on:
      - api

volumes:
  timescale-data:
```

### Alert Configuration

```python
# src/alerts/config.py
ALERT_THRESHOLDS = {
    'CRITICAL': {
        'conformance_score': 50,  # Below 50% for 5+ minutes
        'duration_minutes': 5,
        'actions': ['sms', 'email', 'dashboard_popup'],
        'recipients': ['drilling_supervisor', 'company_man']
    },
    
    'WARNING': {
        'conformance_score': 70,  # Below 70% for 10+ minutes
        'duration_minutes': 10,
        'actions': ['email', 'dashboard_warning'],
        'recipients': ['drilling_engineer']
    },
    
    'INFO': {
        'rop_efficiency': 0.7,  # Below 70% of target
        'duration_minutes': 15,
        'actions': ['dashboard_notification'],
        'recipients': ['driller']
    }
}

# Specific dysfunction alerts
DYSFUNCTION_ALERTS = {
    'STICK_SLIP': {
        'threshold': {'torque_std': 1000},  # ft-lbs
        'message': "Stick-slip detected. Reduce RPM or adjust WOB",
        'severity': 'HIGH'
    },
    
    'BIT_BALLING': {
        'threshold': {'mse_increase': 1.5, 'rop_decrease': 0.3},  # 50% MSE increase, 30% ROP drop
        'message': "Possible bit balling. Check hydraulics and consider POOH",
        'severity': 'HIGH'
    }
}
```

## System Testing & Validation

### Test Data Sets

From field reports PO-1233 and PO-1212:

| Test Case | Zone | PID Config | Expected ROP | Achieved ROP | Conformance |
|-----------|------|------------|--------------|--------------|-------------|
| Fast Drilling | 750-850m | FG=2.0, TC=0.2 | 180 m/h | 187.27 m/h | 92% |
| Transition | 1250-1300m | FG=0.5, TC=0.2 | 130 m/h | 130.60 m/h | 87% |
| CwD Mode | 2020-2055m | FG=0.5, TC=0.2 | 35 m/h | 34.93 m/h | 95% |

### Validation Metrics

```python
# tests/validation.py
def validate_system_performance(test_data: pd.DataFrame) -> Dict:
    """
    Validate system against known field test results
    """
    results = {
        'rop_accuracy': [],
        'wob_control': [],
        'conformance_scores': []
    }
    
    # Test Case 1: Fast Drilling
    fast_drilling_data = test_data[
        (test_data['depth'] >= 750) & 
        (test_data['depth'] <= 850)
    ]
    
    expected_rop = 187.27  # From field test
    achieved_rop = fast_drilling_data['rop'].mean()
    rop_accuracy = 1 - abs(achieved_rop - expected_rop) / expected_rop
    
    assert rop_accuracy > 0.9, f"ROP accuracy {rop_accuracy:.2%} below 90%"
    
    # Test Case 2: WOB Control
    expected_wob_error = 0.33  # 33% from field test
    actual_wob_error = fast_drilling_data['wob_error_pct'].mean() / 100
    
    assert actual_wob_error < expected_wob_error * 1.2, \
           f"WOB error {actual_wob_error:.2%} exceeds tolerance"
    
    return {
        'status': 'PASSED',
        'rop_accuracy': rop_accuracy,
        'wob_control': actual_wob_error,
        'test_cases_passed': 2
    }
```

## Monitoring & Maintenance

### Key Performance Indicators

```mermaid
graph TD
    subgraph "Real-time KPIs"
        KPI1[Conformance Score<br/>Target: >70%]
        KPI2[ROP Efficiency<br/>Target: >80%]
        KPI3[WOB Error<br/>Target: <40%]
        KPI4[MSE Trend<br/>Target: Stable/Decreasing]
    end
    
    subgraph "Operational Metrics"
        OM1[Footage Drilled<br/>In-Curve]
        OM2[Time In-Curve<br/>Percentage]
        OM3[Alert Response<br/>Time]
        OM4[Parameter<br/>Adjustments/Day]
    end
    
    subgraph "System Health"
        SH1[Data Quality<br/>>95% Valid]
        SH2[Processing Latency<br/><1 second]
        SH3[Database Performance<br/><100ms queries]
        SH4[API Uptime<br/>>99.9%]
    end
```

### Troubleshooting Guide

| Issue | Symptoms | Check | Action |
|-------|----------|-------|--------|
| Low Conformance | Score <50% consistently | Zone detection, PID settings | Verify depth ranges, check autodriller config |
| High WOB Error | Error >50% | Sensor calibration, PID gains | Recalibrate sensors, adjust gain factor |
| No Data Flow | Dashboard blank | WITS connection, network | Check TCP port 5000, verify WITS format |
| Incorrect MSE | Values >20000 psi | Unit conversion, bit diameter | Verify units (metric/imperial), update bit size |
| Delayed Insights | >5 second lag | Processing queue, database | Scale processor instances, optimize queries |

## System Requirements

### Hardware
- **Server**: 8 CPU cores, 16GB RAM minimum
- **Storage**: 500GB SSD for 6 months data retention
- **Network**: 1Gbps connection to rig network

### Software
- **OS**: Ubuntu 20.04 LTS or RHEL 8
- **Runtime**: Python 3.9+, Node.js 16+
- **Database**: TimescaleDB 2.0+ or InfluxDB 2.0+
- **Container**: Docker 20.10+, Kubernetes 1.21+ (optional)

### Data Requirements
- **WITS0 Feed**: TCP/IP or Serial connection
- **Sampling Rate**: 1Hz minimum
- **Latency**: <500ms from rig floor to processor
- **Quality**: >95% valid records

---

**Version**: 1.0.0  
**Last Updated**: January 2024  
**Validated Against**: Field Reports PO-1233, PO-1212  
**Protocol Reference**: "Estandarización de seteo de parámetros de perforación en Autodriller"
