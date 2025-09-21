
---

# **NurseBot v3.0 — Robotics Framework  — Prototype**

---

## **1️⃣ Repository Layout (Framework-only, forkable)**

```
nursebot-framework/
├── framework/
│   ├── __init__.py
│   ├── cli.py                 # CLI: create-app, create-node, run
│   ├── message_bus.py         # Async pub/sub (ZeroMQ / asyncio / multiprocessing)
│   ├── base_node.py           # Node lifecycle + FSM hooks
│   ├── topic_manager.py       # Topic registry, auto-updated (was topics.py)
│   ├── fsm.py                 # Base FSM
│   ├── behavior_tree.py       # Base BT
│   ├── can_layer.py           # CAN abstraction for MCU comms
│   ├── ec2_bridge.py          # Cloud offload (vision/SLAM/LLM)
│   ├── config_manager.py      # Auto-generate/update configs
│   └── utils.py               # Logging, health, monitoring
│
├── setup.py
├── requirements.txt
├── nursebot_launcher.py       # Entrypoint
├── README.md
└── LICENSE
```

✅ **Fixes / Notes:**


* Consider adding **template subnode.py** files in framework for dev onboarding.

---

## **2️⃣ Application Layout (per robot/app)**

```
applications/<app_name>/
├── config/
│   └── node_config.json       # Default nodes & params
├── task_manager.py            # Global BT/FSM orchestrator
├── nodes/
│   ├── <module_name>_node/
│   │   ├── main.py            # Module orchestrator
│   │   ├── subnode1.py        # Worker
│   │   ├── subnode2.py
│   │   └── ...
└── README.md
```

**Example: Voice Module**

```
nodes/voice_node/
├── main.py
├── wakeword_subnode.py
├── asr_subnode.py
├── auth_subnode.py
├── intent_parser_subnode.py
└── tts_subnode.py
```

✅ **Fixes / Notes:**

* Add **mock subnodes** for early stress-testing (prints/logs only).
* Consider including **sensor calibration helpers** for multi-sensor modules.

---

## **3️⃣ Orchestration Flow**

### **Node Level (BaseNode FSM)**

Lifecycle:
`INIT → STANDBY → ACTIVE → ERROR → SHUTDOWN`

* Registers with Message Bus + Topic Manager
* Runs subnodes (via FSM or BT)
* Publishes heartbeats & status

⚠️ **Failure risk / Precautions:**

* Node crashes silently → implement heartbeat monitor in `utils.py`.
* High-throughput nodes (stereo cam / RP LiDAR) → ensure topic queues with **QoS / buffering** to prevent message drops.

---

### **Module Level (Task Manager FSM/BT)**

* Manages **sequence + concurrency** of subnodes
* Handles retries, fallback, timeouts
* E.g., Voice: ASR runs only after Wakeword

⚠️ **Failure risk / Precautions:**

* Deadlocks if subnodes wait indefinitely → enforce **timeouts + watchdogs**.
* Subnode-specific exceptions (e.g., ASR crash mid-stream) → handle gracefully.

---

### **Global Level (Orchestrator BT/FSM)**

* Enforces **inter-module priority**
* E.g., Safety > Voice > Navigation > Manipulation
* Can pause/resume modules dynamically

⚠️ **Failure risk / Precautions:**

* Conflicting commands between modules → define **strict global priority tree**.
* Add logging for state transitions to debug multi-module conflicts.

---

## **4️⃣ Communication Layers**

* **Default:** Async pub/sub (ZeroMQ / asyncio)
* **CAN Layer:** Maps high-level → CAN frames, normalizes MCU feedback
* **EC2 Bridge:** Offloads heavy tasks (vision/SLAM/LLM) to cloud
* **MQTT/gRPC (optional):** For external devices

⚠️ **Failure risk / Precautions:**

* CAN packet loss → add checksum + retries
* EC2 latency spikes → keep **local fallback BT node**
* Ensure **QoS / buffering** for high-throughput nodes

---

## **5️⃣ Fork & Use Workflow**

1. **Fork + install**

```bash
git clone <fork>
cd nursebot-framework
pip install -e .
```

2. **Start app**

```bash
nursebot create-app nursebot
```

3. **Add modules/nodes**

```bash
nursebot create-node nursebot voice_node
nursebot create-node nursebot navigation_node
```

4. **Run app**

```bash
nursebot run nursebot
```

✅ CLI auto-wires topics, configs, installs dependencies.

💡 **Note:** Consider adding future CLI commands: `update-node`, `remove-node`.

---

## **6️⃣ Embedded Integration (CAN)**

* Abstracts MCU comms
* Multiple boards/sensors supported
* Publishes normalized events to framework topics

⚠️ **Failure risk / Precautions:**

* Multiple MCUs spamming bus → use **priority arbitration + rate limiting**.
* Calibration helpers needed for accurate sensor fusion.

---

## **7️⃣ Key Features**

* Modular & forkable (framework vs app)
* Async-first, multi-node orchestration
* FSM + BT hybrid control
* Auto topic wiring & config management
* Embedded + cloud ready
* Health-aware (heartbeats, retries)
* Platform-agnostic (Jetson, Pi, EC2)

---

## **8️⃣ Failure Map & Mitigations**

| Layer          | Risk                             | Mitigation / Notes                                       |
| -------------- | -------------------------------- | -------------------------------------------------------- |
| Node FSM       | Silent crash / hang              | Heartbeats + watchdog timers                             |
| Module FSM/BT  | Deadlock / infinite wait         | Timeouts + fallback branches, subnode exception handling |
| Global BT      | Conflicting priorities           | Strict global priority order, state transition logs      |
| Message Bus    | Bottleneck / lost messages       | ZeroMQ with retries + QoS                                |
| CAN Layer      | Packet corruption / bus overload | Checksums + priority arbitration                         |
| EC2 Bridge     | High latency / cloud outage      | Local fallback tasks                                     |
| Config Manager | Invalid configs break system     | Schema validation + defaults                             |
| CLI            | User misuse                      | Sanity checks + error prompts                            |

---

## **9️⃣ Starter / Stress-Test Workflow**

1. **Mock nodes** → replace real sensors with print/logging-only nodes
2. **Stress-test message bus** → multiple nodes publishing simultaneously
3. **FSM/BT transitions** → test timeout, retries, fallback
4. **CAN simulation** → simulate packet loss, multiple boards
5. **EC2 simulation** → latency spikes, cloud disconnects

✅ After mock testing passes, plug in **real sensors**.



