# 🏗️ DevRocket 🚀

<p align="center">
  <a href="https://github.com/DevRocket/DevRocket/stargazers">
    <img src="https://img.shields.io/github/stars/DevRocket/DevRocket?style=flat-square&color=FF6B6B" alt="Stars">
  </a>
  <a href="https://github.com/DevRocket/DevRocket/network/members">
    <img src="https://img.shields.io/github/forks/DevRocket/DevRocket?style=flat-square&color=4ECDC4" alt="Forks">
  </a>
  <a href="https://github.com/DevRocket/DevRocket/issues">
    <img src="https://img.shields.io/github/issues/DevRocket/DevRocket?style=flat-square&color=45B7D1" alt="Issues">
  </a>
  <a href="https://github.com/DevRocket/DevRocket/blob/main/LICENSE">
    <img src="https://img.shields.io/github/license/DevRocket/DevRocket?style=flat-square&color=96CEB4" alt="License">
  </a>
  <img src="https://img.shields.io/github/last-commit/DevRocket/DevRocket?style=flat-square&color=FFEAA7" alt="Last Commit">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Express-TypeScript-1a1a2e?style=for-the-badge&logo=express" alt="Express">
  <img src="https://img.shields.io/badge/PostgreSQL-Prisma-336791?style=for-the-badge&logo=postgresql" alt="PostgreSQL">
  <img src="https://img.shields.io/badge/Redis-BullMQ-DC382D?style=for-the-badge&logo=redis" alt="Redis">
  <img src="https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker" alt="Docker">
  <img src="https://img.shields.io/badge/React-Vite-61DAFB?style=for-the-badge&logo=react" alt="React">
  <img src="https://img.shields.io/badge/Expo-000020?style=for-the-badge&logo=expo" alt="Expo">
  <img src="https://img.shields.io/badge/Flutter-02569B?style=for-the-badge&logo=flutter" alt="Flutter">
</p>

---

## ⚡ What is DevRocket?

> A production-ready **backend blueprint** with 28 skill files that guide you to build complete full-stack applications — from database to deployment.

Stop writing boilerplate. Start shipping. 🚀

---

## 🎯 Why DevRocket?

| Problem | DevRocket Solution |
|---|---|
| "Where do I start?" | 28 skills with clear dependencies — build in order |
| "How do I structure this?" | Opinionated architecture — proven patterns |
| "What about auth?" | JWT + refresh rotation — production-ready |
| "How do I deploy?" | Docker → GitHub Actions → Railway/Fly — fully automated |
| "What about mobile?" | Both Expo & Flutter — your choice |
| "Is this production-ready?" | Sentry monitoring, rate limiting, security headers — all included |

---

## 🗺️ The Blueprint

```
┌─────────────────────────────────────────────────────────────┐
│                    DEVROCKET STACK                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐            │
│   │  React  │    │  Expo   │    │ Flutter │            │
│   │   Web   │    │   iOS   │    │Android  │            │
│   └────┬────┘    └────┬────┘    └────┬────┘            │
│        │               │               │                  │
│        └───────────────┼───────────────┘                  │
│                        ▼                                   │
│               ┌─────────────┐                           │
│               │  REST API  │                           │
│               │  Express+TS │                           │
│               └─────┬───────┘                           │
│                     │                                    │
│    ┌────────────────┼────────────────┐                  │
│    ▼                ▼                ▼                  │
│ ┌──────┐      ┌──────────┐   ┌──────────┐         │
│ │Auth  │      │ Images   │   │ Gallery  │         │
│ │JWT   │      │ Generation│   │ Sharing  │         │
│ └──────┘      └──────────┘   └──────────┘         │
│                     │                                    │
│    ┌────────────────┼────────────────┐                  │
│    ▼                ▼                ▼                  │
│ ┌──────┐      ┌──────────┐   ┌──────────┐         │
│ │Email │      │ WebSocket│   │Billing   │         │
│ │Resend│      │Real-Time │   │Stripe    │         │
│ └──────┘      └──────────┘   └──────────┘         │
│                        │                              │
│   ┌────────────────────┼────────────────────┐        │
│   ▼                    ▼                    ▼        │
│ ┌──────────┐    ┌──────────┐    ┌──────────┐    │
│ │ PostgreSQL│    │   Redis  │    │    S3   │    │
│ │  Prisma  │    │  BullMQ  │    │    R2   │    │
│ └──────────┘    └──────────┘    └──────────┘    │
│                        │                              │
│                        ▼                              │
│               ┌─────────────┐                        │
│               │   Docker    │                        │
│               │   Compose   │                        │
│               └─────┬───────┘                        │
│                     │                                │
│   ┌────────────────┼────────────────┐              │
│   ▼                ▼                ▼              │
│ ┌──────────┐    ┌──────────┐    ┌──────────┐    │
│ │GitHub   │    │ Railway  │    │  Nginx  │    │
│ │ Actions │    │ Deploy   │    │   SSL   │    │
│ └──────────┘    └──────────┘    └──────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 📚 Documentation

| Guide | Description | Time to Read |
|---|---|---|
| [README.md](README.md) | Complete repo guide | 5 min |
| [backend-infra-build-instructions.md](backend-infra-build-instructions.md) | 16-phase walkthrough | 30 min |
| [skills/README.md](skills/README.md) | Skill dependency graph | 10 min |
| [EXPERT_REVIEW.md](EXPERT_REVIEW.md) | Critical review | 15 min |

---

## 🚀 Quick Start

```bash
# 1. Clone
git clone https://github.com/DevRocket/DevRocket.git
cd DevRocket

