# Basic Freqtrade Docker Commands: Start, Stop & Download Data

---

### ✅ **Start the Freqtrade bot (detached mode)**
```bash
docker compose up -d
```

### 🛑 **Stop the bot**
```bash
docker compose down
```

---

### 📥 **Download historical data**
```bash
# Example: Download 1h data for all pairs in config.json, for time period between 1/1/2020 to 1/1/2024
docker compose run --rm freqtrade download-data --config userdata/config.json --timeframe 1h --timerange 20200101-20240101
```

---
