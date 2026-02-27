---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
inputDocuments:
  - technical-oil-bottle-ai-app-poc-research-2026-02-26.md
  - product-brief-Safi-Image-Analysis-2026-02-26.md
workflowType: "architecture"
project_name: "Safi Oil Tracker"
user_name: "Ahmed"
date: "2026-02-26"
---

# Architecture Decision Document — Safi Oil Tracker

_A Progressive Web App that uses LLM vision to estimate oil bottle fill levels and display consumption metrics with nutritional facts._

---

## Table of Contents

1. [Project Context Analysis](#1-project-context-analysis)
2. [POC Scope Boundaries](#2-poc-scope-boundaries)
3. [System Architecture Overview](#3-system-architecture-overview)
4. [Technology Stack Decisions](#4-technology-stack-decisions)
5. [Component Architecture](#5-component-architecture)
6. [Data Architecture](#6-data-architecture)
7. [API Contracts](#7-api-contracts)
8. [LLM Vision Integration](#8-llm-vision-integration)
9. [Data Collection & Feedback Loop](#9-data-collection--feedback-loop)
10. [Security Architecture](#10-security-architecture)
11. [Project Structure](#11-project-structure)
12. [Deployment Architecture](#12-deployment-architecture)
13. [Risk Register](#13-risk-register)
14. [Post-POC Evolution Path](#14-post-poc-evolution-path)

---

## 1. Project Context Analysis

### Problem Statement

Home cooking oil consumers have no practical way to track how much oil they've used from a bottle. This matters for calorie-conscious users, people managing dietary conditions, and the oil company's customer engagement strategy.

### Solution

A PWA accessible via QR code on each oil bottle. The user photographs their bottle, the app uses LLM vision to estimate the remaining fill level, calculates the volume consumed (in ml, tablespoons, and cups), and displays nutritional facts for the amount used.

### Functional Requirements (from Research)

| ID    | Requirement                                                      | POC Scope    |
| ----- | ---------------------------------------------------------------- | ------------ |
| FR-1  | QR code on bottle → opens PWA with bottle SKU pre-loaded         | In scope     |
| FR-2  | Camera capture of oil bottle showing remaining level             | In scope     |
| FR-3  | LLM Vision API estimates fill level as percentage                | In scope     |
| FR-4  | Calculate remaining/consumed volume from fill % + known geometry | In scope     |
| FR-5  | Display results in ml, tablespoons, and cups                     | In scope     |
| FR-6  | Display nutritional facts for consumed amount                    | In scope     |
| FR-7  | User feedback on estimation accuracy ("Was this correct?")       | In scope     |
| FR-8  | Store scan images + metadata + feedback for future fine-tuning   | In scope     |
| FR-9  | Feedback validation (sanity checks on user corrections)          | In scope     |
| FR-10 | Multi-provider LLM fallback chain (Gemini → Groq → local)        | In scope     |
| FR-11 | User accounts / scan history                                     | Out of scope |
| FR-12 | Custom starting level (partially used bottle as baseline)        | Out of scope |
| FR-13 | Multiple brands / dynamic bottle shape detection                 | Out of scope |
| FR-14 | Admin dashboard for reviewing feedback                           | Out of scope |
| FR-15 | Model fine-tuning pipeline                                       | Out of scope |

### Non-Functional Requirements

| NFR                 | Target                                         | Rationale                                           |
| ------------------- | ---------------------------------------------- | --------------------------------------------------- |
| **Latency**         | < 8s photo-to-result                           | User patience threshold for camera-initiated action |
| **Accuracy**        | ±15% fill level on clear glass bottles         | Current LLM vision capability ceiling               |
| **Availability**    | Best-effort (no SLA)                           | POC — free tier infrastructure                      |
| **Offline**         | App shell loads offline; scan requires network | Gemini API call requires connectivity               |
| **iOS support**     | Safari browser mode (not standalone)           | Known WebKit camera bug in standalone PWA mode      |
| **Android support** | Chrome, full PWA capabilities                  | No known blockers                                   |
| **Cost**            | $0/month infrastructure                        | All services on free tiers                          |
| **Data retention**  | All scan data retained indefinitely in R2      | Training data accumulation is strategic             |

---

## 2. POC Scope Boundaries

### In Scope (POC v1)

```
✅ PWA (responsive web app, no App Store submission)
✅ QR code on bottle → deep link to PWA with SKU parameter
✅ 2–3 bottle SKUs (defined shapes, known capacities)
✅ Clear glass bottles only (best LLM accuracy)
✅ Camera capture → LLM Vision API → fill level percentage
✅ Volume calculation: ml → tablespoons → cups
✅ Nutritional facts for consumed amount (bundled USDA data)
✅ Starting level = full bottle (100%) — hardcoded in v1
✅ Multi-provider fallback: Gemini → Groq → Ollama (local)
✅ User feedback collection with automated validation
✅ Image + metadata storage in Cloudflare R2 for future fine-tuning
✅ CI/CD pipeline (GitHub Actions → Cloudflare)
```

### Out of Scope (Post-POC)

```
🔲 Native mobile app (iOS/Android)
🔲 User accounts, authentication, scan history
🔲 Dynamic bottle shape detection (unknown bottles)
🔲 Multiple oil brands / third-party bottle registry
🔲 Custom starting level (partially used bottle baseline)
🔲 Admin dashboard for feedback review
🔲 Model fine-tuning pipeline (data collected now, tuning later)
🔲 Offline-only scan mode (requires network for LLM API)
🔲 Multi-language support
🔲 Push notifications / reminders
```

---

## 3. System Architecture Overview

### Architecture Pattern

**Thin-Client PWA + Serverless Edge Proxy + Object Storage**

All domain logic (volume calculation, unit conversion, nutrition lookup) runs in the PWA client. The Cloudflare Worker is a security proxy with data persistence responsibilities. No application server. No database server.

### System Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      USER DEVICE                             │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                    PWA (React)                        │   │
│  │                                                       │   │
│  │  QrLanding → CameraCapture → ApiStatus → ResultDisplay│   │
│  │                                                       │   │
│  │  Local Logic:                                         │   │
│  │  ├── bottleRegistry.js  (SKU → geometry + nutrition)  │   │
│  │  ├── volumeCalculator.js (fill% → ml → tbsp → cups)  │   │
│  │  └── nutritionCalculator.js (consumed ml → kcal/fat)  │   │
│  └──────────────────┬───────────────────────────────────┘   │
│                     │ POST /analyze (JPEG base64)            │
│                     │ POST /feedback (user correction)       │
└─────────────────────┼───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│              CLOUDFLARE WORKER (Edge Proxy)                   │
│                                                              │
│  Responsibilities:                                           │
│  ├── Origin validation (CORS whitelist)                      │
│  ├── Rate limiting (IP-based, 10 req/min)                    │
│  ├── Request size guard (< 4MB)                              │
│  ├── Multi-provider LLM routing (Gemini → Groq → error)     │
│  ├── Store image → Cloudflare R2                             │
│  ├── Store metadata → Cloudflare R2                          │
│  ├── Validate user feedback (sanity checks)                  │
│  └── Update metadata with validated feedback                 │
│                                                              │
│  Endpoints:                                                  │
│  ├── POST /analyze     → LLM + store                        │
│  ├── POST /feedback    → validate + update metadata          │
│  └── GET  /health      → status check                       │
└────────┬──────────────────────────────┬─────────────────────┘
         │                              │
         ▼                              ▼
┌─────────────────────┐   ┌─────────────────────────────────┐
│   LLM VISION APIs    │   │      CLOUDFLARE R2 STORAGE      │
│                      │   │                                  │
│  Primary:            │   │  images/{scanId}.jpg             │
│  Gemini 2.5 Flash    │   │  metadata/{scanId}.json          │
│                      │   │                                  │
│  Fallback:           │   │  Free tier: 10GB                 │
│  Groq Llama 4 Scout  │   │  ~140,000 scans capacity         │
│                      │   │                                  │
│  Local dev:          │   │  Purpose:                        │
│  Ollama Qwen2.5-VL   │   │  ├── Training data accumulation │
│                      │   │  ├── Feedback collection         │
└──────────────────────┘   │  └── Future fine-tuning dataset  │
                           └──────────────────────────────────┘
```

### Data Flow — Happy Path

```
1. User scans QR code on bottle
2. Browser navigates to: https://safi-oil-tracker.pages.dev/?sku=filippo-berio-500ml
3. PWA loads bottle geometry from local bottleRegistry.js
4. PWA activates rear camera via getUserMedia
5. User captures still photo → canvas → JPEG @ 800px, quality 0.78 (~70KB)
6. PWA POST /analyze { image: base64, sku: "filippo-berio-500ml" }
7. Worker validates origin, rate limit, payload size
8. Worker stores image to R2: images/{scanId}.jpg
9. Worker calls Gemini 2.5 Flash with structured JSON prompt
   └── If Gemini fails → retry once → fall back to Groq Llama 4 Scout
10. Worker receives: { fill_percentage: 42, confidence: "high" }
11. Worker stores metadata to R2: metadata/{scanId}.json
12. Worker returns { scanId, fillPercentage: 42, confidence: "high", provider: "gemini" }
13. PWA calculates: 500ml × 0.42 = 210ml remaining, 290ml consumed
14. PWA converts: 210ml = 14.2 tbsp = 0.89 cups
15. PWA calculates nutrition: 290ml consumed ≈ 19.3 tbsp × 120kcal = 2,320 kcal
16. PWA displays result with feedback prompt
17. User taps "About right" / "Too high" / "Too low"
18. PWA POST /feedback { scanId, accuracy: "about_right", userEstimate: null }
19. Worker validates feedback → updates metadata/{scanId}.json
```

---

_[Architecture document continues with remaining sections...]_