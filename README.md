 Technology Stack**

React Native (Frontend)
Flask (Backend)
OpenCV (Computer Vision)
SQLAlchemy (Database)
Firebase Cloud Messaging (Push Notifications)

---
 Impact & Benefits**

* **Improves teaching quality** by ensuring accountability.
* **Enhances student safety** through real-time monitoring.
* **Reduces incidents** of bullying, theft, or violence.
* **Creates proof** for disciplinary actions.
* **Improves school reputation** as a safe and disciplined environment.

---
 Workflow

 got it — here’s a **low-level, build-ready plan** for the AI CCTV classroom alerts system (Team Guardians). It’s the shortest path to a working MVP you can demo.

# Core pipeline (edge device per classroom)

1. **Input**

   * RTSP stream from CCTV: `rtsp://user:pass@cam-ip/stream1`
2. **Frames → Inference**

   * `ffmpeg` → `OpenCV` grab frames @ 10 FPS.
   * **Models**

     * Person/role: YOLOv8n (custom classes: `teacher`, `student`).
     * Action: lightweight action head (MobileNet-GRU) over 16-frame clips.
   * **Outputs (per frame/clip)**: bbox, class, conf, action, track\_id (DeepSORT).
3. **Event logic**

   * `teacher_present`: teacher count ≥1 for ≥ N seconds.
   * `teacher_absent`: teacher count == 0 for ≥ T=180s during scheduled period.
   * `bullying`: action==`fight` OR `pushing` with ≥2 students and IoU overlap ≥ τ.
   * `theft`: `grabbing_object` + object not returned within Δt.
   * `inappropriate_teacher`: action in {`phone_use_long`, `sleeping`, `aggressive_shout`} for ≥ K seconds.
4. **Alerting**

   * Debounce with sliding window (e.g., 30 s).
   * Publish MQTT + REST to server; snapshot JPEG and 10 s clip.
5. **Storage**

   * Local ring buffer (`/var/lib/guardians/buffer/`) 24 h; offload only clips tied to alerts.

# Processes & services

```
guardians-capture.service     # ffmpeg/opencv capture
guardians-infer.service       # yolo + action head
guardians-events.service      # rules engine
guardians-uploader.service    # MQTT/REST + retries
```

# Topics, payloads, and endpoints

**MQTT topics**

* `guardians/{school_id}/{class_id}/alerts`
* `guardians/{school_id}/{class_id}/health`

**Alert JSON (example)**

```json
{
  "school_id":"S001",
  "class_id":"IX-B",
  "ts":"2025-08-14T10:22:31Z",
  "type":"teacher_absent",
  "score":0.92,
  "duration_sec":210,
  "clip_url":"s3://.../S001/IX-B/2025-08-14_102231.mp4",
  "snapshot_url":"s3://.../frame_102231.jpg",
  "track_ids":[12],
  "meta":{"period":"10:00-10:45","timetable_id":5531}
}
```

**Server REST**

* `POST /api/v1/alerts` (ingest)
* `GET  /api/v1/alerts?class_id=&from=&to=`
* `GET  /api/v1/health`
* `POST /api/v1/acknowledge` (admin acknowledges alert)

# MySQL schema (minimal)

```sql
CREATE TABLE classrooms(
  id INT PK AUTO_INCREMENT, school_id VARCHAR(12), name VARCHAR(32),
  rtsp_url TEXT, timezone VARCHAR(32)
);

CREATE TABLE timetable(
  id INT PK AUTO_INCREMENT, classroom_id INT, teacher_id INT,
  subject VARCHAR(32), start_ts DATETIME, end_ts DATETIME, INDEX(classroom_id)
);

CREATE TABLE alerts(
  id BIGINT PK AUTO_INCREMENT, classroom_id INT, type VARCHAR(32),
  score DECIMAL(4,2), duration_sec INT, ts DATETIME,
  clip_url TEXT, snapshot_url TEXT, timetable_id INT,
  status ENUM('new','ack','dismissed') DEFAULT 'new',
  INDEX(classroom_id, ts), INDEX(type, ts)
);

CREATE TABLE teachers(
  id INT PK AUTO_INCREMENT, name VARCHAR(64), email VARCHAR(128), phone VARCHAR(20)
);
```

# File & config layout

```
/opt/guardians/
  config.yaml             # rtsp, thresholds, mqtt, api keys
  models/yolov8n.engine   # TRT/ONNX
  models/action_head.onnx
  rules.yaml              # event thresholds
  logs/
  buffer/                 # cyclical mp4/jpg
```

**config.yaml (snippet)**

```yaml
rtsp: "rtsp://user:pass@192.168.1.20/stream1"
fps: 10
thresholds:
  presence_secs: 30
  absence_secs: 180
  bullying_score: 0.85
mqtt:
  host: "broker.local"
  topic: "guardians/S001/IX-B/alerts"
server:
  base_url: "https://api.guardians.school"
  token: "ENV:API_TOKEN"
```

# Minimal code skeletons

**Frame grab + inference (Python)**

```python
cap = cv2.VideoCapture(RTSP)
tracker = DeepSort()
while True:
    ok, frame = cap.read()
    if not ok: continue
    dets = yolo(frame)                  # [x1,y1,x2,y2,cls,conf]
    tracks = tracker.update(dets)
    clipbuf.push(frame)                 # 16 frames rolling
    if clipbuf.ready():
        act = action_head(clipbuf.tensor())  # class, prob
        event_bus.push(frame, tracks, act)
```

**Event rule (teacher\_absent)**

```python
if timetable.active(now) and teacher_count == 0:
    absent_secs += dt
else:
    absent_secs = 0
if absent_secs >= cfg.absence_secs:
    raise_alert("teacher_absent", score=0.9, duration=absent_secs)
```

# Deployment quickstart (Ubuntu edge)

```bash
sudo apt install ffmpeg python3-opencv python3-venv mosquitto-clients
python3 -m venv venv && source venv/bin/activate
pip install ultralytics onnxruntime lap trackpy paho-mqtt requests
sudo systemctl enable guardians-*.service && sudo systemctl start guardians-capture
```

# Admin dashboard (must-have)

* **Views**: Live alerts, timeline, classroom presence heatmap, teacher absence leaderboard, bullying incidents.
* **Filters**: school, class, alert type, time range, status.
* **Actions**: acknowledge, add note, export clip.

# Test plan

* Replay labeled RTSP files (day, recess, empty class).
* Unit tests for rules with synthetic events.
* Shadow deployment with “alert-only” mode (no actions) for a week; tune thresholds.

# Privacy & safety (MVP)

* Process on-prem where possible.
* Blur faces in stored snapshots by default.
* RBAC: principal, counselor, IT admin.
* Retention: 30 days for alerts, 24 h for raw buffer.

If you want, I can generate **sample alerts + snapshots** and a **mock dashboard JSON** you can plug into a demo UI.

