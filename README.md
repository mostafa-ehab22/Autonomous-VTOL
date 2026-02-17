<h1 align="center">Cognitive Mission Control for Autonomous VTOL</h1>

<div>

![Python](https://img.shields.io/badge/Python-3776AB.svg?logo=python&logoColor=white)
![ROS2](https://img.shields.io/badge/ROS2-22314E.svg?logo=ros&logoColor=white)
![ArduPilot](https://img.shields.io/badge/ArduPilot-FF0000.svg?logo=ardupilot&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-232F3E.svg?logo=amazonaws&logoColor=white)
![Amazon Bedrock](https://img.shields.io/badge/Amazon%20Bedrock-FF9900.svg?logo=amazonaws&logoColor=white)

</div>

## 🎯 Project Overview

A full-stack autonomous VTOL (Vertical Take-Off and Landing) system combining **onboard embedded flight control** with a **serverless AWS cloud extension** for AI-driven mission decision making.

The system is split into two tightly integrated layers:

- **Part 1 — Onboard VTOL System:** Real-time perception, flight control, and ROS2-based decision logic running on embedded hardware (Raspberry Pi + Pixhawk).
- **Part 2 — Cloud Extension (Cognitive Mission Control):** A serverless AWS architecture that offloads mission intelligence, state logging, pilot alerting, and AI decision making from the onboard computer to the cloud.

> The onboard system handles everything **time-critical**. The cloud handles everything **cognitive**.

---

<br>

# Part 1 — Autonomous VTOL System

## 🧱 System Architecture

The onboard system is structured into three distinct layers, each with a clear responsibility boundary:

<div align="center">
  <img src="docs/onboard_architecture.jpeg" alt="Onboard Architecture" width="70%"/>
</div>

## ⚙️ Tech Stack

### Perception Layer
- **OpenCV** → Frame capture and preprocessing pipeline
- **YOLO11** → Real-time object detection and classification
- **Roboflow** → Real-world annotated dataset for model training

### High-Level Logic Layer
- **Raspberry Pi** → Onboard compute for decision making
- **ROS2 Jazzy** → Middleware for inter-process communication
- **Custom Packages & Nodes** → OOP-designed mission logic modules
- **MAVLink Bridge (Serial/UDP)** → Bidirectional communication with Pixhawk

### Low-Level Control Layer
- **ArduPilot Firmware (Pixhawk)** → Flight controller running on RTOS
- **EKF3** → Extended Kalman Filter for state estimation (position, velocity, attitude)
- **TECS** → Total Energy Control System for speed and altitude management
- **L1 Controller** → Lateral navigation and path following

## 🛡️ Validation & Safety

<div alig="center">
  
| Mechanism | Purpose |
|---|---|
| **SITL Simulation (Linux)** | Software-in-the-loop testing before hardware deployment |
| **Pre-Arm Checks** | Validates sensor health and system readiness before flight |
| **Geofence Failsafe** | Enforces geographic boundaries and triggers RTL on breach |

</div>

<br>

# Part 2 — Cloud Extension (Cognitive Mission Control)

## 🎯 Why a Cloud Extension?

The Raspberry Pi was originally responsible for mission logic, state management, data logging, AND running ROS2 + YOLO11 simultaneously — a heavy compute burden for in-flight hardware.

The cloud extension offloads cognitive and non-time-critical responsibilities to AWS, leaving the Pi focused solely on ROS2 coordination and real-time inference.

| Responsibility | Before (Pi Only) | After (Cloud Extension) |
|---|---|---|
| Mission decision making | Python scripts on Pi | **AWS Bedrock (AI)** |
| State & mission logging | Local files / SQLite | **DynamoDB** |
| Pilot notifications | Ground Control Station only | **SNS (Mobile/Email)** |
| Mission state management | In-memory / local | **Device Shadow (with offline sync)** |
| Message reliability | None | **SQS + Dead Letter Queue** |

> ⚠️ **Nothing safety-critical moves to the cloud.** ArduPilot, EKF3, TECS, L1, YOLO11, and all failsafes remain fully onboard.

## 🧱 Cloud Architecture

```
Autonomous VTOL
      │
      │ MQTT (Telemetry)
      ▼
 AWS IoT Core ──────────────────────────► Amazon S3 (Glacier Lifecycle)
      │                                          Archive
      │ Publish
      ▼
 Amazon SQS (Mission Queue)
      │ 3 Failed Tries → Dead Letter Queue (DLQ)
      │ Trigger via EventBridge Pipes
      ▼
 AWS Step Functions (Mission Workflow)
      │
      ├─► Lambda (Data Normalization)
      ├─► Amazon Bedrock (AI Decision Making)
      ├─► DynamoDB (Mission State Logs)
      │
      ▼
  Safety Check
      │
      ├── SAFE ──► SNS (Log Mission Topic)
      │            ► Mission Notifications (Mobile/Email)
      │            ► Mission Continuation Lambda (Update Mission State)
      │
      └── UNSAFE ► AWS Command Lambda (Update Shadow Device)
                   ► SNS (Log Alert Topic)
                   ► Wait State (30s) ◄──── ACK via IoT Rule
                   ► Verify Acknowledgment Lambda
```

## ⚙️ AWS Services Used

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

**Safe Path:**
```
Safety Check → SAFE
→ SNS (Log Mission Topic)
→ Mission Notifications (Pilot Mobile/Email)
→ Mission Continuation Lambda (DynamoDB update + Shadow sync)
→ END
```

**Unsafe Path:**
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

<br>

# 🗺️ Roadmap

### Onboard System
- [ ] YOLO11 model training on Roboflow dataset
- [ ] ROS2 workspace setup with MAVLink bridge node
- [ ] SITL simulation validation
- [ ] Hardware integration on Pixhawk + Raspberry Pi
- [ ] Geofence and failsafe parameter tuning

### Cloud Extension
- [ ] AWS CDK infrastructure stack
- [ ] Lambda functions (normalization, command, continuation)
- [ ] Step Functions state machine definition
- [ ] Bedrock prompt engineering for safety classification
- [ ] IoT Core rules + Device Shadow integration
- [ ] End-to-end integration test (SITL → Cloud → ACK loop)

---

## 📖 Documentation

| Document | Description |
|---|---|
| `docs/onboard_architecture.png` | Full onboard system flowchart |
| `docs/cloud_architecture.png` | AWS cloud architecture diagram |
| `docs/integration_guide.md` | How MAVLink telemetry flows into AWS IoT Core |

---

## 🤝 Contributing

This is an active graduation project. Issues and suggestions are welcome — feel free to open an issue for discussion.

---

<div align="center">
  <sub>Built as a graduation project — Autonomous VTOL × AWS Serverless</sub>
</div>
