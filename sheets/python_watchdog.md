Here's a Python service that watches the `/data/input` directory for new `.xml` files, extracts the `<Filename>` and base64-encoded `<Body>` from each, decodes the body, and writes the result to a file named according to the `<Filename>` value.

---

### ✅ Features:

* Monitors the directory continuously.
* Supports nested `<Body>` tags.
* Base64 decodes the content and writes to disk.
* Deletes processed `.xml` files.

---

### 📦 Requirements

Install the required libraries (if not already installed):

```bash
pip install watchdog
```

---

### 🐍 Python Service Code

```python
import os
import time
import base64
import xml.etree.ElementTree as ET
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

INPUT_DIR = "/data/input"

class XMLHandler(FileSystemEventHandler):
    def on_created(self, event):
        if not event.is_directory and event.src_path.endswith('.xml'):
            try:
                print(f"[INFO] Processing file: {event.src_path}")
                self.process_file(event.src_path)
            except Exception as e:
                print(f"[ERROR] Failed to process {event.src_path}: {e}")

    def process_file(self, file_path):
        tree = ET.parse(file_path)
        root = tree.getroot()

        # Extract <Filename> content
        filename_elem = root.find(".//Filename")
        body_elem = root.find(".//Body")

        if filename_elem is None or body_elem is None:
            raise ValueError("XML missing <Filename> or <Body> element")

        filename = filename_elem.text.strip()
        base64_data = body_elem.text.strip()

        # Decode and write to file
        decoded_data = base64.b64decode(base64_data)
        output_path = os.path.join(INPUT_DIR, filename)

        with open(output_path, 'wb') as out_file:
            out_file.write(decoded_data)
            print(f"[INFO] Wrote decoded file to: {output_path}")

        # Optionally remove the processed XML
        os.remove(file_path)
        print(f"[INFO] Removed processed XML: {file_path}")

def start_service():
    print(f"[INFO] Monitoring directory: {INPUT_DIR}")
    event_handler = XMLHandler()
    observer = Observer()
    observer.schedule(event_handler, INPUT_DIR, recursive=False)
    observer.start()

    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        print("[INFO] Stopping service...")
        observer.stop()
    observer.join()

if __name__ == "__main__":
    start_service()
```

---

### 🛠️ Notes

* Ensure the XML files are fully written before they’re processed to avoid partial reads. You could enhance this by checking file size stability or using a temporary upload pattern (e.g., `.xml.part` → `.xml`).
* The XML must contain exactly one `<Filename>` and one `<Body>`. Let me know if multiple files per XML are needed.


