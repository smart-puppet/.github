# Smart Puppet

A kid companion robot that **sees**, **moves**, and **talks** on a Jetson Orin — plus an agent tool facade over MQTT.

Four processes share one MQTT bus (Mosquitto). They do not call each other over HTTP in production.

```text
  eyes     see the room          → capture on request; publish robot/nav/scene
  drive    move the base         → robot/drive/cmd | stop | status
  brain    talk with the child   → request capture; Vision: into Gemma
  mcp      tools for the agent   → same MQTT (motion gated)
```

```mermaid
flowchart LR
  brain -->|robot/nav/capture| bus[(Mosquitto)]
  mcp -->|capture_scene| bus
  bus --> eyes
  eyes -->|robot/nav/scene| bus
  brain -->|subscribe| bus
  bus -->|robot/drive/*| drive
  drive --> mcu[MCU motors]
```

## Repositories

| Repo | Role |
|------|------|
| [eyes](https://github.com/smart-puppet/eyes) | Camera, YOLO, metric depth, floor segmentation → traversability scene |
| [drive](https://github.com/smart-puppet/drive) | MQTT ↔ UART ↔ MCU (FIFO, watchdog, estop) |
| [brain](https://github.com/smart-puppet/brain) | Voice: Parakeet STT, llama.cpp Gemma, Piper TTS (Kace) |
| [mcp](https://github.com/smart-puppet/mcp) | Agent MCP tools over the same MQTT bus |
| [docs](https://github.com/smart-puppet/docs) | Architecture and MQTT topic contracts |

Start with **[docs](https://github.com/smart-puppet/docs)** — especially [architecture](https://github.com/smart-puppet/docs/blob/main/architecture.md) and [MQTT contracts](https://github.com/smart-puppet/docs/blob/main/mqtt.md).

## How it fits together

- **eyes** runs perception on demand (debug Capture button or `robot/nav/capture`), then publishes `hint`, sectors, objects, and a compact BEV costmap on `robot/nav/scene`.
- **brain** (when vision MQTT is enabled) requests a capture before each reply and appends the hint to the LLM system prompt so Kace can answer “what’s in front of you?” in kid language. It does not drive yet.
- **drive** owns motion safety on the MCU. Clients publish `robot/drive/cmd` / `stop`; status comes back on `robot/drive/status`.
- **mcp** is the tool facade for the on-robot agent (`capture_scene`, `get_scene`, `drive_stop`, gated `drive_cmd`). Same MQTT contracts as brain. Cursor is for development only.

## Typical bring-up

1. Mosquitto  
2. [drive](https://github.com/smart-puppet/drive) host bridge (UART → MCU)  
3. [eyes](https://github.com/smart-puppet/eyes) (debug web) — listens for capture requests  
4. [brain](https://github.com/smart-puppet/brain) for voice (requests captures)  
5. [mcp](https://github.com/smart-puppet/mcp) when the agent tool server should run  

## Still ahead

- Local planner: costmap → `robot/drive/cmd`  
- Stronger indoor floor segmentation  
- Gemma tool-calling into MCP tools / drive (safely gated)  

Built for Jetson Orin Nano–class hardware. Weights and TensorRT engines are built locally; they are not stored in these repos.
