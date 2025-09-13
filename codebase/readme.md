# Offline OCR Solutions for Scanned PDFs (Windows-Friendly)

This guide shows how to set up and run OCR pipelines for scanned PDFs **offline** on **Windows OS**, with different tradeoffs between speed, accuracy, and complexity.

---

## 1. Tesseract + OCRmyPDF (Simple, CPU)

Great for converting scanned PDFs into **searchable PDFs** with text overlay. Runs fully on CPU.

### Installation (Windows)
1. Install **Tesseract OCR application**:
   - Download from GitHub (Windows binaries maintained by UB Mannheim):  
     👉 https://github.com/UB-Mannheim/tesseract/wiki
   - Install to default path: `C:\Program Files\Tesseract-OCR`.
   - Add this path to your **System PATH** or configure it in Python.

2. Install Python packages:
```bash
pip install ocrmypdf pillow
```

### Example (Python)
```python
import ocrmypdf

# Run OCR on input PDF, save to output
ocrmypdf.ocr("input.pdf", "output.pdf", language="eng", deskew=True)
```

If you use **pytesseract** directly:
```python
import pytesseract
from PIL import Image

# Explicitly set path if not in PATH
tesseract_cmd = r"C:\\Program Files\\Tesseract-OCR\\tesseract.exe"
pytesseract.pytesseract.tesseract_cmd = tesseract_cmd

text = pytesseract.image_to_string(Image.open("scan.png"))
print(text)
```

---

## 2. EasyOCR (GPU optional)

Lightweight PyTorch-based OCR. Runs on **CPU or GPU**. No external applications required.

### Installation (Windows)
```bash
pip install easyocr pdf2image Pillow

# Required for pdf2image to work:
# Download and install Poppler for Windows
# 👉 https://github.com/oschwartz10612/poppler-windows/releases/
# Add bin/ folder to your PATH (e.g. C:\poppler-xx\bin)
```

### Example (Python)
```python
from pdf2image import convert_from_path
import easyocr

# Convert PDF to images
pages = convert_from_path("input.pdf", dpi=300)
reader = easyocr.Reader(['en'], gpu=True)  # set gpu=False if no GPU

for i, page in enumerate(pages):
    image_path = f"page_{i}.png"
    page.save(image_path, "PNG")
    results = reader.readtext(image_path)
    
    print(f"--- Page {i+1} ---")
    for bbox, text, confidence in results:
        print(f"{text} (conf: {confidence:.2f})")
```

---

## 3. PaddleOCR (GPU recommended)

Most accurate & fast OCR for offline usage. Works best with GPU acceleration. No external applications needed (just Python packages).

### Installation (Windows)
```bash
pip install "paddleocr>=2.0.1" pdf2image Pillow

# For CPU-only:
pip install paddlepaddle

# For GPU (CUDA):
pip install paddlepaddle-gpu

# Poppler (required for pdf2image):
# 👉 https://github.com/oschwartz10612/poppler-windows/releases/
# Add bin/ folder to your PATH (e.g. C:\poppler-xx\bin)
```

### Example (Python)
```python
from pdf2image import convert_from_path
from paddleocr import PaddleOCR

# Convert PDF to images
pages = convert_from_path("input.pdf", dpi=300)

ocr = PaddleOCR(use_angle_cls=True, lang='en', use_gpu=True)  # set use_gpu=False if no GPU

for i, page in enumerate(pages):
    image_path = f"page_{i}.png"
    page.save(image_path, "PNG")
    results = ocr.ocr(image_path, cls=True)
    
    print(f"--- Page {i+1} ---")
    for line in results[0]:
        text = line[1][0]
        confidence = line[1][1]
        print(f"{text} (conf: {confidence:.2f})")
```

---

## ✅ Recommendation
- **Simple & CPU only** → Tesseract + OCRmyPDF  
- **Fast + lightweight** → EasyOCR (with GPU if available)  
- **Best accuracy & speed (offline)** → PaddleOCR (with GPU)

