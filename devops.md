# Dropout Prediction Model — DevOps Pipeline


## 1. Project Overview

A **Streamlit web application** that predicts student dropout risk using machine learning. It integrates:

- 🎓 **Dropout risk prediction** (Random Forest model on attendance, grades, infractions)
- 📷 **Face recognition attendance** (OpenCV Haar Cascade)
- 🗄️ **SQLite database** via SQLAlchemy ORM
- 👤 **User authentication** (bcrypt password hashing)
- 📊 **Admin dashboard** with Plotly visualizations

---

## 2. Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend / App | Streamlit |
| ML Model | scikit-learn (Random Forest) |
| Face Recognition | OpenCV (Haar Cascade) |
| Database | SQLite + SQLAlchemy |
| Auth | bcrypt |
| Containerization | Docker (multi-stage) + Docker Compose |
| CI/CD | GitHub Actions |
| Registry | GitHub Container Registry (GHCR) |
| Testing | Pytest |

---

## 3. Project Structure

```
dropout-prediction/
├── app.py                        # Streamlit main entry point
├── app/                          # Core modules
│   ├── main.py                   # Tkinter-based desktop app (legacy)
│   ├── prediction_module.py      # ML prediction logic
│   ├── attendance_manager.py     # Attendance tracking
│   ├── face_recognition_module.py
│   └── user_registration.py
├── pages/                        # Streamlit pages
│   ├── dropout_analyzer.py       # Dropout risk UI
│   ├── admin_dashboard.py        # Admin analytics
│   ├── face_attendance.py        # Face-based attendance
│   ├── student_registration.py
│   └── batch_import.py
├── utils/                        # Utility modules
│   ├── model.py                  # ML model utilities
│   ├── auth.py                   # bcrypt authentication
│   ├── db.py                     # SQLAlchemy engine/session
│   ├── models.py                 # ORM models (User, Student, AttendanceLog)
│   └── face_utils.py             # OpenCV face utilities
├── models/
│   ├── dropout_model.pkl         # Trained RandomForest model
│   └── scaler.pkl                # Feature scaler
├── data/
│   └── dataset.csv               # Training dataset
├── tests/                        # Pytest test suite (16 tests)
│   ├── conftest.py
│   ├── test_auth.py
│   ├── test_db.py
│   ├── test_face_utils.py
│   └── test_model.py
├── Dockerfile                    # Multi-stage Docker build
├── docker-compose.yml            # Service orchestration
├── requirements-docker.txt       # Docker-safe dependencies
└── .github/workflows/ci-cd.yml   # GitHub Actions pipeline
```

---

## 4. Running Locally (without Docker)

```bash
# Install deps (full, with face-recognition + dlib)
pip install -r requirements.txt

# Or Docker-safe deps (no dlib/face-recognition)
pip install -r requirements-docker.txt

# Launch the app
streamlit run app.py
# Visit: http://localhost:8501
```

---

## 5. Docker Setup

### Why Multi-Stage Build?

- **Stage 1 (builder):** Installs all Python packages (large)
- **Stage 2 (runtime):** Copies only the installed packages, not build tools → **smaller image**

Security: app runs as a **non-root user** (`appuser`).

> **Note:** `dlib` and `face-recognition` are excluded from the Docker image (require C++ compilation toolchain, 2GB+ build time). The app uses `opencv-python-headless` for face operations inside Docker.

### Build & Run

```bash
# Build image
docker build -t dropout-prediction .

# Run container
docker run -p 8501:8501 dropout-prediction

# Visit: http://localhost:8501
```

### Docker Compose (recommended)

```bash
# Start with persistent DB volume
docker-compose up -d

# View logs
docker-compose logs -f

# Stop
docker-compose down
```

---

## 6. CI/CD Pipeline (GitHub Actions)

### Trigger

| Event | Trigger |
|-------|---------|
| Push to `main` or `develop` | Runs test + build + deploy |
| Pull Request to `main` | Runs tests only |

### Pipeline Flow

```
Push to main
     │
     ▼
┌──────────────┐
│  Job 1: test │  pytest tests/ -v  (16 tests)
└──────┬───────┘  Runs on every push & PR
       │ ✅ pass
       ▼
┌──────────────┐
│  Job 2:build │  docker build → push to GHCR
└──────┬───────┘  Only on main branch
       │ ✅ pass
       ▼
┌──────────────┐
│  Job 3:deploy│  Pull image → docker-compose up
└──────────────┘  (SSH-deploy ready)
```

### Pipeline File: `.github/workflows/ci-cd.yml`

**Job 1 — Test:**
- Sets up Python 3.11 with pip cache
- Installs `requirements-docker.txt`
- Runs `pytest tests/ -v --tb=short -x`

**Job 2 — Build:**
- Logs into GHCR using `GITHUB_TOKEN` (no secrets needed)
- Tags image as `sha-<commit>` and `latest`
- Pushes to `ghcr.io/<owner>/<repo>:latest`

**Job 3 — Deploy:**
- Simulated by default
- Uncomment `appleboy/ssh-action` block for real SSH deployment

---

## 7. Running Tests

```bash
# With requirements-docker.txt installed
pytest tests/ -v
```

Expected:
```
tests/test_auth.py::test_hash_password              PASSED
tests/test_auth.py::test_verify_password            PASSED
tests/test_auth.py::test_hash_password_special_chars PASSED
tests/test_auth.py::test_hash_password_long         PASSED
tests/test_db.py::test_get_engine                   PASSED
tests/test_db.py::test_init_db                      PASSED
tests/test_db.py::test_get_db_session               PASSED
tests/test_db.py::test_user_model                   PASSED
tests/test_db.py::test_student_model                PASSED
tests/test_db.py::test_student_embedding            PASSED
tests/test_face_utils.py::test_compare_embeddings   PASSED
tests/test_face_utils.py::test_compare_embeddings_normalized PASSED
tests/test_face_utils.py::test_compare_embeddings_opposite   PASSED
tests/test_model.py::test_prepare_features          PASSED
tests/test_model.py::test_get_risk_explanations     PASSED
tests/test_model.py::test_predict_risk_no_model     PASSED

16 passed
```

---

## 8. Key DevOps Concepts Demonstrated

### Containerization
| Concept | Implementation |
|---------|---------------|
| Multi-stage build | Builder + Runtime stages |
| Non-root user | `useradd appuser` + `USER appuser` |
| Headless OpenCV | `opencv-python-headless` (no GUI libs) |
| Health check | `/_stcore/health` endpoint polling |
| Port exposure | `EXPOSE 8501` |

### Orchestration (Docker Compose)
| Concept | Implementation |
|---------|---------------|
| Service definition | Named `app` service |
| Persistent storage | Named volume `dropout_db` |
| Environment vars | `STREAMLIT_SERVER_HEADLESS=true` |
| Restart policy | `unless-stopped` |
| Health monitoring | Built-in healthcheck |

### CI/CD (GitHub Actions)
| Concept | Implementation |
|---------|---------------|
| Automated testing | pytest on every push |
| Conditional jobs | Build only on main |
| Image tagging | SHA + latest |
| Registry push | GHCR via `GITHUB_TOKEN` |
| Deployment | SSH-action ready |

---

## 9. Evidence for CA Submission

Capture screenshots of:
1. ✅ GitHub Actions → all 3 jobs green
2. 🐳 `ghcr.io/<owner>/dropout-prediction` image on GHCR
3. `docker-compose up` output + app at `http://localhost:8501`
4. Pytest output showing 16 passed

---

DevOps Continuous Assessment — LPU CSE
