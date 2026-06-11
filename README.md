# WiseTrack Backend

Express and MongoDB API that collects student and teacher assessment data and calls an external machine learning service to flag students who may be at risk.

## Overview

This repository holds the backend for a hackathon project that tracks academic and well-being indicators for students. Teachers submit performance data (attendance, grades, homework streak, behavior, test scores) and students submit self-reported indicators (bullying, financial issues, mental health, physical health, gender discrimination, physical disability, motivation, and distance to school). The API stores both records in MongoDB, keyed by a shared `uniqueId`, and on request combines them into a single feature set that it forwards to an externally hosted prediction service to classify whether a student is at risk.

The commented-out CORS configuration in `src/app.js` references `https://wisetrack.vercel.app` as the intended frontend, indicating this API was built to serve a project named WiseTrack.

The actual Node.js project lives in the `node_app/` subdirectory of this repository.

## Features

- Separate submission endpoints and Mongoose schemas for student-reported and teacher-reported data
- A prediction endpoint that looks up a student's and teacher's records by `uniqueId`, merges them into the feature set expected by the ML service, and forwards the request with axios
- Consistent JSON responses and centralized error handling via `ApiResponse`, `ApiError`, and an `asyncHandler` wrapper around every route handler
- Request logging middleware that prints the method and URL of every incoming request
- CORS enabled for all origins
- Deployment configuration for Vercel serverless functions (`vercel.json`)

## Tech Stack

**Backend**
- Node.js (ES modules) with Express 4
- Axios for outbound calls to the external prediction service
- dotenv for environment variable loading

**Database**
- MongoDB with Mongoose as the ODM

**Cloud / DevOps**
- Vercel (`@vercel/node` build, configured in `vercel.json`)
- Nodemon for local development

## Architecture

```
Client
  │
  ├── POST /api/aI/feedback/student  ──►  student collection (MongoDB)
  ├── POST /api/aI/feedback/teacher  ──►  teacher collection (MongoDB)
  │
  └── POST /api/aI/predict-student
         │
         ├─► look up student doc by uniqueId
         ├─► look up teacher doc by uniqueId
         ├─► merge both into one feature payload
         └─► POST payload to external ML endpoint (Netlify-hosted)
                └─► prediction result returned to client
```

Student and teacher data are captured independently (for example, a teacher fills in attendance and grades while the student separately reports well-being factors) and joined only at prediction time. The Express app itself does not run any model — inference is delegated to a separately hosted service.

## Project Structure

```
backend-for-hackathon/
└── node_app/
    ├── vercel.json                    # Vercel serverless build/route config
    ├── package.json
    └── src/
        ├── index.js                   # Loads env vars, connects to MongoDB, starts the server
        ├── app.js                     # Express app: middleware, CORS, route mounting
        ├── constants.js               # DB_NAME constant
        ├── db/
        │   └── index.js               # Mongoose connection logic
        ├── routes/
        │   ├── aI.routes.js           # /api/aI/predict-student
        │   └── submission.routes.js   # /api/aI/feedback/student, /api/aI/feedback/teacher
        ├── controllers/
        │   ├── aI.controllers.js      # Looks up records, calls the external prediction service
        │   └── submission.controllers.js  # Validates and persists student/teacher submissions
        ├── models/
        │   ├── students.models.js     # Mongoose schema for student submissions
        │   └── teacher.models.js      # Mongoose schema for teacher submissions
        ├── functions/
        │   └── predict-student.js     # Alternate handler (CommonJS require in an ES module project); not wired to any route
        └── utils/
            ├── ApiError.js
            ├── ApiResponse.js
            └── asyncHandler.js
```

## Getting Started

### Prerequisites

- Node.js
- npm
- A MongoDB connection string (e.g. MongoDB Atlas)

### Environment variables

`src/index.js` loads variables from a `.env` file in `node_app/`, and the following are read directly in the source:

