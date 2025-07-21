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

