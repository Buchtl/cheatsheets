
### Options checking a dir for files an check with Prometheus

1. **`node_exporter` + textfile collector (recommended)**

   * Node exporter has a [textfile collector](https://github.com/prometheus/node_exporter#textfile-collector) where you can drop in metrics as plain text.
   * Example script:

     ```bash
     #!/bin/bash
     dir="/path/to/your/dir"
     count=$(find "$dir" -type f | wc -l)

     cat <<EOF > /var/lib/node_exporter/textfile_collector/filecount.prom
     directory_file_count{directory="$dir"} $count
     EOF
     ```
   * Prometheus will scrape it as `directory_file_count{directory="/path/to/your/dir"}`.

2. **Custom exporter**

   * Write a small script (Python, Go, etc.) that:

     * Counts files in a directory.
     * Exposes them via HTTP in Prometheus exposition format.
   * Useful if you want multiple metrics (e.g., subdirectories, file types, sizes).

3. **`node_exporter` filesystem metrics (limited)**

   * Node exporter does have metrics like `node_filesystem_files` and `node_filesystem_files_free`, but those count inodes, **not actual files in one directory**, so usually not what you want.

---

## Implementation

👉 The easiest way:
Use **node\_exporter + textfile collector**, and drop a cron job/systemd timer to update the file count metric every minute.

---

Perfect 👍 we’ll do it with **node\_exporter textfile collector + systemd timer**.

---

## 1. Enable textfile collector in `node_exporter`

Check if your node\_exporter is started with a textfile directory. For example:

```bash
/usr/local/bin/node_exporter \
  --collector.textfile.directory=/var/lib/node_exporter/textfile_collector
```

If you don’t see `--collector.textfile.directory=...` in your systemd unit, edit:

```bash
sudo systemctl edit node_exporter.service
```

and add:

```ini
[Service]
ExecStart=
ExecStart=/usr/local/bin/node_exporter \
  --collector.textfile.directory=/var/lib/node_exporter/textfile_collector
```

Then reload + restart:

```bash
sudo systemctl daemon-reexec
sudo systemctl restart node_exporter
```

Create the directory:

```bash
sudo mkdir -p /var/lib/node_exporter/textfile_collector
sudo chown node_exporter:node_exporter /var/lib/node_exporter/textfile_collector
```

---

## 2. File count script

Create `/usr/local/bin/export_filecount.sh`:

```bash
#!/bin/bash
set -e

DIR="/path/to/your/dir"
OUT="/var/lib/node_exporter/textfile_collector/filecount.prom"

COUNT=$(find "$DIR" -type f | wc -l)

cat <<EOF > "$OUT"
# HELP directory_file_count Number of files in directory
# TYPE directory_file_count gauge
directory_file_count{directory="$DIR"} $COUNT
EOF
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/export_filecount.sh
```

---

## 3. Systemd service + timer

**Service file** `/etc/systemd/system/export_filecount.service`:

```ini
[Unit]
Description=Export file count to node_exporter textfile

[Service]
Type=oneshot
ExecStart=/usr/local/bin/export_filecount.sh
```

**Timer file** `/etc/systemd/system/export_filecount.timer`:

```ini
[Unit]
Description=Run export_filecount script every minute

[Timer]
OnCalendar=*:0/1
Persistent=true

[Install]
WantedBy=timers.target
```

Enable + start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now export_filecount.timer
```

---

## 4. Check metrics

After 1–2 minutes, open:

```
http://YOUR_HOST:9100/metrics
```

Search for:

```
directory_file_count{directory="/path/to/your/dir"}
```

---

✅ Now Prometheus can scrape this metric like any other.

---

## Here’s a ready-to-import **Grafana panel JSON** that shows the file count as a time series (you can also switch it to a single-stat if you prefer).

---

### Grafana Panel JSON

```json
{
  "id": null,
  "gridPos": {
    "h": 8,
    "w": 12,
    "x": 0,
    "y": 0
  },
  "type": "timeseries",
  "title": "Directory File Count",
  "targets": [
    {
      "expr": "directory_file_count{directory=\"/path/to/your/dir\"}",
      "legendFormat": "{{directory}}",
      "refId": "A"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "none",
      "decimals": 0,
      "color": {
        "mode": "palette-classic"
      },
      "custom": {
        "lineWidth": 2,
        "fillOpacity": 20
      }
    },
    "overrides": []
  },
  "options": {
    "legend": {
      "showLegend": true,
      "displayMode": "list"
    },
    "tooltip": {
      "mode": "single",
      "sort": "none"
    }
  }
}
```

---

### Import into Grafana

1. Go to your Grafana dashboard.
2. Click **Add panel → Import panel JSON**.
3. Paste the JSON above.
4. Replace `/path/to/your/dir` in the query with the exact directory you used in the script.

---

Here’s a **Grafana Stat panel JSON** you can import directly.

---

### Grafana Stat Panel JSON

```json
{
  "id": null,
  "gridPos": {
    "h": 6,
    "w": 6,
    "x": 0,
    "y": 0
  },
  "type": "stat",
  "title": "Directory File Count",
  "targets": [
    {
      "expr": "directory_file_count{directory=\"/path/to/your/dir\"}",
      "legendFormat": "{{directory}}",
      "refId": "A"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "none",
      "decimals": 0,
      "color": {
        "mode": "thresholds"
      },
      "thresholds": {
        "mode": "absolute",
        "steps": [
          {
            "color": "green",
            "value": null
          },
          {
            "color": "orange",
            "value": 1000
          },
          {
            "color": "red",
            "value": 5000
          }
        ]
      }
    },
    "overrides": []
  },
  "options": {
    "reduceOptions": {
      "calcs": ["lastNotNull"],
      "fields": "",
      "values": false
    },
    "orientation": "auto",
    "textMode": "value",
    "colorMode": "background",
    "graphMode": "none",
    "justifyMode": "auto"
  }
}
```

---

### How to use

1. In Grafana, go to **Dashboards → Add panel → Import JSON**.
2. Paste the JSON above.
3. Change `/path/to/your/dir` in the `expr` to match your actual directory.
4. Save the dashboard.

👉 The panel will show a **big number** (latest file count) with background color thresholds (green/orange/red depending on file count).

---