| Variable      | Read in              | Purpose                              |
|----------------|-----------------------|---------------------------------------|
| `PORT`         | `src/index.js`        | Port the Express server listens on    |
| `MONGODB_URI`  | `src/db/index.js`     | MongoDB connection string (database name `hackathonDatabase` is appended from `constants.js`) |

### Run locally

```bash
cd node_app
npm install
npm run dev
```

`npm run dev` runs `nodemon src/index.js`, which connects to MongoDB and starts listening on `PORT`.

### Build

There is no separate build step — the project runs directly on Node.js. For production, Vercel builds `node_app` using the `@vercel/node` runtime as configured in `vercel.json`.

## Usage

Submit a teacher assessment:

```bash
curl -X POST http://localhost:3000/api/aI/feedback/teacher \
  -H "Content-Type: application/json" \
  -d '{
        "uniqueId": "student-123",
        "attendance": 85,
        "grades": 72,
        "streak": 4,
        "behaviour": 3,
        "test": 65,
        "attention": 60
      }'
```

Submit a student self-assessment:

```bash
curl -X POST http://localhost:3000/api/aI/feedback/student \
  -H "Content-Type: application/json" \
  -d '{
        "uniqueId": "student-123",
        "bullying": 0,
        "financialIssues": 1,
        "mentalHealth": 0,
        "physicalHealth": 0,
        "genderDiscrimination": 0,
        "physicalDisability": 0,
        "workingAndStudying": 1,
        "interestedInStudies": 1,
        "schoolFarOff": 0
      }'
```

Request a prediction once both records exist:

```bash
curl -X POST http://localhost:3000/api/aI/predict-student \
  -H "Content-Type: application/json" \
  -d '{ "uniqueId": "student-123" }'
```

## API Documentation

| Method | Endpoint                        | Description                                                                 |
|--------|----------------------------------|-------------------------------------------------------------------------------|
| GET    | `/`                              | Basic health check                                                            |
| POST   | `/api/aI/feedback/student`       | Create a student submission (bullying, financial/mental/physical health, etc.) |
| POST   | `/api/aI/feedback/teacher`       | Create a teacher submission (attendance, grades, streak, behaviour, test, attention) |
| POST   | `/api/aI/predict-student`        | Look up a student's and teacher's records by `uniqueId`, merge them, and forward to the external prediction service |

All responses follow the same shape via `ApiResponse` / `ApiError`:

```json
{
  "statusCode": 201,
  "data": { "...": "..." },
  "message": "Submission created successfully",
  "success": true
}
```

## Design Decisions

- **Split submissions, joined at prediction time.** Student and teacher data are captured through separate endpoints and Mongoose models, and only combined into a single feature set inside `predictStudent` when a prediction is actually requested, keyed on a shared `uniqueId`.
- **Inference kept out of the Express app.** Rather than embedding a model, `aI.controllers.js` forwards the combined record to a separately hosted prediction endpoint over HTTP, keeping this service responsible for data collection and orchestration only.
- **Shared response/error shape.** `ApiResponse` and `ApiError` give every route a consistent JSON contract, and every controller is wrapped in `asyncHandler` so a single `catch` forwards errors to Express's error handling instead of repeating `try/catch` in each controller.

## Future Improvements

- The `vercel.json` build points to a root-level `index.js` (`"src": "index.js"`), but the actual entry point is `src/index.js`; this mismatch should be corrected for the Vercel deployment to build the right file.
- The external prediction service URL is hardcoded in `aI.controllers.js` rather than read from an environment variable, so changing the ML endpoint currently requires a code change.
- `node_app/.env` is committed to version control with a live MongoDB Atlas connection string; it should be removed from git history, the credentials rotated, and a `.env.example` added in its place.
- `src/functions/predict-student.js` uses CommonJS `require()` in a project configured for ES modules (`"type": "module"`) and is not referenced by any route — it should either be wired up correctly or removed.
- `package.json`'s `test` script is a placeholder that always exits with an error; there are no automated tests for the routes or controllers.
