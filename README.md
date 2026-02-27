# Safi Oil Tracker

> AI-powered PWA for tracking cooking oil consumption using LLM vision analysis. Scan QR code → photograph bottle → get instant volume and nutrition insights.

[![Status](https://img.shields.io/badge/status-planning-blue)]() [![License](https://img.shields.io/badge/license-MIT-green)]() [![PRD](https://img.shields.io/badge/docs-PRD-orange)](docs/prd.md)

## 🎯 Project Overview

**Safi Oil Tracker** is a Progressive Web App that enables home cooking oil consumers to measure remaining oil in their bottle using their phone camera and AI vision analysis. Accessed instantly via QR code printed on the bottle — no app store download required.

### The Problem

Consumers have no practical way to track how much oil they've consumed from a bottle. Visual estimation is unreliable, measuring before every use is disruptive, and no existing product bridges the gap between physical cooking and dietary tracking.

### The Solution

1. **Scan QR code** on oil bottle → app opens with bottle context pre-loaded
2. **Photograph bottle** with phone camera → AI estimates fill level
3. **Get instant insights** → remaining/consumed volume in ml/tbsp/cups + nutritional facts
4. **Provide feedback** → help improve AI accuracy over time

### Key Innovation

**Hybrid AI + Geometry Measurement:** The LLM estimates only fill *percentage* from the photo; precise volume is calculated deterministically client-side using known bottle dimensions. This achieves ±15% accuracy without demanding precision from the imprecise component.

## ✨ Features

### POC v1 (Current Scope)

- ✅ **Zero-friction entry** via QR code deep link
- ✅ **Camera capture** with bottle framing guide
- ✅ **AI vision analysis** using Gemini 2.5 Flash (with Groq fallback)
- ✅ **Volume calculation** in ml, tablespoons, and cups
- ✅ **Nutritional insights** for consumed amount (calories, fat)
- ✅ **User feedback collection** with validation
- ✅ **Training data pipeline** for future model improvement
- ✅ **PWA capabilities** with offline app shell
- ✅ **$0/month infrastructure** (all free tiers)

### Post-POC Roadmap

- 🔲 User accounts & scan history
- 🔲 Custom starting level (partially used bottles)
- 🔲 Additional bottle SKUs & brands
- 🔲 Model fine-tuning with collected data
- 🔲 Native mobile app (React Native)
- 🔲 Multi-language support

## 🏗️ Architecture

### Tech Stack

| Layer | Technology | Rationale |
|-------|------------|----------|
| **Frontend** | Vite + React + vite-plugin-pwa | Fast dev, excellent PWA tooling |
| **Hosting** | Cloudflare Pages | Free, unlimited bandwidth |
| **API Proxy** | Cloudflare Worker | Zero cold start, R2 binding |
| **Storage** | Cloudflare R2 | S3-compatible, zero egress |
| **Primary AI** | Gemini 2.5 Flash | Free tier, JSON mode, fast |
| **Fallback AI** | Groq + Llama 4 Scout | Free tier, OpenAI-compatible |
| **Testing** | Vitest + Playwright | Modern, PWA-aware E2E |
| **CI/CD** | GitHub Actions | Free, wrangler integration |

### System Architecture

```
User Device (PWA)
  ├── QR Landing → Camera → AI Analysis → Result Display
  ├── Local: Volume calc, nutrition calc, unit conversion
  └── POST /analyze, POST /feedback
          ↓
Cloudflare Worker (Edge Proxy)
  ├── Origin validation, rate limiting
  ├── Multi-provider LLM routing (Gemini → Groq)
  ├── R2 storage (images + metadata)
  └── Feedback validation
          ↓
Cloudflare R2 Storage
  ├── images/{scanId}.jpg
  └── metadata/{scanId}.json (training data)
```

## 📚 Documentation

### Planning Artifacts

- **[Product Requirements Document (PRD)](docs/prd.md)** - Complete product specification
- **[Architecture Decision Document](docs/architecture.md)** - Technical architecture and decisions
- **[Epic Breakdown](docs/epics.md)** - 5 epics, 39 user stories
- **[UX Design Specification](docs/ux-design-specification.md)** - Complete UX design
- **[Epic Reorganization Summary](docs/epic-reorganization-summary.md)** - User value sequence

### Technical Documentation

- **[API Specification](docs/api-spec.md)** - Worker endpoints and contracts
- **[Data Schemas](docs/data-schemas.md)** - Bottle registry, nutrition data, scan metadata
- **[LLM Prompts](docs/llm-prompts.md)** - Gemini, Groq, and Ollama prompts
- **[Deployment Guide](docs/deployment-guide.md)** - Complete deployment instructions
- **[Definition of Done](docs/definition-of-done.md)** - Story/Epic/Release DoD

## 🚀 Quick Start

### Prerequisites

- Node.js 18+
- Cloudflare account (free tier)
- Gemini API key (free from Google AI Studio)
- Groq API key (free from Groq Console)

### Local Development

```bash
# Clone repository
git clone https://github.com/AhmedTElKodsh/safi-oil-tracker.git
cd safi-oil-tracker

# Install dependencies
npm install

# Set up environment variables
cp .env.example .env
# Edit .env with your API keys

# Run PWA dev server (Terminal 1)
npm run dev

# Run Worker dev server (Terminal 2)
npm run worker:dev

# Open browser
open http://localhost:5173?sku=filippo-berio-500ml
```

### Deployment

See [Deployment Guide](docs/deployment-guide.md) for complete instructions.

```bash
# Deploy to Cloudflare
npm run deploy
```

## 📊 Success Metrics

### User Success
- Scan completion rate: ≥ 80%
- Feedback submission rate: ≥ 30%
- Photo-to-result latency: < 8 seconds (p95)

### Technical Success
- LLM accuracy (MAE): ≤ 15% fill level delta
- Feedback acceptance rate: ≥ 60%
- Infrastructure cost: $0/month

### Business Success
- ≥ 50 real scans in first month
- ≥ 30 training-eligible records
- POC validates ±15% accuracy target

## 🧪 Testing

```bash
# Run unit tests
npm test

# Run E2E tests
npm run test:e2e

# Run tests with coverage
npm run test:coverage
```

## 🤝 Contributing

This is currently a POC project. Contributions will be welcomed after POC validation.

## 📝 License

MIT License - see [LICENSE](LICENSE) for details

## 👤 Author

**Ahmed**
- GitHub: [@AhmedTElKodsh](https://github.com/AhmedTElKodsh)

## 🙏 Acknowledgments

- USDA FoodData Central for nutrition data (CC0 license)
- Google Gemini for AI vision capabilities
- Groq for fallback AI infrastructure
- Cloudflare for free-tier infrastructure

---

**Status:** Planning Complete | **Next:** Implementation Sprint 1 (Epic 1)