# 2. Choose your path:
#    → Read skills/README.md for the full skill map
#    → Follow backend-infra-build-instructions.md for the narrative

# 3. Build skill by skill:
#    skills/01-scaffold.md → skills/02-database.md → skills/03-auth.md → ...
```

---

## 📂 Project Structure

```text
DevRocket/
│
├── skills/                          # 🎯 28 build skills
│   ├── 01-scaffold.md            # Project setup
│   ├── 02-database.md            # Prisma schema
│   ├── 03-auth.md                # JWT authentication
│   ├── 04-middleware.md         # Rate limiting, security
│   ├── 05-images-crud.md         # Image generation API
│   ├── 06-gallery.md             # Gallery & sharing
│   ├── 07-storage.md             # S3/R2 file upload
│   ├── 08-queue-worker.md      # BullMQ background jobs
│   ├── 09-websocket.md          # Socket.IO real-time
│   ├── 10-email.md              # Resend email
│   ├── 11-notifications.md      # In-app notifications
│   ├── 12-billing-stripe.md     # Stripe subscriptions
│   ├── 13-oauth-google.md       # Google OAuth
│   ├── 14-2fa-totp.md          # Two-factor auth
│   ├── 15-docker.md            # Docker + Compose
│   ├── 16-ci-cd.md             # GitHub Actions
│   ├── 17-nginx-ssl.md         # Nginx + SSL
│   ├── 18-api-docs.md           # Swagger/OpenAPI
│   ├── 19-load-testing.md       # k6 load tests
│   ├── 20-pwa.md                # PWA service worker
│   ├── 21-monitoring.md          # Sentry + UptimeRobot
│   ├── 22-deployment.md         # Railway/Fly.io
│   ├── 23-frontend-setup.md    # React frontend
│   ├── 24-flutter-android.md    # Flutter app
│   ├── 25-expo-react-native.md  # Expo app
│   ├── 26-db-indexing.md        # Database optimization
│   ├── 27-testing.md             # Test strategy
│   └── 28-env-readme.md         # Env + docs
│
├── skills/
│   ├── README.md                # Skill dependency graph
│   └── manifest.json            # Progress tracker
│
├── Guides/
│   ├── backend-infra-build-instructions.md
│   ├── frontend-build-instructions.md
│   ├── expo-react-native-build-instructions.md
│   ├── flutter-android-build-instructions.md
│   ├── testing-strategy.md
│   ├── ci-cd-pipelines.md
│   └── prisma-migrations-guide.md
│
└── EXPERT_REVIEW.md            # Critical review
```

---

## ✨ Features

### Backend
- ✅ JWT authentication with refresh token rotation
- ✅ Email verification & password reset
- ✅ Google OAuth sign-in
- ✅ Two-factor authentication (TOTP)
- ✅ Stripe billing & subscriptions
- ✅ Image generation with multiple providers (DALL-E, Flux, Stable Diffusion)
- ✅ Gallery & public sharing
- ✅ In-app notifications
- ✅ WebSocket real-time updates
- ✅ Background job processing
- ✅ Email service (Resend)
- ✅ File upload to S3/R2

### Infrastructure
- ✅ Docker + Docker Compose
- ✅ PostgreSQL + Prisma ORM
- ✅ Redis (cache + queue)
- ✅ Nginx reverse proxy + SSL
- ✅ GitHub Actions CI/CD
- ✅ Deployment to Railway/Fly.io
- ✅ Sentry error tracking
- ✅ Uptime monitoring

### Frontend
- ✅ React + Vite + TypeScript
- ✅ TanStack Query + Zustand
- ✅ Dark/light theming

### Mobile
- ✅ Expo (React Native) — iOS + Android
- ✅ Flutter — iOS + Android
- ✅ Push notifications

---

## 📊 Stats

<div align="center">

| Metric | Value |
|---|---|
| Documentation | **32,000+ lines** |
| Skill Files | **28** |
| Guides | **8** |
| Time to Build | **7-10 days with AI** |
| Production Ready | ✅ |

</div>

---

## 🤝 Contributing

Contributions welcome! Here's how:

1. Fork the repo
2. Create a branch: `git checkout -b feature/awesome`
3. Commit your changes: `git commit -m 'Add awesome feature'`
4. Push: `git push origin feature/awesome`
5. Open a Pull Request

See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

---

## 📝 License

MIT License — free to use, modify, and distribute.

---

## ⭐ Show Your Support

If DevRocket helped you ship faster — star the repo!

<p align="center">
  <a href="https://github.com/DevRocket/DevRocket/stargazers">
    <img src="https://gpvc.arturio.dev/DevRocket" alt="visitor count">
  </a>
</p>

---

<p align="center">
  <strong>Built with ☕ + 🧠 + 🚀</strong>
</p>

<p align="center">
  <sub>Follow the blueprint. Ship faster. 🚀</sub>
</p>