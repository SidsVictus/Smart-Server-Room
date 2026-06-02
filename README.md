#  Project Zephyr

An AI-powered intelligent environment controller for a server room.
Zephyr runs a closed-loop agent that monitors indoor temperature, fetches real outdoor weather, and uses an LLM to decide the optimal cooling action every 10 seconds — all visualised on a dashboard.

> **"Zephyr"** is Latin for *Gnetle breeze* — a nod to efficient cooling systems. 
---

## Architecture

![Flow diagram](<img width="1024" height="1536" alt="flow" src="https://github.com/user-attachments/assets/47c58886-b350-4a5d-91ca-90d4560291b9" />
)



1. `scraper.py` — fetches live outdoor weather (BrightData → OpenWeatherMap → static fallback)
2. `environment.py` — applies the last cooling action to a physics-based thermal model, returns new indoor temperature
3. `main.py` — builds a full state vector (temps, humidity, wind, AQI)
4. `controller.py` — sends state + history to Agent (or falls back to a rule engine) to decide the next action
5. `actuator.py` — maps the action string to hardware power values (AC duty cycle, fan speed, watts)
6. `evaluator.py` — scores the outcome (comfort 50% + energy 30% + stability 20%) and returns a reward in [0, 1]
7. `memory.py` — saves state, action, reason, last 5 decisions and reward
8. `ws_server.py` — broadcasts the full payload via WebSocket to the dashboard

---


## Thermal Physics Model

`environment.py` uses a simplified Newton's law of cooling, with the incoming outdoor data:

```
dT/dt = (1/C) × [ Q_servers(traffic) – Q_cooling(action) – Q_leak(T_in, T_out) ]
```
when the outdoor data is compromised, fallback is given by:

```
def _rule_engine(state: dict, target_temp: float) -> dict:
    gap = state["indoor_temp"] - target_temp
    
    if gap > 4.0:
        return {"action": "ac_high", "reason": "Critical: ..."}
    elif gap > 2.0:
        return {"action": "ac_medium", "reason": "Moderate: ..."}
    
```

## Project Structure

```
Zephyr/
├── main.py            # Orchestrator — runs the full agent loop
├── controller.py      # LLM agent decision-making 
├── environment.py     # Physics-based indoor temperature simulation
├── scraper.py         # Outdoor weather fetching (BrightData / OWM / fallback)
├── actuator.py        # Action → hardware power numbers
├── evaluator.py       # Reward scoring
├── memory.py          # Decision history store
├── ws_server.py       # WebSocket broadcast to React dashboard
├── src/               # React + TypeScript frontend dashboard
├── requirements.txt   # Python dependencies
└──package.json       # Node dependencies (Vite + React)
```

---

## Prerequisites

- Python 3.10+
- Node.js 18+

---


**Rule engine fallback** (used when no API key is set or the LLM call fails):

| Condition | Action |
|---|---|
| indoor_temp > target + 4°C | `ac_high` |
| indoor_temp > target + 2°C | `ac_medium` |
| Small gap  | `ac_low` |
| indoor_temp < target − 1.5°C | `off` |
| Otherwise | `fan_only` |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Agent / backend | Python 3.10+ |
| LLM controller | Grok API   |
| Weather primary | BrightData Web Scraper API |
| Weather fallback | OpenWeatherMap |
| Real-time comms | WebSockets (`websockets`) |
| Config | python-dotenv |

---

## License

"This project is licensed under Apache 2.0."


