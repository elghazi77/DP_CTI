# 🚀 Quick Start Guide - ATDRP Platform

## Issue Fixed
- PostgreSQL port conflict resolved (changed from 5432 → 5433)
- Docker compose version warning removed
- All import paths corrected

## Prerequisites Check
```bash
# Check if PostgreSQL is running on port 5432
sudo netstat -tulpn | grep 5432

# If you see PostgreSQL running, that's fine - we're using port 5433 now
```

## Step 1: Start Infrastructure (Docker)

```bash
# Stop any existing containers
sudo docker-compose down

# Start all infrastructure services
sudo docker-compose up -d

# Check status
sudo docker-compose ps

# Watch logs (optional)
sudo docker-compose logs -f
```

Expected output:
```
✔ Container soc-redis      Started
✔ Container soc-postgres   Started  
✔ Container soc-redis-ui   Started
```

## Step 2: Verify Services

```bash
# Check Redis
sudo docker-compose exec redis redis-cli ping
# Should return: PONG

# Check PostgreSQL
sudo docker-compose exec postgres psql -U soc -d socdb -c "SELECT COUNT(*) FROM iocs;"
# Should return: count = 4 (seed IOCs)

# Access Redis UI (optional)
# Open browser: http://localhost:8081
```

## Step 3: Activate Virtual Environment

```bash
source venv/bin/activate
```

## Step 4: Launch the Platform

### Option A: Launch Everything (Recommended for first run)
```bash
python launch.py
```

This starts all services:
- Collector (ingests logs)
- Rule Engine (YAML-based detection)
- ML Engine (anomaly detection)
- CTI Engine (threat intelligence)
- Alert Manager (deduplication & scoring)
- Response Engine (automated actions)
- Dashboard (web UI)
- Attack Simulator (generates test traffic)

### Option B: Launch Without Simulator (Use real logs)
```bash
python launch.py --no-sim
```

### Option C: Dashboard Only
```bash
python launch.py --dashboard
```

### Option D: Manual Component Testing (Advanced)

Terminal 1 - Simulator:
```bash
source venv/bin/activate
python simulator/attack_simulator.py --rate 5 --attack-prob 0.3
```

Terminal 2 - Collector:
```bash
source venv/bin/activate
python collector/collector.py --source redis-queue
```

Terminal 3 - Rule Engine:
```bash
source venv/bin/activate
python detection/rule_engine.py
```

Terminal 4 - Dashboard:
```bash
source venv/bin/activate
python dashboard/app.py
```

## Step 5: Access the Dashboard

Open your browser:
```
http://localhost:5000
```

You should see:
- Real-time alert feed
- Statistics (total alerts, by severity, by engine)
- Top attackers table
- Timeline chart
- Recent alerts table

## Step 6: Watch It Work!

The simulator will generate:
- Normal traffic (web requests, SSH logins)
- Attack scenarios:
  - SSH brute force attacks
  - Port scans
  - Web attacks (SQLi, XSS)
  - Privilege escalation
  - Known malicious IPs

Watch the dashboard light up with detections! 🔥

## Monitoring Commands

```bash
# Watch Redis streams
sudo docker-compose exec redis redis-cli XREAD COUNT 10 STREAMS soc:raw_logs 0

# Watch alerts stream
sudo docker-compose exec redis redis-cli XREAD COUNT 10 STREAMS soc:alerts 0

# Check database alerts
sudo docker-compose exec postgres psql -U soc -d socdb -c "SELECT COUNT(*), severity FROM alerts GROUP BY severity;"

# View recent alerts
sudo docker-compose exec postgres psql -U soc -d socdb -c "SELECT title, severity, source_ip, created_at FROM alerts ORDER BY created_at DESC LIMIT 10;"
```

## Troubleshooting

### Port 5432 already in use
✅ Fixed! We're now using port 5433 for the Docker PostgreSQL

### Permission denied (Docker)
```bash
# Add your user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Or use sudo with docker commands
sudo docker-compose up -d
```

### Redis connection refused
```bash
# Check if Redis is running
sudo docker-compose ps
sudo docker-compose logs redis

# Restart if needed
sudo docker-compose restart redis
```

### Python module not found
```bash
# Make sure virtual environment is activated
source venv/bin/activate

# Reinstall dependencies
pip install -r requirements.txt
```

### No alerts appearing
1. Check if simulator is running: `ps aux | grep attack_simulator`
2. Check Redis streams: `sudo docker-compose exec redis redis-cli XLEN soc:raw_logs`
3. Check detection engines are running: `ps aux | grep engine`
4. Check logs for errors in the terminal where you ran `launch.py`

## Testing Specific Attack Scenarios

```bash
# Run a specific attack once
python simulator/attack_simulator.py --scenario brute_force
python simulator/attack_simulator.py --scenario port_scan
python simulator/attack_simulator.py --scenario web_attack
python simulator/attack_simulator.py --scenario malicious_ip
python simulator/attack_simulator.py --scenario privilege_escalation
```

## Stopping the Platform

```bash
# Stop Python services (Ctrl+C in the terminal running launch.py)

# Stop Docker infrastructure
sudo docker-compose down

# Stop and remove volumes (clean slate)
sudo docker-compose down -v
```

## Configuration

Edit `.env` file to customize:
- Redis/PostgreSQL connection settings
- API keys for AbuseIPDB and VirusTotal (for real CTI)
- Telegram bot token (for notifications)
- Simulator parameters (event rate, attack probability)

## Next Steps

1. **Add Real Log Sources**: Modify `collector/collector.py` to read from:
   - `/var/log/auth.log` (SSH logs)
   - `/var/log/nginx/access.log` (web logs)
   - Syslog socket
   - Kafka/Filebeat

2. **Create Custom Rules**: Add YAML files to `detection/rules/`

3. **Tune ML Models**: Adjust thresholds in `detection/ml_engine.py`

4. **Add CTI Sources**: Get free API keys:
   - AbuseIPDB: https://www.abuseipdb.com/api
   - VirusTotal: https://www.virustotal.com/gui/join-us

5. **Set Up Telegram Notifications**:
   - Create bot: https://t.me/BotFather
   - Get chat ID: https://api.telegram.org/bot<TOKEN>/getUpdates
   - Add to `.env`

## Architecture Overview

```
┌─────────────┐
│   Logs      │ (Auth, Web, Network, Custom)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Collector  │ (Normalize → JSON events)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│Redis Streams│ (Message bus)
└──────┬──────┘
       │
       ├──────────┬──────────┬──────────┐
       ▼          ▼          ▼          ▼
   ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
   │ Rule │  │  ML  │  │ CTI  │  │Alert │
   │Engine│  │Engine│  │Engine│  │ Mgr  │
   └───┬──┘  └───┬──┘  └───┬──┘  └───┬──┘
       │         │         │         │
       └─────────┴─────────┴─────────┘
                    │
                    ▼
            ┌──────────────┐
            │   Response   │ (Block IP, Notify)
            └──────────────┘
                    │
                    ▼
            ┌──────────────┐
            │  Dashboard   │ (Real-time UI)
            └──────────────┘
```

Enjoy your SIEM platform! 🛡️
