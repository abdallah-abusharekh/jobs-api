# Jobs API

A REST API for tracking job applications. Users register or log in to get a JSON Web Token (JWT), then use that token to create, read, update and delete their own job entries. Each job records the company, the position and the application status (`pending`, `interview` or `declined`).

Each user can see and manage only the jobs they created.

## Tech Stack

- **Node.js + Express**: HTTP server and routing
- **MongoDB + Mongoose**: data storage and schema validation
- **jsonwebtoken**: stateless authentication
- **bcryptjs**: password hashing
- **express-async-errors**: forwards errors thrown in async controllers to the error handler
- **http-status-codes**: named HTTP status constants
- **helmet, cors, xss-clean, express-rate-limit**: security middleware

## Getting Started

### Prerequisites

- Node.js 14.x (the version set in `package.json` `engines`)
- A MongoDB database, local or on MongoDB Atlas

### Environment Variables

Create a `.env` file in the project root:

```env
MONGO_URI=mongodb+srv://<user>:<password>@<cluster>/<db-name>
JWT_SECRET=<a-long-random-string>
JWT_LIFETIME=30d
PORT=5000            # optional, defaults to 5000
```

### Install and Run

```bash
npm install
npm start
```

The server connects to MongoDB first and starts listening only after the connection succeeds. `GET /` returns `Jobs API`, which you can use as a quick check that the server is up.

## Project Structure

```
.
├── app.js                  # Entry point: middleware, routes, DB connection, server start
├── db/
│   └── connect.js          # Mongoose connection helper
├── models/
│   ├── User.js             # User schema, password hashing, JWT creation, password check
│   └── Job.js              # Job schema
├── controllers/
│   ├── auth.js             # register, login
│   └── jobs.js             # getAllJobs, getJob, createJob, updateJob, deleteJob
├── routes/
│   ├── auth.js             # /api/v1/auth/*
│   └── jobs.js             # /api/v1/jobs/*
├── middleware/
│   ├── authentication.js   # Verifies the Bearer token and sets req.user
│   ├── error-handler.js    # Central error handler
│   └── not-found.js        # 404 for unknown routes
├── errors/                 # Custom error classes
│   ├── custom-api.js
│   ├── bad-request.js      # 400
│   ├── unauthenticated.js  # 401
│   ├── not-found.js        # 404
│   └── index.js
└── Procfile                # Heroku start command
```

## API Reference

Base URL: `/api/v1`

### Auth

| Method | Endpoint         | Body                          | Response                    |
| ------ | ---------------- | ----------------------------- | --------------------------- |
| POST   | `/auth/register` | `{ name, email, password }`   | `201 { user: { name }, token }` |
| POST   | `/auth/login`    | `{ email, password }`         | `200 { user: { name }, token }` |

### Jobs (authentication required)

Every jobs request needs this header:

```
Authorization: Bearer <token>
```

| Method | Endpoint     | Body                                   | Response                    |
| ------ | ------------ | -------------------------------------- | --------------------------- |
| GET    | `/jobs`      | none                                   | `200 { jobs, count }`       |
| POST   | `/jobs`      | `{ company, position, status? }`       | `201 { job }`               |
| GET    | `/jobs/:id`  | none                                   | `200 { job }`               |
| PATCH  | `/jobs/:id`  | `{ company?, position?, status? }`     | `200 { job }`               |
| DELETE | `/jobs/:id`  | none                                   | `200`                       |

`GET /jobs` returns only the current user's jobs, sorted by creation date.

### Example

```bash
# Register
curl -X POST http://localhost:5000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"john","email":"john@example.com","password":"secret123"}'

# Create a job with the returned token
curl -X POST http://localhost:5000/api/v1/jobs \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"company":"Google","position":"Backend Developer"}'
```

## Data Models

### User

| Field      | Type   | Rules                                                   |
| ---------- | ------ | ------------------------------------------------------- |
| `name`     | String | required, 3 to 50 characters                            |
| `email`    | String | required, unique, must match an email regex             |
| `password` | String | required, at least 6 characters, stored as a bcrypt hash |

### Job

| Field       | Type     | Rules                                                      |
| ----------- | -------- | ---------------------------------------------------------- |
| `company`   | String   | required, max 50 characters                                |
| `position`  | String   | required, max 100 characters                               |
| `status`    | String   | one of `interview`, `declined`, `pending`; defaults to `pending` |
| `createdBy` | ObjectId | reference to `User`, required, set from the token           |

Jobs also get `createdAt` and `updatedAt` timestamps automatically.

## How It Works

### Request Pipeline

`app.js` sets up middleware in this order:

1. `trust proxy`, so the rate limiter sees the real client IP behind a proxy such as Heroku
2. **Rate limiter**: at most 100 requests per IP every 15 minutes
3. **helmet**: secure HTTP headers
4. **cors**: allows cross-origin requests
5. **xss-clean**: removes HTML and script content from user input
6. **express.json()**: parses JSON request bodies
7. **Routes**: `/api/v1/auth` is public; `/api/v1/jobs` goes through the authentication middleware first
8. **not-found**: handles unmatched routes
9. **error-handler**: turns every thrown error into a JSON response

### Registration

1. `User.create(req.body)` runs the Mongoose schema validation on name, email and password.
2. A `pre('save')` hook generates a salt and hashes the password with bcrypt before storing it.
3. `user.createJWT()` signs a token with the payload `{ userId, name }`, using `JWT_SECRET` and `JWT_LIFETIME`.
4. The response contains the user's name and the token.

### Login

1. The controller checks that both email and password are present. If one is missing, it throws `BadRequestError` (400).
2. It looks up the user by email.
3. `user.comparePassword()` compares the candidate password with the stored hash using bcrypt.
4. If the user doesn't exist or the password is wrong, it throws `UnauthenticatedError` (401) with the same generic message, so the response doesn't reveal which emails are registered.
5. If both are correct, it returns a new token.

### Authentication Middleware

`middleware/authentication.js` reads the `Authorization` header, checks that it starts with `Bearer `, and verifies the token with `JWT_SECRET`. If the token is valid, it sets `req.user = { userId, name }`. If not, it throws `UnauthenticatedError`.

### Job Ownership

The jobs controllers use `req.user.userId`. On create, `createdBy` is set from the token, not taken from the request body. When reading a single job, the query filters by both the job `_id` and `createdBy`, so a user can't read someone else's job.

### Error Handling

Controllers throw custom errors (`BadRequestError`, `UnauthenticatedError`, `NotFoundError`). Each one extends `CustomAPIError` and has its own `statusCode`. Because of `express-async-errors`, errors thrown inside async handlers reach the central error handler without any try/catch.

The error handler returns `{ msg }` with the matching status code. It also turns common Mongoose errors into readable responses:

| Mongoose error                  | Status | Message                                               |
| ------------------------------- | ------ | ----------------------------------------------------- |
| `ValidationError`               | 400    | The validation messages, joined with commas           |
| Duplicate key (`code: 11000`)   | 400    | `Duplicate value entered for <field> field, ...`      |
| `CastError` (invalid ObjectId)  | 404    | `No item found with id : <value>`                     |

Any other error returns 500 with the error's message, or a generic fallback message.

## Deployment

The `Procfile` (`web: node app.js`) lets the app run on Heroku or any similar platform. Set `MONGO_URI`, `JWT_SECRET` and `JWT_LIFETIME` as config vars there. The platform sets `PORT` automatically.
