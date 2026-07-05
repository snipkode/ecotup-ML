# ecotup-ML — Finding Driver Service

Flask-based REST API for finding the nearest driver to a user using the **Haversine formula**, backed by MySQL and containerized with Docker.

---

## Table of Contents

- [Features](#features)
- [API Endpoints](#api-endpoints)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Step-by-Step Deployment](#step-by-step-deployment)
  - [1. Clone Repository](#1-clone-repository)
  - [2. Setup MySQL with Docker](#2-setup-mysql-with-docker)
  - [3. Create Database User](#3-create-database-user)
  - [4. Build & Run the Flask Service](#4-build--run-the-flask-service)
  - [5. Verify the Service](#5-verify-the-service)
- [Issues & Fixes Encountered](#issues--fixes-encountered)
- [Backend Integration](#backend-integration)
- [Project Structure](#project-structure)

---

## Features

- Find nearest driver based on user GPS coordinates (Haversine calculation)
- Clustering and sorting user houses using TSP Greedy Algorithm
- Fully containerized — runs via Docker with zero host dependencies
- Blueprint-based Flask app for modular routing

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/` | Health check |
| `GET` | `/find_nearest_driver/<user_id>` | Find nearest driver for a user |
| `GET` | `/clustering_and_sorting` | Cluster and sort user pickup routes |

### Example Request

```
GET http://43.157.208.51/find_nearest_driver/44
```

### Example Response

```json
{
  "distance": 0.0,
  "driver_latitude": -6.8972838,
  "driver_longitude": 107.5388277,
  "nearest_driver_id": 9,
  "user_id": 44
}
```

---

## Architecture

```
Android App
    │
    ▼
Flask API (Docker) ── port 80 ──► http://43.157.208.51
    │
    │ internal-net (Docker Network)
    ▼
MySQL 8 (Docker) ── database: ecotup
    │
    ▼
phpMyAdmin ── port 8080 ── http://43.157.208.51:8080
```

All services communicate through Docker network `internal-net`. The Flask app resolves MySQL using the container hostname `mysql8`.

---

## Prerequisites

- Docker Engine installed on the server
- Port `80` open for inbound traffic (Flask API)
- Port `8080` open (optional, for phpMyAdmin)
- Port `3306` does NOT need to be public (MySQL stays internal)

---

## Step-by-Step Deployment

### 1. Clone Repository

```bash
git clone https://github.com/snipkode/ecotup-ML.git
cd ecotup-ML
```

---

### 2. Setup MySQL with Docker

If MySQL is not yet running, create the Docker network and start MySQL:

```bash
# Create internal Docker network
docker network create internal-net

# Run MySQL 8 container
docker run -d \
  --name mysql8 \
  --network internal-net \
  --restart unless-stopped \
  -e MYSQL_ROOT_PASSWORD=RootPass123! \
  -e MYSQL_DATABASE=ecotup \
  mysql:8

# Run phpMyAdmin (optional, for DB management)
docker run -d \
  --name phpmyadmin \
  --network internal-net \
  --restart unless-stopped \
  -e PMA_HOST=mysql8 \
  -e PMA_PORT=3306 \
  -p 8080:80 \
  phpmyadmin/phpmyadmin
```

> phpMyAdmin accessible at: `http://<your-server-ip>:8080`  
> Login: `root` / `RootPass123!`

---

### 3. Create Database User

The Flask app connects using user `Ecotup_user`. Create this user with access to the `ecotup` database:

```bash
docker exec mysql8 mysql -uroot -pRootPass123! -e "
CREATE USER IF NOT EXISTS 'Ecotup_user'@'%' IDENTIFIED BY 'ecotup!';
GRANT ALL PRIVILEGES ON ecotup.* TO 'Ecotup_user'@'%';
FLUSH PRIVILEGES;
"
```

> **Note:** The `%` wildcard allows the user to connect from any host within Docker network. MySQL 8 uses `caching_sha2_password` by default — make sure `cryptography` package is in `requirements.txt` (already included).

---

### 4. Build & Run the Flask Service

```bash
cd Feature_python_dockerize

# Build Docker image
docker build -t ecotup-finding-driver:latest .

# Run container on internal-net, expose port 80
docker run -d \
  --name ecotup-finding-driver \
  --restart unless-stopped \
  --network internal-net \
  -p 80:5000 \
  ecotup-finding-driver:latest
```

Key flags:
- `--network internal-net` — connects to the same network as MySQL so it can reach `mysql8` by hostname
- `-p 80:5000` — maps host port 80 to Flask's port 5000 inside the container
- `--restart unless-stopped` — auto-restarts on server reboot

---

### 5. Verify the Service

```bash
# Health check
curl http://localhost/

# Test finding driver (replace 44 with a valid user_id from your DB)
curl http://localhost/find_nearest_driver/44
```

Expected output:
```json
{
  "distance": 0.0,
  "driver_latitude": -6.8972838,
  "driver_longitude": 107.5388277,
  "nearest_driver_id": 9,
  "user_id": 44
}
```

Check running containers:
```bash
docker ps
```

Check service logs:
```bash
docker logs ecotup-finding-driver
```

---

## Issues & Fixes Encountered

### Issue 1: MySQL Connection Timeout (Original Remote DB)

**Problem:**  
The original `finding_driver.py` pointed to a remote MySQL host `34.128.90.76` which was not reachable from the new server:

```
pymysql.err.OperationalError: (2003, "Can't connect to MySQL server on '34.128.90.76' (timed out)")
```

**Fix:**  
A MySQL 8 Docker container (`mysql8`) was already running on this server with the `ecotup` database. Updated the connection string to use the local container hostname instead:

```python
# Before
engine = create_engine('mysql+pymysql://Ecotup_user:ecotup!@34.128.90.76/db_ecotup')

# After
engine = create_engine('mysql+pymysql://Ecotup_user:ecotup!@mysql8/ecotup')
```

---

### Issue 2: MySQL Auth — Missing `cryptography` Package

**Problem:**  
MySQL 8 uses `caching_sha2_password` authentication by default. PyMySQL requires the `cryptography` package to support this auth method:

```
'cryptography' package is required for sha256_password or caching_sha2_password auth methods
```

**Fix:**  
Added `cryptography==41.0.7` to `requirements.txt`:

```
Flask==3.0.0
SQLAlchemy==2.0.23
Werkzeug==3.0.0
pymysql==1.0.3
numpy==1.26.0
scikit-learn==1.3.2
cryptography==41.0.7
```

Then rebuilt the Docker image.

---

### Issue 3: Container Not on Same Docker Network as MySQL

**Problem:**  
Even after updating the hostname to `mysql8`, the Flask container couldn't resolve it because it was on a different Docker network (default `bridge`), not `internal-net` where MySQL runs.

**Fix:**  
Added `--network internal-net` flag when running the Flask container:

```bash
docker run -d \
  --name ecotup-finding-driver \
  --network internal-net \   # ← this was missing
  -p 80:5000 \
  ecotup-finding-driver:latest
```

---

## Backend Integration

To call the Finding Driver API from your backend (Node.js, Spring, etc.):

### HTTP GET Request

```
GET http://43.157.208.51/find_nearest_driver/{user_id}
```

### Example — JavaScript / Node.js (fetch)

```javascript
const userId = 44;
const response = await fetch(`http://43.157.208.51/find_nearest_driver/${userId}`);
const data = await response.json();

console.log(data);
// {
//   user_id: 44,
//   nearest_driver_id: 9,
//   driver_longitude: 107.5388277,
//   driver_latitude: -6.8972838,
//   distance: 0.0   // in kilometers
// }
```

### Example — Kotlin / Android (Retrofit)

```kotlin
interface EcotupApiService {
    @GET("find_nearest_driver/{userId}")
    suspend fun findNearestDriver(
        @Path("userId") userId: Int
    ): FindDriverResponse
}

data class FindDriverResponse(
    @SerializedName("user_id") val userId: Int,
    @SerializedName("nearest_driver_id") val nearestDriverId: Int,
    @SerializedName("driver_longitude") val driverLongitude: Double,
    @SerializedName("driver_latitude") val driverLatitude: Double,
    @SerializedName("distance") val distance: Double
)

// Base URL
val retrofit = Retrofit.Builder()
    .baseUrl("http://43.157.208.51/")
    .addConverterFactory(GsonConverterFactory.create())
    .build()
```

### Example — Python (requests)

```python
import requests

user_id = 44
res = requests.get(f"http://43.157.208.51/find_nearest_driver/{user_id}")
print(res.json())
```

---

## Project Structure

```
ecotup-ML/
├── Feature_python_dockerize/
│   ├── flask_app_capstone.py     # Main Flask app, registers blueprints
│   ├── finding_driver.py         # Finding nearest driver logic + blueprint
│   ├── clustering_and_sort.py    # Clustering & TSP sorting logic + blueprint
│   ├── requirements.txt          # Python dependencies
│   ├── Dockerfile                # Docker build instructions
│   └── .dockerignore
└── Model_Code/
    ├── Trash_Classification_Sigmoid.ipynb
    ├── Trash_Classification_Softmax.ipynb
    └── Clustering_houses_AI.ipynb
```

---

## Database Schema (ecotup)

Tables used by the Finding Driver feature:

**`tbl_user`**
| Column | Type | Description |
|--------|------|-------------|
| `user_id` | INT | Primary key |
| `user_longitude` | DOUBLE | User's GPS longitude |
| `user_latitude` | DOUBLE | User's GPS latitude |

**`tbl_driver`**
| Column | Type | Description |
|--------|------|-------------|
| `driver_id` | INT | Primary key |
| `driver_longitude` | DOUBLE | Driver's GPS longitude |
| `driver_latitude` | DOUBLE | Driver's GPS latitude |

---

## License

MIT
