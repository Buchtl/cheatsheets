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

Nice! If you want to convert a **whole website** (multiple linked HTML pages) into **one PDF**, you have a few options depending on how the site is structured. Here's a breakdown and a full solution.

---

## 🧩 Option A: If You Have All HTML Files Locally

Let’s say your site folder looks like this:

```
my-site/
├── index.html
├── about.html
├── contact.html
├── css/
└── images/
```

### 🛠 Script to Merge All Pages into One PDF:

```bash
#!/bin/bash

# List of HTML files in desired order
pages=("index.html" "about.html" "contact.html")

# Temp folder for intermediate PDFs
mkdir -p tmp_pdfs

# Convert each HTML to a PDF
for page in "${pages[@]}"; do
    wkhtmltopdf --enable-local-file-access "$page" "tmp_pdfs/${page%.html}.pdf"
done

# Merge all PDFs into one
pdfunite tmp_pdfs/*.pdf full_website.pdf

# Clean up
rm -r tmp_pdfs

echo "✅ Full site saved as full_website.pdf"
```

> 📝 Requires: `wkhtmltopdf` and `pdfunite` (from `poppler-utils`)
```bash
sudo apt install wkhtmltopdf poppler-utils
```

---

## 🌐 Option B: If the Site Is Online

If it’s a live site with several internal links you want to include:

### 💡 Option 1: Manually list pages and use `wkhtmltopdf`
```bash
wkhtmltopdf \
  https://example.com \
  https://example.com/about \
  https://example.com/contact \
  site.pdf
```

### 💡 Option 2: Crawl the website and convert all pages

Install `wget` and `wkhtmltopdf`:
```bash
sudo apt install wget wkhtmltopdf poppler-utils
```

Then:
```bash
# Download full site
wget --mirror --convert-links --adjust-extension --page-requisites --no-parent https://example.com

# Convert HTML files to PDF
cd example.com
find . -name "*.html" | sort > html_pages.txt

mkdir -p tmp_pdfs

while read -r file; do
    out="tmp_pdfs/$(basename "${file%.html}").pdf"
    wkhtmltopdf --enable-local-file-access "$file" "$out"
done < html_pages.txt

# Merge
pdfunite tmp_pdfs/*.pdf full_site.pdf

echo "✅ full_site.pdf created!"
```

---

Would you prefer a GUI tool instead? Or want to automate with Python instead of Bash?
