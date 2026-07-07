# OCR
# CDS Receipt Audit — Setup & Usage Guide

This tool reads a folder of photographed CDS (Container Deposit Scheme) receipts
and pulls out three things into an Excel file, with a zoomed image next to each
value so you can visually double-check it:

- **Receipt No.** (handwritten, top of receipt)
- **Session ID** (the long UUID printed under "Session #")
- **Date/Time** (the timestamp line under the Session ID)

It runs entirely inside **Google Colab**, reading photos from and writing the
result back to **Google Drive**.

---

## 1. One-time folder setup in Google Drive

Create this folder structure in your Drive (names must match, or you'll need to
edit the paths in Step 3):

```
My Drive/
  CDS_Audit/
    cds_audit_ocr_colab.py          <- the script (upload this file here)
    CDS Audit Batches Required/     <- your receipt photos go here
      day 1 batches/
        20260612_152601.jpg
        20260612_152609.jpg
        ...
      day 2 batches/
        ...
```

The script scans every subfolder under `CDS Audit Batches Required` for image
files, so you can organize photos into as many "batch" subfolders as you like —
each subfolder's name shows up as the **Batch** column in the results.

---

## 2. Open Colab and mount Drive + install dependencies

Open a new notebook at [colab.research.google.com](https://colab.research.google.com),
paste this into the **first cell**, and run it:

```python
from google.colab import drive
drive.mount('/content/drive')
!apt-get -qq install -y tesseract-ocr libzbar0
!pip -q install opencv-python-headless pytesseract openpyxl pillow numpy requests pyzbar
```

Wait for it to fully finish before moving to the next cell. This installs:
- `tesseract-ocr` — reads the printed Receipt No./Date-Time text
- `libzbar0` + `pyzbar` — decodes the QR code on the receipt (see Step 5, this is
  the main accuracy fix for Session ID)
- everything else — image processing and Excel writing

**If you ever restart the Colab runtime** (Runtime → Restart session), all
installed packages are wiped — you must re-run this cell before anything else.

---

## 3. Set your paths and run the main extraction

In a **new cell**, run:

```python
%run /content/drive/MyDrive/CDS_Audit/cds_audit_ocr_colab.py
```

Before running this the first time, open the script (either in Colab's file
browser or any text editor) and check the three paths near the top under
`CONFIG` match your folder setup:

```python
INPUT_FOLDER = "/content/drive/MyDrive/CDS_Audit/CDS Audit Batches Required"
OUTPUT_EXCEL = "/content/drive/MyDrive/CDS_Audit/CDS_Audit_Results.xlsx"
TEMP_CROPS_FOLDER = "/content/drive/MyDrive/CDS_Audit/temp_crops"
```

**What you'll see while it runs:**
```
Tesseract version: 5.3.4
Found 40 images. Starting...
[1/40] day 1 batches / 20260612_152601.jpg
[2/40] day 1 batches / 20260612_152609.jpg
...
Saved Excel file: /content/drive/MyDrive/CDS_Audit/CDS_Audit_Results.xlsx
ALL DONE.
```

This can take a few minutes for ~40 images. When it finishes, open
`CDS_Audit_Results.xlsx` from Drive.

---

## 4. What's in the output file

**Sheet 1 — "CDS Audit Results"**, one row per receipt photo, with columns:

| Column | What it is |
|---|---|
| Receipt Photo | thumbnail of the whole deskewed receipt |
| Receipt No. (OCR guess) + image | handwritten number — treat as a draft, verify by eye |
| Session ID (corrected) + image | the extracted UUID, plus a zoomed crop to check it against |
| Char Flags / Notes | explains *how* the Session ID was read (see Step 5) and flags any character worth double-checking |
| Date / Time + image | the extracted timestamp, plus a zoomed crop |
| Image Quality | Good / Fair / Poor, based on blur/contrast/brightness |
| Manual Review? | `YES` if either field failed a basic plausibility check — check these rows by eye first |
| Session ID Correct? (Y/N) / Date-Time Correct? (Y/N) | **blank — this is for you to fill in**, see Step 6 |

Rows are color-coded: yellow = flagged for manual review, red = poor image
quality, green = looks fine.

**Sheet 2 — "Read Me - Methodology"**: a plain-English explanation of exactly
how each field is extracted, kept inside the workbook itself for reference.

---

## 5. How Session ID is actually read (why it's usually reliable now)

Every receipt has a QR code (meant to be scanned at the ATM to redeem it). That
QR code almost always encodes the Session ID digitally. The script:

1. **Tries the QR code first.** If it decodes and contains a UUID, that value
   is used directly — no OCR involved, so blur/small print doesn't matter.
   You'll see `"Read from QR code - high confidence"` in the Char Flags column.
2. **Falls back to OCR only if there's no QR / it didn't decode.** In that case
   it crops tightly around the printed Session ID, reads it several different
   ways, and rebuilds it into the correct UUID shape. Any character it wasn't
   fully sure about is flagged in the Notes column (e.g. `pos7:'c'(verify -
   c/e/f/1/l risk)` — meaning check that character against the image, since
   this font renders those letters very similarly).

Rows where neither method produces a cleanly-shaped ID are marked
`Manual Review? = YES` — read the zoomed image directly for those.

---

## 6. Getting Accuracy / Confusion Matrix / Type I & II errors

OCR can't grade its own correctness — that needs a human glance. So:

1. Open the Excel file. For each row, compare the zoomed **Session ID** image
   and zoomed **Date/Time** image to what's printed in those columns.
2. Fill in `Y` or `N` in the **"Session ID Correct? (Y/N)"** and
   **"Date/Time Correct? (Y/N)"** columns.
3. Save the file back to the same path in Drive.
4. Back in Colab, run:
   ```python
   %run /content/drive/MyDrive/CDS_Audit/cds_audit_ocr_colab.py --evaluate-only
   ```
5. This prints, for each field: **Accuracy**, a **Confusion Matrix**, and
   **Type I / Type II error rates** — and adds an "Accuracy Report" sheet to
   the Excel file so the numbers are saved alongside your data.

Rows you leave blank are simply skipped in the calculation, not counted as
wrong — the report tells you how many rows were actually evaluated.

*Type I error* = a row that was actually correct but got flagged for review
(wastes your time, not dangerous). *Type II error* = a row that was actually
wrong but was **not** flagged (the one to worry about — it slipped through).

---

