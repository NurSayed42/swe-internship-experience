# Engineering Case Studies

Architecture and engineering write-ups of software systems I built in a banking environment.

The source code for these systems belongs to the organization and is **not public**. This repository documents what each system does, how it is designed and the main engineering decisions — without exposing proprietary code, internal endpoints, credentials or data.

---

## Case Studies

| Case study | Summary | Stack |
|---|---|---|
| [Virtual Conference System (VCS)](./Virtual-Conference-System-%28VCS%29/README.md) | Prototype real-time meeting platform: instant and scheduled meetings, audio/video, group and private chat, screen sharing, participant and attendance tracking. | Real-time web communication |
| [Board Result Automation](./bd-board-result-automation/README.md) | Batch retrieval of SSC/HSC results for academic-credential verification, combining browser automation, a separate OCR microservice and relational persistence. | Spring Boot, Selenium, Python FastAPI, PaddleOCR, PostgreSQL |

## What these cover

- Breaking a business requirement into modules and services
- Real-time communication and session/attendance tracking
- Browser automation with retry and graceful-degradation strategies
- Isolating ML inference (OCR) as an independent microservice
- Persisting structured results for downstream review

## Note on confidentiality

Each write-up is limited to functionality and architecture at a level that is safe to share publicly. Implementation details, configurations and organizational data are intentionally omitted.

## Author

**Nur Sayed** — Lead Engineer at VecoSoft · [GitHub](https://github.com/NurSayed42)
