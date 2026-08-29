# CivicSafe

**Civic Issue Reporting Platform** — Node.js/Express backend + React Native mobile app

CivicSafe lets citizens capture and submit reports of civic issues (garbage dumps, plastic pollution, water pollution, suspicious objects, emergencies, etc.) with photo evidence and GPS location tagging, then track each report through its resolution lifecycle. Administrators use a companion set of screens in the same app to triage reports on a live map, manage status/department assignment, upload resolution evidence, and view analytics.

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Report Lifecycle](#report-lifecycle)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [API Overview](#api-overview)
- [Codebase Composition](#codebase-composition)

---

## Features

**Citizens**
- Capture an incident photo directly from the in-app camera (no gallery uploads) with automatic GPS tagging
- Select a category and add a description, then submit the report
- Optional AI-assisted validation (Google Gemini) to catch selfies, screenshots, or mismatched category/description before submission
- Track a report through 5 stages: `submitted → under_review → assigned → action_started → resolved`
- Receive push notifications as status changes
- Leave a 1–5 star rating + comment once a report is resolved

**Admins / Authorities**
- View all incoming reports on a live map, filterable by status/category/department
- Update report status and assign to a department or officer
- Upload before/after resolution photos, optionally AI-verified against the original report (same location, issue actually resolved)
- View analytics: totals by status, category, and department
- Auto-routing of new reports to the responsible department (Municipal Sanitation vs. Police/Emergency) with a derived priority level

**Platform**
- Phone-based OTP authentication via Firebase Admin SDK, backed by a first-party JWT session
- Bilingual UI (English / Hindi) via a lightweight translation layer
- Cloudinary-hosted images, MongoDB persistence via Mongoose, Expo push notifications

---

## Tech Stack

### Backend (`backend/`)

| Layer | Technology |
|---|---|
| Runtime & Framework | Node.js, Express.js, TypeScript |
| Database | MongoDB via Mongoose (schema-based models) |
| Authentication | Firebase Admin SDK (phone OTP) + JWT-based sessions with role middleware |
| Media Storage | Cloudinary |
| AI Integration | Google Gemini (`gemini-2.5-flash`) for report/resolution validation |
| Notifications | Expo Push Notifications |
| Email | SMTP (Nodemailer-style config), with a console "mock mode" fallback |
| Security Middleware | Helmet, CORS, centralized error handler, rate limiting on auth routes |
| Logging | Morgan (HTTP) + a custom JSON logger |
| Validation | Zod schemas per endpoint |

### Frontend (`app/`)

| Layer | Technology |
|---|---|
| Framework | React Native (TypeScript / `.tsx`) |
| Navigation | React Navigation — separate stacks for Auth, Citizen, and Admin flows |
| State Management | Zustand (`useAuthStore`, `useSettingsStore`) |
| Networking | Centralized Axios-based API service layer |
| Device Integration | Camera capture, GPS/geolocation |
| Internationalisation | Custom `useTranslation` hook + translation constants (English/Hindi) |
| AI Integration | Client-side service calling the Gemini-powered validation endpoints |
| Maps | `react-native-maps` for the admin live map |

---

## Project Structure

```
backend/src/
├── app.ts                  # Express app setup, middleware, route mounting
├── config/                 # env, db, cloudinary, firebase setup
├── controllers/            # admin, auth, feedback, report
├── middleware/             # auth, role, upload, error handling
├── models/                 # Mongoose schemas: User, Report, StatusHistory, Feedback, OTP
├── routes/                 # REST endpoint definitions
├── services/               # Gemini AI, Cloudinary upload, email, push notifications, routing
├── types/                  # Shared TypeScript types (JWT payload, Express augmentation)
└── utils/                  # logger, response helpers, zod validators

app/src/
├── components/             # Reusable UI: admin, auth, common, dashboard, report
├── config/                 # firebaseConfig.ts
├── constants/               # categories, colors, roles, translations, typography
├── context/                 # Zustand stores (auth, settings)
├── hooks/                   # useTranslation
├── navigation/               # Auth / Citizen / Admin navigators + root navigator
├── screens/
│   ├── admin/                # analytics, dashboard, map, reports, resolution
│   ├── auth/                 # login, signup
│   ├── citizen/               # dashboard, profile, report flow, tracking
│   ├── onboarding/
│   └── splash/
├── services/                 # API clients mirroring backend endpoints
└── types/                    # navigation, report, user types
```

---

## Report Lifecycle

```
submitted → under_review → assigned → action_started → resolved
                                                     (or → invalid, if AI/admin rejects)
```

Each transition is recorded in `StatusHistory` (who changed it, when, and any remarks), and the citizen receives a push notification at each step. Priority is derived automatically: `emergency_situation` and `suspicious_object` are always **high**; other categories default to **medium**, downgraded to **low** if AI detection confidence is under 0.4.

Category → department routing:

| Category | Department |
|---|---|
| garbage_dump, plastic_pollution, waste_accumulation, water_pollution | Municipal Sanitation |
| suspicious_object, emergency_situation | Police / Emergency |

---

## Getting Started

> The exact install/run commands depend on your `package.json` scripts, which aren't reproduced here — adjust as needed for your setup.

### Backend

```bash
cd backend
npm install
cp .env.example .env   # populate with the variables listed below
npm run dev            # or: npm run build && npm start
```

The server exposes a health check at `GET /api/health`.

### Mobile app

```bash
cd app
npm install
npx expo start
```

Update `app/src/config/firebaseConfig.ts` and the API base URL in the Axios client (`services/api.ts`) to point at your backend.

---

## Environment Variables

Configured in `backend/src/config/env.ts`:

| Variable | Purpose | Required |
|---|---|---|
| `PORT` | Server port (default `3000`) | No |
| `NODE_ENV` | `development` / `production` | No |
| `MONGO_URI` | MongoDB connection string | Yes (prod) |
| `FIREBASE_PROJECT_ID` / `FIREBASE_CLIENT_EMAIL` / `FIREBASE_PRIVATE_KEY` | Firebase Admin SDK credentials (phone OTP verification, push) | Yes (prod) |
| `CLOUDINARY_CLOUD_NAME` / `CLOUDINARY_API_KEY` / `CLOUDINARY_API_SECRET` | Image hosting | Yes (prod) |
| `JWT_SECRET` / `JWT_EXPIRES_IN` | Backend session tokens | Yes (prod) |
| `ADMIN_INVITE_CODE` | Invite code required to register as an admin | Yes (prod) |
| `CORS_ORIGIN` | Allowed origin(s), comma-separated or `*` | No |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_USER` / `SMTP_PASS` / `SMTP_FROM` | Email OTP delivery (falls back to console mock mode if unset) | No |
| `GEMINI_API_KEY` | Enables AI-assisted report/resolution validation (falls back to mock validation if unset) | No |

In `development`, missing required variables fall back to defaults instead of crashing the server; in `production` they throw on startup.

---

## API Overview

All routes are mounted under `/api`.

| Method | Endpoint | Description |
|---|---|---|
| POST | `/auth/register` | Register (citizen or admin, rate-limited) |
| POST | `/auth/login` | Login via Firebase ID token |
| POST | `/auth/send-otp` / `/auth/verify-otp` | Email OTP verification |
| GET | `/auth/me` | Current user profile |
| PATCH | `/auth/me/push-token` | Update device push token |
| POST | `/reports` | Submit a new report (citizen, multipart image) |
| GET | `/reports/user/:id` | Citizen's own reports (paginated) |
| GET | `/reports/:id` | Single report + status history |
| GET | `/admin/reports` | All reports (admin, filterable) |
| PATCH | `/admin/reports/:id/status` | Update report status |
| POST | `/admin/reports/:id/resolution` | Upload resolution evidence (multipart, optionally AI-verified) |
| GET | `/admin/analytics` | Counts by status / category / department |
| POST | `/feedback` | Submit feedback on a resolved report (citizen) |
| GET | `/health` | Health check |

Authenticated routes require `Authorization: Bearer <JWT>`; admin-only routes additionally require the `admin` role.

---

## Codebase Composition

| Area | Files | Lines of Code |
|---|---|---|
| **Backend total** | 34 | ~2,110 |
| Controllers | 4 | 716 |
| Services | 6 | 517 |
| Database Models | 5 | 307 |
| Configuration | 4 | 151 |
| Middleware | 4 | 145 |
| Utilities | 3 | 106 |
| Routes | 5 | 103 |
| Application Root | 1 | 45 |
| Type Definitions | 2 | 20 |
| **Frontend total** | 64 | ~13,170 |
| Screens | 20 | 9,113 |
| Services | 9 | 761 |
| Components | 18 | 1,989 |
| Navigation | 4 | 494 |
| Constants | 5 | 519 |
| Type Definitions | 3 | 135 |
| State / Context Stores | 2 | 85 |
| Configuration | 1 | 28 |
| Custom Hooks | 1 | 16 |
| Utilities | 1 | 30 |
| **Grand total** | **98** | **~15,280** |

---

## Notes on AI Validation

Two Gemini-backed checks exist and are gated by a client version header (`x-client-version: 2.0.0-AI`):

1. **Report validation** — checks that the submitted photo genuinely depicts the selected category (and isn't a selfie, screenshot, or clean indoor space suggesting spoofing).
2. **Resolution verification** — compares the original incident photo against the uploaded resolution photo to confirm the same location and that the issue is actually cleaned up/resolved.

Both fall back to permissive mock responses if `GEMINI_API_KEY` isn't configured, so the app remains fully functional without an AI key — validation is simply skipped.