On **Ubuntu**, you can create a **standalone executable** (not a `.exe`, which is for Windows) from your Python script using a tool like [`PyInstaller`](https://pyinstaller.org/), which bundles the Python interpreter and dependencies into a single executable.

---

#Absolutely! Here's a complete guide to setting up your `xml-watcher` Python script as a **`systemd` service** on Ubuntu, assuming you want it to run as a **standalone binary** (built using PyInstaller) and automatically monitor `/data/input`.

---

## ✅ Overview

You will:

1. Convert the Python script to a binary using PyInstaller.
2. Move the binary to a system path.
3. Create a `systemd` service to run it on boot.
4. Enable and manage the service.

---

## ✅ Step-by-Step Instructions

---

### 🧱 1. Build the Executable (Binary) with PyInstaller

**Install PyInstaller** (if not done):

```bash
pip install pyinstaller
```

**Navigate to your script directory** and run:

```bash
pyinstaller --onefile xml_watcher.py
```

It creates:

* `dist/xml_watcher` ← the binary you want
* `build/`, `__pycache__`, etc. ← can be deleted

---

### 📂 2. Install the Binary to System Path

Copy it to `/usr/local/bin`:

```bash
sudo cp dist/xml_watcher /usr/local/bin/xml-watcher
sudo chmod +x /usr/local/bin/xml-watcher
```

You can now test it:

```bash
xml-watcher
```

---

### 🛠️ 3. Create the `systemd` Service File

Create the file:

```bash
sudo nano /etc/systemd/system/xml-watcher.service
```

Paste this:

```ini
[Unit]
Description=XML Watcher Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/xml-watcher
Restart=on-failure
RestartSec=3
User=root
Group=root
WorkingDirectory=/data/input
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

> ✅ **Important**:
>
> * `User` and `Group` should match who owns `/data/input`. Change `root` if needed.
> * If you want logs written to a file instead, use `StandardOutput=file:/var/log/xml-watcher.log`.

---

### 🚀 4. Reload and Start the Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable xml-watcher
sudo systemctl start xml-watcher
```

---

### 🧪 5. Check the Logs

```bash
journalctl -u xml-watcher -f
```

> Press `Ctrl+C` to stop following logs.

---

### 🧹 6. Optional Clean Up

If you're done with PyInstaller outputs:

```bash
rm -rf build/ dist/ __pycache__/ xml_watcher.spec
```

---

## ✅ Summary

| Task           | Command                                             |
| -------------- | --------------------------------------------------- |
| Build binary   | `pyinstaller --onefile xml_watcher.py`              |
| Install binary | `sudo cp dist/xml_watcher /usr/local/bin/`          |
| Create service | `sudo nano /etc/systemd/system/xml-watcher.service` |
| Enable + Start | `sudo systemctl enable --now xml-watcher`           |
| Check logs     | `journalctl -u xml-watcher -f`                      |

---

Would you like me to give you a sample XML file for testing?


Here’s a **sample XML file** you can use to test your `xml-watcher` service. This XML includes a `<Filename>` and a `<Body>` tag with base64-encoded content.

---

## 📄 Sample XML: `sample_input.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Envelope>
    <Header>
        <Timestamp>2025-07-21T12:00:00Z</Timestamp>
    </Header>
    <Body>
        VGhpcyBpcyBhIHRlc3QgZmlsZSBjb250ZW50Lg==
    </Body>
    <Filename>example.txt</Filename>
</Envelope>
```

---

## 📦 What's Inside?

* `<Body>` contains this base64-encoded string:
  `VGhpcyBpcyBhIHRlc3QgZmlsZSBjb250ZW50Lg==`

* Which decodes to:
  `This is a test file content.`

* `<Filename>` is:
  `example.txt`

So after the watcher processes this file, it should create:

```
/data/input/example.txt
```

With the content:

```
This is a test file content.
```

---

## 🧪 How to Test

1. Place this file in the monitored directory:

   ```bash
   cp sample_input.xml /data/input/test_file.xml
   ```

2. Watch the logs:

   ```bash
   journalctl -u xml-watcher -f
   ```

3. Confirm the output file is created:

   ```bash
   cat /data/input/example.txt
   ```

---

Would you like support for multiple `<Body>` / `<Filename>` pairs in a single XML file?



# Here's how to run the Python watcher as a **systemd service** and also how to **containerize it using Docker**.

---

## ✅ Option 1: Run as a `systemd` Service (on Linux)

### 1. **Save the Python Script**

Save the Python script as `/opt/xml_watcher/xml_watcher.py`:

```bash
sudo mkdir -p /opt/xml_watcher
sudo nano /opt/xml_watcher/xml_watcher.py
```

Paste the code from the previous message, save, and exit.

Make it executable:

```bash
sudo chmod +x /opt/xml_watcher/xml_watcher.py
```

---

### 2. **Create the `systemd` Service File**

```bash
sudo nano /etc/systemd/system/xml-watcher.service
```

Paste:

```ini
[Unit]
Description=XML Directory Watcher Service
After=network.target

[Service]
ExecStart=/usr/bin/python3 /opt/xml_watcher/xml_watcher.py
WorkingDirectory=/opt/xml_watcher
Restart=always
User=root
Group=root

[Install]
WantedBy=multi-user.target
```

---

### 3. **Start and Enable the Service**

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable xml-watcher
sudo systemctl start xml-watcher
```

Check logs:

```bash
journalctl -u xml-watcher -f
```

---

## ✅ Option 2: Run as a Docker Container

### 1. **Create the Project Structure**

```bash
mkdir xml_watcher_docker
cd xml_watcher_docker
```

Create these files:

* `xml_watcher.py` → Paste the script
* `Dockerfile`
* `docker-compose.yml` (optional, for easy volume mount)

---

### 2. **Dockerfile**

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY xml_watcher.py .

RUN pip install watchdog

CMD ["python", "xml_watcher.py"]
```

---

### 3. **docker-compose.yml** (optional)

```yaml
version: '3.8'

services:
  xml-watcher:
    build: .
    volumes:
      - /data/input:/data/input:rw
    restart: always
```

---

### 4. **Build and Run**

Using Docker Compose:

```bash
docker compose build
docker compose up -d
```

Or plain Docker:

```bash
docker build -t xml-watcher .
docker run -d --name xml-watcher -v /data/input:/data/input xml-watcher
```

---

## ✅ Summary

| Option    | Use When...                                      |
| --------- | ------------------------------------------------ |
| `systemd` | You want it as a native service on Linux.        |
| Docker    | You want portability, isolation, or CI/CD usage. |

---

Would you like this to support multiple `<Body>`/`<Filename>` pairs per XML?
