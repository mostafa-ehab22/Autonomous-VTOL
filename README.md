## 🎯 Project Overview
A full-stack autonomous VTOL (Vertical Take-Off and Landing) system combining **onboard embedded flight control** with a **serverless AWS cloud extension** for AI-driven mission decision making.
The system lives in two separate parts:
  
<div align="center">
  
| | ✈️ Onboard System | 🌥️ Cloud Extension |
|---|:---:|:---:|
| 🔧 **What** | Embedded flight control & perception | Serverless AWS mission intelligence |
| 🖥️ **Runs on** | Raspberry Pi + Pixhawk | AWS (eu-central-1) |
| ⚡ **Handles** | Anything time-critical | Anything cognitive |
| 🛠️ **Key tech** | ROS2, ArduPilot, YOLO11 | Bedrock, Step Functions, IoT Core |

</div>

> The two parts are **independent by design**: Onboard system handles everything **time-critical**. Cloud handles everything **cognitive**.

<br>

<div align="center">

# ✈️ Part 1: Autonomous VTOL System 

</div>

<div align="center">
  <img src="docs/onboard_architecture.jpeg" alt="Onboard Architecture" width="90%"/>
</div>

## 🧱 System Architecture

The onboard system is structured into three distinct layers, each with a clear responsibility boundary:
### 1️⃣ Perception Layer (AI & Vision)
- **OpenCV** → Frame capture and preprocessing pipeline
- **YOLO11** → Real-time object detection and classification
- **Roboflow** → Real-world annotated dataset for model training

### 2️⃣ High-Level Logic Layer (Decision Making on Raspberry Pi)
- **Raspberry Pi** → Onboard compute for decision making
- **ROS2 Jazzy** → Middleware for inter-process communication
- **Custom Packages & Nodes** → OOP-designed mission logic modules
- **MAVLink Bridge (Serial/UDP)** → Bidirectional communication with Pixhawk

### 3️⃣ Low-Level Control Layer (Flight Dynamics on Pixhawk RTOS)
- **ArduPilot Firmware (Pixhawk)** → Flight controller running on RTOS
- **EKF3** → Extended Kalman Filter for state estimation (position, velocity, attitude)
- **TECS** → Total Energy Control System for speed and altitude management
- **L1 Controller** → Lateral navigation and path following

### 4️⃣ Validation & Safety (Pre-flight & In-flight Guardrails)
- **SITL Simulation (Linux)** → Software-in-the-loop testing before hardware deployment 
- **Pre-Arm Checks** → Validates sensor health and system readiness before flight 
- **Geofence Failsafe** → Enforces geographic boundaries and triggers RTL on breach 

<br>

<div align="center">
  
# 🌥️ Part 2: Cloud Extension

<div align="center">
  <img src="docs/cloud_architecture.png" alt="Cloud Architecture" width="90%"/>
</div>

</div>

### 🎯 Why a Cloud Extension?

The Raspberry Pi was originally responsible for mission logic, state management, data logging, AND running ROS2 + YOLO11 simultaneously — a heavy compute burden for in-flight hardware.

The cloud extension offloads cognitive and non-time-critical responsibilities to AWS, leaving the Pi focused solely on ROS2 coordination and real-time inference.

<div align="center">

| Responsibility | Before (Pi Only) | After (Cloud Extension) |
|---|:---:|:---:|
| 🧠 Mission decision making | Python scripts on Pi | **AWS Bedrock (AI)** |
| 📋 State & mission logging | Local files / SQLite | **DynamoDB** |
| 🔔 Pilot notifications | Ground Control Station only | **SNS (Mobile/Email)** |
| 🔄 Mission state management | In-memory / local | **Device Shadow (with offline sync)** |
| 📨 Message reliability | None | **SQS + Dead Letter Queue** |

</div>

> ⚠️ **Nothing safety-critical moves to the cloud.** All flight control, YOLO11 & failsafes remain fully onboard.

## 🧱 AWS Cloud Architecture

### Core Pipeline
- **AWS IoT Core** → MQTT ingestion point, Device Shadow for offline sync
- **Amazon SQS** → Mission message queue with Dead Letter Queue after 3 failed retries
- **EventBridge Pipes** → Serverless trigger from SQS to Step Functions
- **AWS Step Functions** → Orchestrates the full mission workflow state machine

