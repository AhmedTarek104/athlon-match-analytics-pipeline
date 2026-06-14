# Egyptian U-21 National Team — Match Analytics ETL Pipeline

> **Built during my role as Data Analyst/Engineer at ATHLON Sports Analytics (Cairo, Egypt)**

Production data engineering pipeline that processes computer vision 
tracking outputs from Egyptian U-21 National Team matches and uploads 
structured match analytics to Firebase Firestore for real-time 
consumption by the coaching staff dashboard.

**This code ran on real Egyptian U-21 National Team matches.**

---

## Background

ATHLON Sports Analytics (now rebranded as REE) provided data support 
to the Egyptian U-21 National Team coaching staff. Match footage was 
processed through computer vision models to extract player tracking 
events — passes, duels, shots, goals, set pieces, and more.

This pipeline is the bridge between raw CV output and the coaches' 
live dashboard. It transforms messy tracking CSVs into clean, 
structured Firestore documents that the coaching app consumes 
in real time during and after matches.

---

## What This Pipeline Does

### Module 1 — Event Processing (firebase_upload.py)

Processes 10 distinct event types extracted from computer vision:

| Event Type | Key Fields Extracted |
|------------|---------------------|
| Passes | Start/end timestamp, position (x,y), shirt number, success flag, team |
| Aerial Duels | Position, success/failure, shirt number, team, timestamp |
| Ground Duels | Position, success/failure, shirt number, team, timestamp |
| Interceptions | Ball recovery/loss flag, position, team |
| Tackles | Success/failure, position, shirt number, team |
| Dribbles | Success/failure, position, team (deduplicates "dribbled past" events) |
| Set Pieces | Type classification (corner/throw/direct FK/indirect FK/penalty), position, team |
| Goals | Scorer shirt number, position, type (goal vs assist), team |
| Shots | On target/off target/saved, position, shirt number, team |
| Pass Networks | Sequential pass strings, position clusters, pass count per sequence |

**Firestore structure:**
match_analytics/

{match_id}/

distribution/

passes/         ← all pass events

pass_network/   ← passing sequences

duels/

aerial/

ground/

interceptions/

tackles/

dribbles/

set_pieces/

goals/

shots/

### Module 2 — Match Summary (metrics.py)

Aggregates all event-level data into match-level KPIs per team:

| KPI | How Calculated |
|-----|---------------|
| Ball Possession % | Successful pass count per team / total successful passes |
| Total Shots | Shot events per team |
| Shots on Target | Filtered by Shots_on_Target flag |
| Shots Saved | Filtered by Saved_shots flag |
| Total Goals | Goal events per team |
| Corners | Set pieces where corner == 1 |
| Free Kicks | Set pieces where direct/indirect kick == 1 |
| Tackles | Tackle events per team |
| Active Shirt Numbers | Unique shirt numbers per team from pass data |
| Successful Passes | Filtered by success == 1 AND fail == 0 |

**Firestore structure:**
matches/

{match_id}/

team1_id

team2_id

ball_possession/

corners/

free_kicks/

passes/

shots_on_target/

shots_saved/

tackles/

total_goals/

total_shots/

shirt_numbers/

---

## Technical Architecture
Match Footage

↓

Computer Vision Models

↓

Raw Tracking CSVs (10 event types)

↓

Python ETL Pipeline

├── Position parsing (string tuple → float coordinates)

├── Dynamic team ID mapping (handles CV numbering inconsistencies)

├── Event classification (success/failure flags per event type)

├── Set piece type detection (5 types)

├── Pass network sequence reconstruction

└── Firestore document structuring

↓

Firebase Firestore (real-time NoSQL)

↓

Coaching Staff Dashboard

---

## Key Engineering Decisions

**Dynamic team mapping**
Computer vision assigns team IDs inconsistently across event types 
(0/1 in some CSVs, 1/2 in others). The pipeline handles both 
conventions dynamically and maps to actual team identifiers 
(e.g., "VVC#214", "DRC#334").

**Safe Firebase initialization**
Prevents duplicate app initialization errors when running across 
multiple Colab sessions using `if not firebase_admin._apps` guard.

**Position parsing**
CV outputs positions as string tuples `"(x, y)"`. Pipeline safely 
evaluates and splits into typed float coordinates for Firestore.

**Dribble deduplication**
Excludes rows where `dribbled_past == 1` to avoid double-counting 
dribble attempts from the defender's perspective.

**Zero-safe aggregation**
All stat lookups use `.get(team, 0)` fallback — prevents KeyErrors 
when a team has no events of a given type in a match.

**Timestamp safety**
All timestamps use `max(0, int(value))` to prevent negative frame 
numbers from CV edge cases crashing the upload.

---

## Data Flow Example

**Input** (passes_metrics.csv):
start_frame_num, end_frame_num, position,    shirt_num1, team_id_1, success, fail

1240,            1285,          "(45.2,32.1)", 7,         1,         1,       0

**Output** (Firestore document under `match_analytics/{match_id}/distribution/passes/0`):
```json
{
  "start_timestamp": 1240,
  "end_timestamp": 1285,
  "position_x": 45.2,
  "position_y": 32.1,
  "shirt_number": 7,
  "success": true,
  "team_id": "VVC#214"
}
```

---

## Tech Stack

| Tool | Purpose |
|------|---------|
| Python 3 | Core pipeline language |
| pandas | CSV ingestion and transformation |
| numpy | Numerical operations and inf/nan handling |
| Firebase Admin SDK | Firestore authentication and document upload |
| Google Firestore | Real-time NoSQL document database |
| Google Colab | Execution environment |

---

## Files

| File | Description |
|------|-------------|
| `firebase_upload.py` | Main ETL — processes all 10 event types and uploads to Firestore |
| `metrics.py` | Match summary aggregation — computes and uploads match-level KPIs |
| `README.md` | This file |

*Raw tracking CSVs and Firebase service account credentials 
are not included for confidentiality reasons.*

---

## Professional Context

| | |
|---|---|
| **Company** | ATHLON Sports Analytics (Cairo, Egypt), now rebranded as REE |
| **Role** | Data Analyst / Data Engineer |
| **Client** | Egyptian U-21 National Football Team |
| **Deliverable** | Real-time match analytics pipeline feeding coaching staff dashboard |
| **Season** | 2024-25 |

This was part of a broader data infrastructure supporting the 
coaching staff's tactical analysis workflow — enabling real-time 
access to match events, possession stats, and player-level metrics 
during and after games.

---

## Related Projects

- [ScoutEdge v2](https://github.com/AhmedTarek104/scoutedge-v2) — 
  Football recruitment intelligence platform for Al Ahly SC
