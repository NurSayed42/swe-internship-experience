# Bangladesh Board Result Automation System

> Intelligent automation system for bulk SSC/HSC result retrieval from the Bangladesh Education Board portal — with AI-powered CAPTCHA solving, zero manual intervention, and structured result persistence.

---

## Overview

This system automates the end-to-end process of fetching official exam results for large student cohorts from [eboardresults.com](https://eboardresults.com/v2/home). It eliminates manual lookup by combining browser automation with a locally hosted AI OCR service, handling CAPTCHA challenges autonomously and storing structured results in a relational database.

Built for institutional use where hundreds of student records need to be processed reliably and efficiently.

---

## Key Features

- **Fully automated** — no human interaction required after trigger
- **AI CAPTCHA solving** — deep learning OCR (PaddleOCR) solves image-based CAPTCHAs
- **Smart retry logic** — reloads only the CAPTCHA element on failure, preserving form state
- **Dual exam support** — handles both SSC and HSC results per student in a single run
- **Fault tolerant** — skips incomplete records, retries failed attempts, logs every step
- **Structured output** — stores GPA, pass/fail status per student in the database

---

## System Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Trigger Layer                                │
│                    REST API  (Spring Boot)                           │
│                  POST /api/results/fetch [studentIds]                │
└──────────────────────────┬───────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                       Automation Engine                              │
│                      ResultService (Java)                            │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  Step 1 — Form Initialization (once per student)            │   │
│   │                                                             │   │
│   │  Open browser → Select Board / Exam / Year                  │   │
│   │  → Fill Roll & Registration → Ready for CAPTCHA             │   │
│   └──────────────────────────┬──────────────────────────────────┘   │
│                              │                                       │
│   ┌──────────────────────────▼──────────────────────────────────┐   │
│   │  Step 2 — CAPTCHA Retry Loop  (up to 45 attempts)           │   │
│   │                                                             │   │
│   │  Screenshot CAPTCHA element                                 │   │
│   │       │                                                     │   │
│   │       ▼                                                     │   │
│   │  Send to AI OCR Service ──────────────────────────────────┐ │   │
│   │       │                                                   │ │   │
│   │       ▼                                                   │ │   │
│   │  4-digit code returned?                                   │ │   │
│   │       │                                                   │ │   │
│   │    YES ──► Fill input ──► Submit ──► Parse result page    │ │   │
│   │       │                                                   │ │   │
│   │    NO  ──► Reload CAPTCHA only ──► Retry                  │ │   │
│   └──────────────────────────┬─────────────────────────────────┘   │
│                              │                                       │
│   ┌──────────────────────────▼──────────────────────────────────┐   │
│   │  Step 3 — Result Extraction & Persistence                   │   │
│   │                                                             │   │
│   │  Parse GPA from result page → Resolve Pass/Fail status      │   │
│   │  → Save to database                                         │   │
│   └─────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                       AI OCR Microservice                            │
│                    Python FastAPI  (localhost)                        │
│                                                                      │
│   Receive image bytes → PaddleOCR inference → Return 4-digit code   │
└──────────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        Data Layer                                    │
│                   PostgreSQL  via Spring Data JPA                    │
│                                                                      │
│   students table — SSC result, HSC result, GPA, Pass/Fail status    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Technology Stack

| Concern | Technology |
|---|---|
| Backend Framework | Spring Boot 3.x · Java 21 |
| Browser Automation | Selenium 4 · WebDriverManager |
| AI OCR Engine | PaddleOCR (deep learning) |
| OCR API Layer | Python · FastAPI |
| Database | PostgreSQL · Spring Data JPA |
| Inter-service Communication | HTTP multipart (RestTemplate) |

---

## Data Model

Each student record tracks both SSC and HSC examination outcomes independently.

| Field | Description |
|---|---|
| Board | Education board (e.g. Dhaka, Comilla, Rajshahi) |
| Exam Type | SSC or HSC |
| Year | Examination year |
| Roll | Examinee roll number |
| Registration | Examinee registration number |
| Result | Extracted GPA (e.g. `GPA: 5.00`) |
| Status | Resolved status — `Pass` / `Fail` / `Not Found` |

---

## Design Highlights

**Selenium-based browser automation**
The system uses Selenium 4 with WebDriverManager to drive a Chrome browser — handling dynamic page rendering, dropdown selection, form filling, and element-level screenshot capture entirely through code. No manual browser interaction is required at any point during execution.

**Stateful form preservation**
The browser form (board, year, roll, registration) is populated only once per student. On a wrong CAPTCHA, only the CAPTCHA image is refreshed via the reload button — all other form data remains intact. This eliminates redundant page loads and reduces per-attempt overhead significantly.

**Isolated AI microservice**
The OCR engine runs as a separate Python FastAPI service on localhost. This keeps the JVM process clean, allows the OCR model to be updated or swapped independently, and makes the CAPTCHA solver reusable across other projects.

**Resilient retry strategy**
With approximately 65–75% per-attempt accuracy from PaddleOCR on this CAPTCHA style, 45 retry attempts yield a mathematically near-certain success rate (>99.9%) while completing within an acceptable time window per student.

**Graceful degradation**
Records with incomplete data (missing roll, board, or year) are logged and skipped without interrupting the batch. Previously successful results are not re-fetched, making the process safely re-runnable.

---

## Performance Characteristics

| Metric | Value |
|---|---|
| Average time per CAPTCHA attempt | ~1–2 seconds |
| OCR accuracy per attempt | ~65–75% |
| Success rate over 45 attempts | >99.9% |
| Approximate throughput | 200–300 students / hour |

---

## Use Case

Designed for financial institutions and corporate HR teams that need to verify academic credentials of job applicants at scale. During recruitment drives — where hundreds of candidates submit SSC and HSC results — manual portal lookups are time-consuming and error-prone. This system automates the entire verification pipeline, cross-checking each applicant's self-declared result against the official Bangladesh Education Board records and storing the verified outcome directly in the applicant database.

---

*This repository contains documentation only. Source code is maintained in a private repository.*
