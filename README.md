# Automated EPUB & Journal XML Production System
## Complete Project Specification & Management Overview

---

## 1. Executive Summary

This project defines a **production-grade, automated system** that converts **PDF books, scanned pages, and screenshots** into **publisher-ready EPUB 3 files and Journal XML**.

The system is designed specifically for **academic publishing workflows**, including:
- Journals and research articles
- Academic books and monographs
- Multi-author volumes (multigraphs)

The primary objective is to **reduce manual conversion effort**, **ensure strict publisher compliance**, and **deliver high-quality outputs with minimal rework**.

---

## 2. Business Problem

Current publishing workflows rely heavily on:
- Manual tagging
- Multiple disconnected tools
- Repeated quality corrections
- High turnaround time
- Inconsistent formatting

Generic PDF-to-EPUB tools fail to handle:
- Paragraph indentation rules
- Equation numbering logic
- Inline vs figure image classification
- Journal-specific XML requirements

This leads to:
- High production cost
- Frequent publisher rejections
- Long delivery cycles

---

## 3. Proposed Solution

A **single unified system** that:
- Accepts PDF or image-based inputs
- Automatically detects structure and layout
- Produces EPUB 3 and Journal XML
- Provides a clean UI for review and correction
- Outputs publisher-ready ZIP packages

The system works **offline** and supports **confidential manuscripts**.

---

## 4. Key Business Benefits

- 70–85% reduction in manual conversion effort
- Publisher-compliant EPUB and XML output
- Offline and secure processing
- Reduced rework and correction cycles
- Scales from single articles to full books
- One platform replaces multiple tools

---

## 5. Supported Inputs

- Digital PDFs
- Scanned PDFs
- Page screenshots
- Mixed-content academic documents

---

## 6. Supported Outputs

### 6.1 EPUB 3
- Strict XHTML structure
- Validated with epubcheck
- Correct handling of:
  - Headings
  - Paragraphs
  - Figures
  - Equations
  - Tables

### 6.2 Journal XML
- Article XML
- Book XML
- Multigraph XML
- Metadata extraction:
  - Title
  - Authors
  - Affiliations
  - Abstract
  - Keywords
  - References

### 6.3 Final Delivery
- Publisher-ready ZIP package
- Includes all assets and validation-ready files

---

## 7. Core Features

### 7.1 OCR & Text Extraction
- Multi-engine OCR for high accuracy
- Confidence scoring
- Page-wise processing for stability

### 7.2 Layout Detection
- Headings
- Paragraph blocks
- Images
- Tables
- Equations

### 7.3 Paragraph Indentation Detection (Critical)
- Visual indentation analysis
- Rule-based enforcement:
  - First paragraph of chapter → no indent
  - Paragraph after heading → no indent
  - Paragraph after figure/equation → no indent
- Majority-vote logic for reliability

### 7.4 Image Classification
Images are classified as:
- Inline images
- Figures
- Display equations

Classification uses:
- Image size
- Position on page
- Caption proximity
- Math symbol density
- Confidence scoring

### 7.5 Equation Handling
- Numbered equations
- Unnumbered equations
- Regex-based number detection
- Correct XHTML structure
- Stable equation IDs (e.g., `eq2.3`)

### 7.6 Strict XHTML Rules
- Only allowed tags used
- No invented or custom tags
- `<b>` and `<i>` used instead of `<strong>` and `<em>`
- Publisher-compliant markup

---

## 8. Image Naming & Asset Rules

- Naming format: `ch{chapter}-{type}-{index}.png`
- Types:
  - `fig` → figures
  - `eq` → equations
- Index resets per chapter
- Ensures deterministic and debuggable assets

---

## 9. System Architecture

### 9.1 Backend
- Python 3.10+
- FastAPI
- Modular pipeline design

### 9.2 Frontend
- React 18
- Clean, user-friendly UI
- Live progress tracking
- Manual override tools

### 9.3 Deployment
- Local machine
- Docker
- On-premise server
- Optional cloud deployment

---

## 10. Processing Pipeline (End-to-End)