### AI & Data
- **Amazon Bedrock** → LLM-powered mission decision making (Safe / Unsafe classification)
- **AWS Lambda** → Data normalization, command dispatch, mission continuation
- **Amazon DynamoDB** → Mission state logs and event history

### Alerting & Feedback
- **Amazon SNS** → Dual-topic alerting: Mission Log Topic (safe) and Alert Topic (unsafe)
- **Device Shadow** → Bidirectional state sync between cloud and VTOL (offline-resilient)

### Observability
- **Amazon CloudWatch** → Monitoring and logs
- **AWS CloudTrail** → API call audit logs
- **AWS X-Ray** → End-to-end distributed tracing
- **IAM** → Least-privilege access control across all services

## 🔄 Mission Workflow Detail

**✅ Safe Path:**
```
Safety Check → SAFE
→ SNS (Log Mission Topic)
→ Mission Notifications (Pilot Mobile/Email)
→ Mission Continuation Lambda (DynamoDB update + Shadow sync)
→ END
```

**❌ Unsafe Path:**
```
Safety Check → UNSAFE
→ AWS Command Lambda (Update Device Shadow → VTOL receives abort command)
→ SNS (Log Alert Topic)
→ Wait State (30 seconds, waitForTaskToken)
→ ACK received via IoT Rule → Verify Acknowledgment Lambda
→ END
```

> The Wait State uses Step Functions' `.waitForTaskToken` callback pattern. The task token is embedded in the command sent to the VTOL. The VTOL acknowledges via MQTT → IoT Rule → `SendTaskSuccess` to resume execution.

## 🌉 Integration: How Onboard Meets Cloud

The bridge between the two systems is a single well-defined interface:

```
ArduPilot (Pixhawk)
    │
    │ MAVLink (Serial/UDP)
    ▼
ROS2 Node (MAVLink Bridge)
    │
    │ MQTT over TLS (Port 8883)
    ▼
AWS IoT Core
    │
    └─► Mission Queue → Step Functions → Bedrock Decision
    └─► Device Shadow ← Command Lambda (mission updates back to VTOL)
```

The ROS2 MAVLink bridge node publishes telemetry to IoT Core and subscribes to Device Shadow delta updates — allowing cloud-originated commands (e.g., abort, reroute) to flow back down to the VTOL seamlessly.

## 📂 Cloud Project Structure

```
cloud/
│
├── iot/                          ⬅️ IoT Core rules & certificate configs
├── lambda/
│   ├── data_normalization/       ⬅️ Telemetry normalization function
│   ├── command/                  ⬅️ Shadow update & abort command dispatch
│   └── mission_continuation/     ⬅️ Safe path state update function
│
├── step_functions/               ⬅️ State machine JSON definition (ASL)
├── bedrock/                      ⬅️ Prompt templates & model configuration
└── infrastructure/               ⬅️ AWS CDK (Python) — deploy full stack
```

## 🚀 Deployment

### Prerequisites
```bash
Python 3.11+
AWS CLI configured (aws configure)
AWS CDK installed (npm install -g aws-cdk)
```

### Deploy Cloud Stack
```bash
cd cloud/infrastructure
pip install -r requirements.txt
cdk bootstrap
cdk deploy
```

---

# 🗺️ Roadmap

### Onboard System
- [x] YOLO11 model training on Roboflow dataset
- [x] ROS2 workspace setup with MAVLink bridge node
- [x] SITL simulation validation
- [x] Hardware integration on Pixhawk + Raspberry Pi
- [x] Geofence and failsafe parameter tuning

### Cloud Extension
- [ ] AWS CDK infrastructure stack
- [ ] Lambda functions (normalization, command, continuation)
- [ ] Step Functions state machine definition
- [ ] Bedrock prompt engineering for safety classification
- [ ] IoT Core rules + Device Shadow integration
- [ ] End-to-end integration test (SITL → Cloud → ACK loop)

## 📖 Documentation

<div align="center">

| Document | Description |
|---|---|
| `docs/onboard_architecture.png` | Full onboard system flowchart |
| `docs/cloud_architecture.png` | AWS cloud architecture diagram |
| `docs/integration_guide.md` | How MAVLink telemetry flows into AWS IoT Core |

</div>

## 🤝 Contributing

This is an active graduation project. Issues and suggestions are welcome - feel free to open an issue for discussion.

---

<div align="center">
  <sub>Built as a graduation project at Faculty of Engineering, Alexandria University, Egypt: Autonomous VTOL × AWS Serverless</sub>
</div>
