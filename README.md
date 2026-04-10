# 🏗️ GENAI — Production Backend & Infrastructure Blueprint

> A comprehensive design repository for building production-ready backend systems. Contains guides, skill files, and step-by-step instructions to build complete backend + frontend + mobile apps.

## 📚 Documentation

| File | What It Is |
|------|------------|
| [README.md](README.md) | Complete guide to this repo |
| [backend-infra-build-instructions.md](backend-infra-build-instructions.md) | 16-phase walkthrough |
| [frontend-build-instructions.md](frontend-build-instructions.md) | React + Vite + TypeScript |
| [expo-react-native-build-instructions.md](expo-react-native-build-instructions.md) | Expo mobile app |
| [flutter-android-build-instructions.md](flutter-android-build-instructions.md) | Flutter mobile app |
| [testing-strategy.md](testing-strategy.md) | Unit + Integration + E2E testing |
| [ci-cd-pipelines.md](ci-cd-pipelines.md) | GitHub Actions workflows |
| [prisma-migrations-guide.md](prisma-migrations-guide.md) | Database schema evolution |
| [EXPERT_REVIEW.md](EXPERT_REVIEW.md) | Critical review of the repo |

## ⚡ Quick Start

```bash
# 1. Clone this repo (it contains only documentation)
git clone https://github.com/DevRocket/DevRocket.git
cd DevRocket

# 2. Follow the skill files in order:
#    skills/01-scaffold.md → skills/02-database.md → skills/03-auth.md → ...

# 3. Each skill creates the actual code for your backend
```

## 🎯 What You Get

### Backend Stack
- Express.js + TypeScript
- PostgreSQL + Prisma ORM
- Redis (cache + queue)
- JWT authentication with refresh rotation
- File storage (S3/R2)
- WebSocket real-time updates
- Background job processing (BullMQ)

### Frontend Options
- **Web**: React + Vite + TypeScript + TanStack Query + Zustand
- **Mobile**: Expo (React Native) or Flutter
- **Desktop**: Tauri (optional)

### Infrastructure
- Docker + Docker Compose
- GitHub Actions CI/CD
- Nginx reverse proxy + SSL
- Deployment to Railway / Fly.io / AWS

## 📂 Project Structure

```
├── skills/                    # 28 skill files (build instructions)
│   ├── 01-scaffold.md      # Project setup
│   ├── 02-database.md      # Prisma schema
│   ├── 03-auth.md          # JWT authentication
│   ├── 04-middleware.md    # Rate limiting, validation
│   ├── 05-images-crud.md   # Image generation API
│   └── ...
│
├── skills/README.md         # Skill dependency graph
├── skills/manifest.json    # Progress tracker
│
├── backend-infra-build-instructions.md    # Narrative walkthrough
├── frontend-build-instructions.md        # React frontend guide
├── expo-react-native-build-instructions.md  # Mobile app
├── testing-strategy.md                   # Testing guide
├── ci-cd-pipelines.md                  # CI/CD workflows
└── prisma-migrations-guide.md         # Database guide
```

## 🗺️ Reading Order

1. Read `README.md` (this file)
2. Read `backend-infra-build-instructions.md` for the big picture
3. Follow skill files in order to build

See [README.md](README.md) for the complete reading order and file index.

## 📊 Stats

- **32,000+ lines** of documentation
- **28 skill files** covering backend, frontend, mobile, infra
- **6 standalone guides** for specific topics
- **0 lines of code** (this is a design repo — you build the code)

## ⚠️ Note

This repo contains **documentation only** — no runnable code. The skill files tell you what to create, but you build the actual code by following them. This is intentional: it gives you full ownership of the codebase while providing the blueprint.

## 📄 License

MIT

---

**Built with** ☕ + 🧠 by following best practices from 2025-2026