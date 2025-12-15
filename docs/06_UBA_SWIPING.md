# uba-swiping Service Documentation

## Table of Contents
1. [Description](#description)
2. [API Documentation](#api-documentation)
3. [Setup from Scratch](#setup-from-scratch)
4. [Dependencies](#dependencies)
5. [Configuration](#configuration)
6. [Deployment](#deployment)
7. [Process Flow](#process-flow)
8. [Database Interactions](#database-interactions)
9. [Additional Information](#additional-information)

---

## Description

- **Service Name:** `uba-swiping`
- **Type:** ML Inference Service (Mobile Touch/Swipe Pattern Detection)
- **Language:** Python 3.12
- **Framework:** BentoML + XGBoost
- **Port:** 3003 (configurable)

### Purpose

The `uba-swiping` service  detects bot behavior by analyzing touch/swipe patterns and dynamics using an XGBoost model trained on touch/swipe behavior features to distinguish between human and automated mobile touch interaction.

### Features

**Gesture Analysis:**
- **Touch Sequences:** Analyzes patterns across touch events (touchstart → touchmove → touchend)
- **Velocity Dynamics:** Measures speed and acceleration of finger movements
- **Trajectory Analysis:** Evaluates path smoothness, curvature, and linearity

**Behavioral Features (200+ features across 6 categories):**
- **Velocity Features (40+):** Instantaneous velocity, average velocity, acceleration patterns, peak velocity, velocity consistency
- **Trajectory Features (50+):** Path curvature, linearity, direction changes, straightness indices, deviation from ideal paths
- **Pressure Features (30+):** Average pressure, pressure variance, pressure gradient, peak pressure points, pressure consistency
- **Temporal Features (40+):** Touch duration (dwell time), inter-gesture intervals, gesture completion time, pause patterns, timing consistency
- **Spatial Features (30+):** Touch coordinates distribution, screen region preferences, touch point clustering, movement patterns, contact area variations
- **Touch Point Features (10+):** Contact area size, touch shape, multi-touch coordination, simultaneous touch count, touch point stability

**Bot Detection Indicators:**
- Perfect straight-line swipes (no natural hand jitter)
- Constant velocity throughout gesture
- No pressure variation
- Perfectly regular timing intervals
- Missing finger wobble or micro-adjustments
- Identical repeated gestures
- Touch coordinates aligned to perfect grid positions

### Model Information

- **Algorithm:** XGBoost (Gradient Boosting) Classifier
- **Training Data:** Human vs. bot mobile touch/swipe sessions
- **Features:** 200+ engineered features across 6 categories
- **Performance:** ~90-95% accuracy on test set
- **Model Path:** `swipe_bot_detector/checkpoints/xgboost/15_6/dev/`
- **Framework:** XGBoost + BentoML
- **Threshold:** 0.7 (probability > 0.7 = BOT)

---

## API Documentation

> **Complete API Reference:** For detailed API documentation including request/response schemas, feature explanations, touch dynamics concepts, and code examples, see the **[OpenAPI Specification](../api/uba-swiping-openapi.yaml)**

### Quick Reference

| Aspect             | Value                                                    |
| ------------------ | -------------------------------------------------------- |
| **Port**           | 3004                                                     |
| **Protocol**       | HTTP REST                                                |
| **Language**       | Python 3.12                                              |
| **Framework**      | BentoML + XGBoost                                        |
| **Model**          | XGBoost Gradient Boosting                                |
| **Features**       | 200+ (velocity, trajectory, pressure, temporal, spatial, touch point)                              |
| **Accuracy**       | ~90-95%                                                  |
| **Inference Time** | 50-150ms                                                  |
| **Workers**        | 4 (default)                                              |
| **Database**       | None (stateless)                                         |
| **API Spec**       | [uba-swiping-openapi.yaml](../api/uba-swiping-openapi.yaml) |

**Base URL:**
- Development: `http://localhost:3003`
- Docker: `http://uba-swiping:3003`

**Authentication:** None (internal service)
**Data Format:** JSON

### Endpoint Overview

#### POST /predict

Analyzes touch/swipe patterns to classify behavior as BOT or Human.

**Request:**
```json
{
  "data": [
    {
      "timestamp": 1234567890123,
      "button": "left",
      "state": "touchstart",
      "x": 200.5,
      "y": 400.2,
      "key": "",
      "deltaY": 0
    },
    {
      "timestamp": 1234567890173,
      "button": "left",
      "state": "touchmove",
      "x": 220.3,
      "y": 395.8,
      "key": "",
      "deltaY": 0
    },
    {
      "timestamp": 1234567890273,
      "button": "left",
      "state": "touchend",
      "x": 270.1,
      "y": 380.5,
      "key": "",
      "deltaY": 0
    }
  ]
}
```

**Response:**
```json
["BOT", 0.85]
```
Or:
```json
["Human", 0.92]
```

**Quick Test:**
```bash
curl -X POST http://localhost:3003/predict \
  -H "Content-Type: application/json" \
  -d '{
    "data": [
      {"timestamp": 1234567890123, "button": "left", "state": "touchstart",
       "x": 200, "y": 400, "key": "", "deltaY": 0},
      {"timestamp": 1234567890173, "button": "left", "state": "touchmove",
       "x": 220, "y": 395, "key": "", "deltaY": 0},
      {"timestamp": 1234567890273, "button": "left", "state": "touchend",
       "x": 270, "y": 380, "key": "", "deltaY": 0}
    ]
  }'
```

### Key Data Structures

**Input Event Schema:**
```python
{
    "timestamp": int,      # Unix timestamp in milliseconds
    "button": str,         # "left" for touch events
    "state": str,          # "touchstart" | "touchmove" | "touchend"
    "x": float,            # Touch X coordinate in pixels
    "y": float,            # Touch Y coordinate in pixels
    "key": str,            # Always "" for touch events
    "deltaY": float        # Always 0 for touch events
}
```

**Output Schema:**
```python
Tuple[str, float]
# ["BOT" | "Human", probability_score]
# Example: ("BOT", 0.85) or ("Human", 0.92)
```
### External Integrations

| Service | Purpose           | Protocol  |
| ------- | ----------------- | --------- |
| uba-api | Request gateway   | HTTP POST |

---

## Setup 

### Prerequisites

- Python 3.12 or higher
- pip or poetry package manager
- git (for cloning repository)
- **Docker:** Optional, for containerized deployment
- **Model Files:** Pre-trained models in checkpoints directory

### Step 1: Navigate to Service Directory

```bash
cd uba-swiping
```

### Step 2: Install Dependencies

```bash
pip install -r requirements.txt
```

### Step 3: Verify Package Installation

```bash
python -c "from swipe_bot_detector import ScrollClassifier; print('Package loaded successfully')"
```

### Step 4: Run Development Server

```bash
bentoml serve service:ScrollService --port 3003 --host 0.0.0.0
```

### Step 5: Verify Server is Running

```bash
curl http://localhost:3003/healthz
```

### Step 6: Test Prediction Endpoint

```bash
curl -X POST http://localhost:3003/predict \
  -H "Content-Type: application/json" \
  -d '{
    "data": [
      {"timestamp": 1234567890, "button": "scroll", "state": "move",
       "x": 500, "y": 300, "key": "", "deltaY": -120},
      {"timestamp": 1234567890100, "button": "scroll", "state": "move",
       "x": 500, "y": 400, "key": "", "deltaY": -100},
      {"timestamp": 1234567890200, "button": "scroll", "state": "move",
       "x": 500, "y": 500, "key": "", "deltaY": -95}
    ]
  }'
```

Expected response:
```json
["Human", 0.XX]
```

---

## Dependencies

### Runtime Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| bentoml | latest | ML model serving framework |
| pandas | ==2.2.3 | Data manipulation and feature extraction |
| numpy | ==2.0.2 | Numerical computations |
| xgboost | ==3.0.0 | Gradient boosting model |
| scikit-learn | ==1.6.1 | ML utilities and preprocessing |
| pydantic | ==2.10.6 | Data validation |
| scipy | ==1.13.1 | Scientific computing |
| python-dotenv | latest | Environment variable management |
| seaborn | latest | Statistical visualizations |
| mlflow | latest | Model tracking and versioning |
| ipykernel | latest | Jupyter notebook support |
| hyperopt | latest | Hyperparameter tuning |

### Python Package Structure

```
swipe_bot_detector/
├── __init__.py
├── scroll_classifier.py      # Main ScrollClassifier class
├── feature_extractor.py      # Feature engineering pipeline
├── models.py                  # Data validation models
├── core/
│   └── model.py               # XGBoost wrapper class
└── checkpoints/
    └── xgboost/
        └── 15_6/
            └── dev/           # Trained model files
```


---

## Configuration

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `WORKERS` | No | `4` | Number of BentoML worker processes |
| `PORT` | No | `3004` | HTTP server port |
| `LOG_LEVEL` | No | `INFO` | Logging level (DEBUG, INFO, WARN, ERROR) |
| `WINDOW_HEIGHT` | No | `1080` | Window height for deltaY normalization |

### BentoML Service Configuration

**File:** `service.py`

```python
import bentoml
import os

@bentoml.service(
    workers=int(os.getenv("WORKERS", default=4)),
    metrics={
        "enabled": False,
        "namespace": "bentoml_service"
    }
)
class ScrollService:
    def __init__(self):
        self.pipeline = ScrollClassifier()
```

**Configuration Options:**
- `workers`: Number of parallel worker processes (default: 4)
- `metrics.enabled`: Enable Prometheus metrics (default: False)
- `metrics.namespace`: Metrics namespace prefix

### Model Configuration

Models are loaded from the `checkpoints/` directory at service startup:

```python
# Model loaded automatically from
swipe_bot_detector/checkpoints/xgboost/15_6/dev/
├── model files (XGBoost artifacts)
```

---

## Deployment

### Docker Deployment

#### Build Docker Image

```bash
docker build -t uba-swiping:latest .
```

#### Run Container

```bash
docker run -d \
  --name uba-swiping \
  -p 3004:3003 \
  -e WORKERS=4 \
  uba-swiping:latest
```

#### View Logs

```bash
docker logs -f uba-swiping
```

#### Stop Container

```bash
docker stop uba-swiping
docker rm uba-swiping
```

### Docker Compose

**File:** `docker-compose.yml`

```yaml
services:
  uba-swiping:
    image: stormbird/uba-swiping:latest
    container_name: uba-swiping
    environment:
      - WORKERS=4
      - LOG_LEVEL=INFO
    command: bentoml serve service:ScrollService --port 3003 --host 0.0.0.0
    ports:
      - "3004:3003"
    networks:
      - uba-network
    restart: unless-stopped

networks:
  uba-network:
    driver: bridge
```

**Start service:**
```bash
docker-compose up -d uba-swiping
```

**View logs:**
```bash
docker-compose logs -f uba-swiping
```

**Stop service:**
```bash
docker-compose down
```

---
## Process Flow

### Activity Diagram - ML Inference Pipeline

This flowchart illustrates the complete touch/swipe analysis workflow from request to prediction.

```mermaid
flowchart TD
    Start([HTTP POST /predict]) --> Validate[Validate Input Schema]
    Validate --> Preprocess[Preprocess Touch Events]

    Preprocess --> NormalizeTime[Normalize Timestamps]
    NormalizeTime --> GroupGestures[Group by 1-Second Windows]
    GroupGestures --> FilterEvents[Filter Touch Events Only]

    FilterEvents --> ExtractFeatures[Extract 200+ Features]
    ExtractFeatures --> VelocityFeats[Velocity Features:<br/>40+ features]
    ExtractFeatures --> TrajectoryFeats[Trajectory Features:<br/>50+ features]
    ExtractFeatures --> PressureFeats[Pressure Features:<br/>30+ features]
    ExtractFeatures --> TemporalFeats[Temporal Features:<br/>40+ features]
    ExtractFeatures --> SpatialFeats[Spatial Features:<br/>30+ features]
    ExtractFeatures --> TouchFeats[Touch Point Features:<br/>10+ features]

    VelocityFeats --> CombineFeatures[Combine Feature Matrix]
    TrajectoryFeats --> CombineFeatures
    PressureFeats --> CombineFeatures
    TemporalFeats --> CombineFeatures
    SpatialFeats --> CombineFeatures
    TouchFeats --> CombineFeatures

    CombineFeatures --> XGBoostPredict[XGBoost Prediction]
    XGBoostPredict --> CalcProb[Calculate Mean Probability]
    CalcProb --> Threshold{Probability<br/>> 0.7?}

    Threshold -->|Yes| BotLabel[Label: BOT, Prob: p]
    Threshold -->|No| HumanLabel[Label: Human, Prob: 1-max p,0.5]

    BotLabel --> ErrorCheck{Error<br/>Occurred?}
    HumanLabel --> ErrorCheck

    ErrorCheck -->|Yes| SafeDefault[Return 'Human', 1.0]
    ErrorCheck -->|No| ReturnResult[Return Label, Probability]

    SafeDefault --> End([HTTP 200 Response])
    ReturnResult --> End

    style Start fill:#90EE90
    style End fill:#FFB6C1
    style Threshold fill:#FFE4B5
    style XGBoostPredict fill:#DDA0DD
    style SafeDefault fill:#FF6B6B
    style ExtractFeatures fill:#87CEEB
```

---

### Data Flow Diagram - Service Interactions

This diagram shows how the uba-swiping service integrates with the UBA ecosystem.

```mermaid
graph TB
    subgraph Client["Mobile Client"]
        MobileApp[Mobile App<br/>Touch/Swipe Events]
    end

    subgraph Collector["client-collector :3000"]
        WS[WebSocket Server]
    end

    subgraph Gateway["uba-api :7018"]
        API[API Gateway]
        Router[Request Router]
    end

    subgraph MLServices["ML Inference Services"]
        Mouse[uba-mouse :3001]
        Keyboard[uba-keyboard :3002]
        Scroll[uba-scroll :3004]

        subgraph SwipeService["uba-swiping :3003"]
            Endpoint[/predict Endpoint]
            Pipeline[Swipe Pipeline]
            Preprocessor[Event Preprocessor]
            FeatureEngine[Feature Extractor<br/>200+ features]
            Model[XGBoost Classifier]
        end
    end

    MobileApp -->|"1. Encrypted<br/>Touch Events"| WS
    WS -->|"2. Decrypt &<br/>Forward"| API
    API --> Router

    Router -.->|"Parallel"| Mouse
    Router -.->|"Requests"| Keyboard
    Router -.->|"Parallel"| Scroll
    Router -->|"3. POST /predict<br/>{data: events}"| Endpoint

    Endpoint --> Pipeline
    Pipeline --> Preprocessor
    Preprocessor -->|"Normalized<br/>Touch Events"| FeatureEngine
    FeatureEngine -->|"6 Categories:<br/>Velocity, Trajectory,<br/>Pressure, Temporal,<br/>Spatial, Touch Point"| Model
    Model -->|"4. ['BOT', 0.85]"| Endpoint
    Endpoint -->|"5. Prediction"| Router
    Router -->|"6. Aggregated<br/>Response"| API
    API -->|"7. Result"| WS
    WS -->|"8. Response"| MobileApp

    style MobileApp fill:#E8F5E9
    style Endpoint fill:#BBDEFB
    style Model fill:#CE93D8
    style FeatureEngine fill:#FFE082
    style Preprocessor fill:#FFCCBC
```

1. **Request Reception**
   - BentoML receives HTTP POST to `/predict`
   - Validates request body against `DataframeSchema`
   - Converts JSON to pandas DataFrame

2. **Event Preprocessing** (`feature_extractor.py:8-25`)
   - **Validation:** Validate events against RawData schema
   - **Sorting:** Sort events by timestamp
   - **Time Diff:** Calculate time difference between consecutive events
   - **Filtering:** Filter only touch events (button == "scroll")
   - **Normalization:** Convert deltaY to pixels if needed (multiply by WINDOW_HEIGHT if values < 1)
   - **Action Grouping:** Group touch events by 1-second windows (scroll_timediff > 1000ms)
   - Output: Grouped touch events with action_id

1. **Feature Extraction** (`feature_extractor.py:27-100`)
   **Global Features:**
   - `total_scroll_distance`: Sum of all deltaY values
   - `avg_scroll_speed`: Average deltaY / average time_diff
   - `num_scroll_actions`: Count of unique action_ids
   - `avg_time_between_actions`: Mean duration per action
   **Per-Action Features:**
   - `action_scroll_distance`: Sum of deltaY per action_id
   - `action_avg_speed`: Mean deltaY / mean time_diff per action
   - `action_time_taken`: Max timestamp - Min timestamp per action
   - Output: DataFrame with 7 features per action row

4. **ML Prediction** (`scroll_classifier.py:38-58`)
   - XGBoost model inference on feature matrix
   - Calculate probability for each action row
   - Compute mean probability across all actions
   - Threshold at 0.5: > 0.5 = BOT, <= 0.5 = Human
   - Return appropriate label and probability

5. **Response Formatting**
   - Convert prediction to tuple: `(label, probability)`
   - JSON serialization: `["BOT", 0.85]` or `["Human", 0.92]`

6. **Error Handling**
   - Catch all exceptions
   - Log error details
   - Return safe default: `("Human", 1.0)`
   - Prevents false positives blocking legitimate users

---

## Database Interactions

> **None.** The uba-swiping service is a stateless ML inference service.

---

## Additional Information

### Error Handling

**Safe Default Strategy:**

The service implements fail-safe error handling to prevent false positives:

```python
@bentoml.api
async def predict(self, data: pd.DataFrame) -> Tuple[str, float]:
    try:
        preprocess_data = self.pipeline.preprocess(session=data)
        features = self.pipeline.extract_features(preprocess_data)
        label, prob = self.pipeline.predict(features)
        return (label, prob)
    except Exception as e:
        logger.error(e)
        return ("Human", 1.0)
```

### Logging

**Log Levels:**
- `INFO`: Service startup, model loading, prediction statistics
- `WARN`: Unusual patterns, edge cases
- `ERROR`: Feature extraction failures, model errors

**Example Logs:**
```
INFO: ScrollService initialized with 4 workers
INFO: Loaded XGBoost model from checkpoints/xgboost/15_6/dev
INFO: Processing scroll session with 25 touch events, 5 actions
INFO: Prediction: Human (0.82 confidence)
ERROR: Feature extraction failed: Empty DataFrame after filtering touch events
ERROR: Scroll would return Human by default
```

### Troubleshooting

**Issue: Service returns "Human" for all requests**

```bash
# Check model files exist
ls -la swipe_bot_detector/checkpoints/xgboost/15_6/dev/

# Verify model loading
python -c "from swipe_bot_detector import ScrollClassifier; sc = ScrollClassifier(); print('OK')"
```

**Issue: No touch events detected**
- Ensure events have button == "scroll"
- Verify touch events in request data
- Minimum data: 1+ touch events

**Issue: Feature extraction fails**
- Check that deltaY values are populated
- Verify timestamp ordering (should be chronological)
- Ensure at least 1 scroll event exists

**Issue: deltaY normalization incorrect**
- If deltaY values are decimals (0.0 to 1.0), they'll be multiplied by WINDOW_HEIGHT (1080)
- If already in pixels, no multiplication occurs
- Check `WINDOW_HEIGHT` environment variable if needed

---

**Last Updated:** 2025-12-11
