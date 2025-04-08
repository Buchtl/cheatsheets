To convert all the files of an HTML homepage (like `index.html` and its related assets) into **one PDF** on Linux, here's how you can do it:

## Converting HTML to PDF
---

### ✅ Option 1: **Use `wkhtmltopdf` (Best for full HTML to PDF)**
Install it:
```bash
sudo apt install wkhtmltopdf
```

Then run:
```bash
wkhtmltopdf index.html output.pdf
```

If your homepage includes CSS/JS/images and they are **local and properly linked**, `wkhtmltopdf` will render them in the PDF.

**Optional flags:**
```bash
wkhtmltopdf --enable-local-file-access index.html output.pdf
```

This is **required** if your HTML includes relative links to local files.

---

### ✅ Option 2: **Use a headless browser like Google Chrome**

If you want Chrome's rendering engine:

```bash
google-chrome --headless --disable-gpu --print-to-pdf=output.pdf file:///path/to/index.html
```

Or with Chromium:
```bash
chromium --headless --disable-gpu --print-to-pdf=output.pdf file:///path/to/index.html
```

---

### 🧠 Tip: What if it's a remote homepage?

If the homepage is online, you can use:
```bash
wkhtmltopdf https://example.com output.pdf
```
or
```bash
google-chrome --headless --disable-gpu --print-to-pdf=output.pdf https://example.com
```

---

Need to convert a whole **site** (multiple pages) into one PDF? Let me know and I’ll show how to script that too.