1. Input validation
2. Page extraction at 300 DPI
3. Layout analysis
4. OCR processing
5. Text normalization
6. Paragraph indentation detection
7. Image extraction and classification
8. Equation detection and numbering
9. Document structure assembly
10. EPUB XHTML generation
11. Journal XML generation (optional)
12. Validation checks
13. Final ZIP packaging

---

## 11. User Interface Workflow

1. Upload PDF or images
2. Select output format (EPUB / XML / Both)
3. Start processing
4. Monitor live progress and logs
5. Review detected structure
6. Manually adjust:
   - Paragraph indentation
   - Image type
   - Equation classification
7. Run validation
8. Download final ZIP

---

## 12. Validation & Quality Control

- EPUB validation using epubcheck
- XML well-formedness validation
- Image reference consistency checks
- OCR confidence reporting
- Structural integrity checks

---

## 13. Implementation Phases

### Phase 1 – Core EPUB Pipeline
- PDF ingestion
- OCR
- EPUB XHTML generation

### Phase 2 – Layout & Accuracy Enhancements
- Multi-engine OCR
- Improved image classification
- Equation handling

### Phase 3 – Journal XML Support
- Article, book, multigraph XML
- Metadata extraction

### Phase 4 – Frontend UI
- Review interface
- Manual correction tools

### Phase 5 – Advanced Editor
- Inline corrections
- Structural overrides

### Phase 6 – Validation & Scoring
- Automated quality metrics
- Publisher readiness score

---

## 14. Estimated Timeline

| Phase | Duration |
|------|---------|
| Phase 1 | 4–6 weeks |
| Phase 2 | 3–4 weeks |
| Phase 3 | 2–3 weeks |
| Phase 4 | 3–4 weeks |
| Phase 5–6 | 3–5 weeks |

**Total:** ~3–4 months

---

## 15. Success Metrics

- 95%+ EPUB validation pass rate
- 90%+ OCR accuracy on academic text
- <10% manual correction required
- Publisher acceptance without rework

---

## 16. Risks & Mitigation

| Risk | Mitigation |
|----|-----------|
| OCR errors | Multi-engine OCR + confidence scoring |
| Layout ambiguity | Manual override UI |
| Publisher rejection | Strict compliance rules |
| Large files | Page-wise processing |

---

## 17. Final Conclusion

This system provides a **single, scalable, and publisher-compliant platform** for academic EPUB and Journal XML production.

It replaces fragmented, manual workflows with an **automated, auditable, and production-ready solution**, significantly reducing cost, time, and quality risk.

---

---

## 20.2 Key Architectural Principles

- Modular pipeline (easy to extend)
- Offline-first design
- Deterministic output
- Manual override at every critical step
- Publisher compliance enforced at code level

---
flowchart TD
    UI[User Interface<br/>(React Web App)]
    API[API Gateway<br/>(FastAPI)]
    CORE[Document Processing Core]
    OCR[OCR Engine Manager]
    LAYOUT[Layout Analyzer]
    INDENT[Indentation Detector]
    IMG[Image Classifier]
    EQ[Equation Handler]
    ASSEMBLY[Content Assembly Layer]
    XHTML[XHTML Generator]
    XML[Journal XML Generator]
    VALIDATION[Validation Layer]
    OUTPUT[Final Output<br/>(EPUB / XML / ZIP)]

    UI --> API
    API --> CORE

    CORE --> OCR
    CORE --> LAYOUT
    CORE --> INDENT
    CORE --> IMG
    CORE --> EQ

    OCR --> ASSEMBLY
    LAYOUT --> ASSEMBLY
    INDENT --> ASSEMBLY
    IMG --> ASSEMBLY
    EQ --> ASSEMBLY

    ASSEMBLY --> XHTML
    ASSEMBLY --> XML

    XHTML --> VALIDATION
    XML --> VALIDATION

    VALIDATION --> OUTPUT
---
**Document Status:** Final  
**Audience:** Management, Engineering, Publishing Operations  
**Ready for:** Approval & Implementation
