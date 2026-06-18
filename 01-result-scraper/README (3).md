# 🎓 Bangladesh Board Result Automation

Automatic SSC/HSC result fetcher from [eboardresults.com](https://eboardresults.com/v2/home) using Selenium + AI-powered CAPTCHA solving (PaddleOCR).

---

## 📌 Overview

This project automates fetching SSC and HSC exam results for students stored in a database. It handles the CAPTCHA challenge using a local Python OCR API, fills the form via Selenium, and saves the extracted GPA back to the database — fully hands-free.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Spring Boot App                          │
│                                                                 │
│   REST Controller                                               │
│        │                                                        │
│        ▼                                                        │
│   ResultService                                                 │
│        │                                                        │
│        ├──► loadPageAndFillForm()  ──► Selenium WebDriver       │
│        │         (once per student)      (eboardresults.com)    │
│        │                                                        │
│        └──► retry loop (max 45x)                                │
│                  │                                              │
│                  ├──► trySolveCaptchaAndSubmit()                │
│                  │         │                                    │
│                  │         ├──► Screenshot captcha_img          │
│                  │         ├──► POST /solve-captcha-bytes ──────┼──► Python API
│                  │         ├──► Fill captcha input              │         │
│                  │         ├──► Submit form                     │         ▼
│                  │         └──► extractGPA()                    │    PaddleOCR
│                  │                                              │         │
│                  └──► reloadCaptchaOnly() if wrong              │    Returns 4-digit
│                                                                 │    captcha code
│   StudentRepository (JPA) ◄─── save result + status            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📁 Project Structure

```
project-root/
│
├── captcha_api.py                          # Python FastAPI CAPTCHA solver
│
└── src/main/java/com/example/resultautomation/
    ├── ResultAutomationApplication.java    # Spring Boot entry point
    │
    ├── controller/
    │   └── ResultController.java           # REST endpoint to trigger fetch
    │
    ├── service/
    │   └── ResultService.java              # Core automation logic
    │
    ├── model/
    │   └── Student.java                    # JPA entity
    │
    └── repository/
        └── StudentRepository.java          # Spring Data JPA
```

---

## ⚙️ Tech Stack

| Layer | Technology |
|---|---|
| Backend Framework | Spring Boot 3.x (Java 21) |
| Browser Automation | Selenium 4 + WebDriverManager |
| Database | PostgreSQL / MySQL (JPA) |
| CAPTCHA Solver | Python FastAPI + PaddleOCR |
| HTTP Client (Java→Python) | Spring RestTemplate (multipart) |

---

## 🔄 How It Works — Step by Step

### 1. Trigger
Call the REST endpoint with a list of student IDs:
```
POST /api/results/fetch
Body: [1, 2, 3, 4, 5]
```

### 2. Form Fill (once per student)
`loadPageAndFillForm()` opens the browser, selects Board / Exam Type / Year, fills Roll and Registration — **only once per student**.

### 3. CAPTCHA Retry Loop (up to 45x)
`trySolveCaptchaAndSubmit()` runs in a loop:
```
Screenshot captcha_img element
        │
        ▼
POST bytes → Python API /solve-captcha-bytes
        │
        ▼
4 digits returned?
   YES → Fill input → Submit → Check result page
   NO  → Click #captcha_reload → retry (form data stays intact)
```

### 4. Result Extraction
`extractGPA()` parses the result page:
- Regex match on `GPA=X.XX` in page source
- Table cell scan for `Result` column
- Fallback: detect "Failed" / "FAILED" text

### 5. Save to Database
GPA + Pass/Fail status saved to `students` table via JPA.

---

## 🐍 Python CAPTCHA API

### Endpoint
```
POST http://127.0.0.1:8000/solve-captcha-bytes
Content-Type: multipart/form-data
Body: file = <captcha image bytes>

Response:
{
  "success": true,
  "captcha_code": "5143"
}
```

### OCR Pipeline
```
Raw image bytes
      │
      ▼
3x Resize (INTER_CUBIC)
      │
      ▼
PaddleOCR → 4 digits? → Return ✅
      │
      ▼ (if not 4 digits)
{"success": false, "captcha_code": ""}
```

---

## 🚀 Setup & Run

### Prerequisites
- Java 21
- Maven
- Python 3.9+
- Google Chrome installed

### 1. Start the Python CAPTCHA API
```bash
pip install fastapi uvicorn paddleocr opencv-python numpy pillow
python captcha_api.py
# Runs on http://127.0.0.1:8000
```

### 2. Configure Database
`src/main/resources/application.properties`:
```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/yourdb
spring.datasource.username=youruser
spring.datasource.password=yourpassword
spring.jpa.hibernate.ddl-auto=update
```

### 3. Run Spring Boot
```bash
mvn spring-boot:run
```

### 4. Trigger Result Fetch
```bash
curl -X POST http://localhost:8080/api/results/fetch \
  -H "Content-Type: application/json" \
  -d '[1, 2, 3]'
```

---

## 🗃️ Student Entity Fields

| Field | Type | Description |
|---|---|---|
| `id` | Long | Primary key |
| `sscBoard` | String | e.g. `comilla`, `dhaka` |
| `sscYear` | String | e.g. `2018` |
| `sscRoll` | String | Roll number |
| `sscReg` | String | Registration number |
| `sscResult` | String | e.g. `GPA: 5.00` |
| `sscStatus` | String | `Pass` / `Fail` / `Not Found` |
| `hscBoard` | String | Same as above for HSC |
| `hscYear` | String | |
| `hscRoll` | String | |
| `hscReg` | String | |
| `hscResult` | String | |
| `hscStatus` | String | |

---

## ⚡ Key Design Decisions

**Why separate page load from captcha retry?**
The form (board/year/roll/reg) is filled only once. Only the captcha element is reloaded on each wrong attempt using `#captcha_reload` button — saving 2-3 seconds per retry.

**Why PaddleOCR only?**
After testing EasyOCR, Tesseract, and PaddleOCR on this specific captcha style (dark noisy background + white tilted digits), PaddleOCR gave the best accuracy. Adding more OCR engines increased latency without meaningful accuracy gain.

**Why 45 retries?**
Each retry takes ~1-2 seconds. With ~60-70% per-attempt accuracy, 45 retries gives >99.99% overall success probability mathematically.

---

## 📊 Performance

| Metric | Value |
|---|---|
| Avg time per captcha attempt | ~1–2 seconds |
| PaddleOCR accuracy (this captcha type) | ~65–75% per attempt |
| Success rate after 45 retries | >99.9% |
| Students processed per hour (approx) | ~200–300 |

---

## ⚠️ Notes

- The Python API **must be running** before starting the Spring Boot app.
- Chrome must be installed; WebDriverManager handles `chromedriver` automatically.
- This tool is intended for **legitimate bulk result checking** (e.g. institutional use).
- Do not use this to spam or overload the board result server.
