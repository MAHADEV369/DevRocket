# Skills Directory

Each skill is a self-contained, executable build instruction for one module.

## How to Use Skills

1. Start with `01-scaffold.md` (no dependencies)
2. Follow the dependency graph — each skill lists what it requires
3. After completing a skill, run its verification commands
4. Git commit after each skill: `git commit -m "skill XX: module name complete"`
5. Update `manifest.json` status to "complete"

## Dependency Graph

```
01-scaffold ─── 02-database ─── 03-auth ─── 04-middleware ─── 05-images-crud
                    │                │                              │
                    ├── 07-storage   ├── 09-websocket ─── 11-notifications
                    │                ├── 10-email
                    │                ├── 12-billing-stripe
                    │                ├── 13-oauth-google
                    │                ├── 14-2fa-totp
                    │                └── 23-frontend-setup ─── 20-pwa
                    │                                           └── 25-expo-react-native
                    ├── 08-queue-worker
                    ├── 06-gallery
                    ├── 26-db-indexing
                    └── 27-testing

15-docker ─── 16-ci-cd ─── 22-deployment
     └── 17-nginx-ssl

18-api-docs ─── (depends on 05-images-crud)
19-load-testing ─── (depends on 03-auth, 05-images-crud)
21-monitoring ─── (depends on 01-scaffold)
28-env-readme ─── (depends on 01-scaffold)
24-flutter-android ─── (depends on 03-auth, 05-images-crud)
25-expo-react-native ─── (depends on 03-auth, 05-images-crud)
```

## Skill Naming Convention

- `01` through `28`: Core skills (must do in order)
- `M-01`, `M-02`: Composition skills (combine multiple skills)
- `B-*`: Branch skills (alternatives to core skills)
- `SQ-*`: Side quests (optional enhancements)
- `I-*`: Integration skills (verify multiple skills work together)

## Input/Output Contracts

Each skill defines:

- **Input Contract**: What files/config must exist before starting
- **Output Contract**: What files the skill produces
- **Verification**: Commands to run to prove the skill worked
- **Rollback**: How to undo the skill if something goes wrong

## Composition Skills

| ID | Name | Skills Included | Time |
|----|------|----------------|------|
| M-01 | MVP Backend | 01,02,03,04,05,07,15 | 2-3 days |
| M-02 | Full Production | 01-28 | 7-10 days |
| M-03 | Minimal Deployable | 01,02,03,05,07,15 | 1 day |