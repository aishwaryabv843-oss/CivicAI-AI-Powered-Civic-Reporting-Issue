# CivicAI — AI-Powered Civic Issue Reporting & Resolution Platform

CivicAI lets a citizen photograph a civic problem (pothole, garbage, water
leak, broken streetlight, etc.) and automatically:

**Photo → AI detection → severity/priority → GPS → ward/constituency →
department routing → authority notification → live status tracking.**

This is a real, working full-stack application (not a static mockup):
React/Vite frontend, Node/Express backend, MongoDB, Socket.IO real-time
updates, and Google Gemini for image analysis, with safe local fallbacks
for every paid service so the whole pipeline can be demoed without any
credentials configured.

---

## 1. Features

- **Citizen**: register/login, report an issue (camera or upload), GPS
  auto-detect with manual fallback, AI analysis preview with
  confirm/edit, complaint history, live status tracking, upvote &
  comment, notifications.
- **Authority**: dashboard of assigned complaints, accept / start work /
  resolve with a resolution photo + note, analytics.
- **Admin**: all complaints with filters, manage departments (the
  category → department → ward routing table), manage users/roles,
  system-wide analytics, priority heatmap.
- **Core pipeline**: Gemini-based image classification (with a clearly
  labeled local heuristic fallback), a backend priority engine that
  combines AI severity/confidence with community upvotes and safety
  floors, ward/constituency resolution via point-in-polygon against a
  GeoJSON boundary file (demo data included, official data pluggable),
  configurable department routing (not hard-coded), real-time
  Socket.IO notifications and status updates.

## 2. Architecture

```
CivicAI/
  client/            React + Vite + Tailwind frontend
    src/
      api/           axios service wrappers per resource
      components/    shared UI (badges, cards, toasts, maps, dashboard charts)
      context/       AuthContext, SocketContext
      layouts/       AppLayout (navbar + outlet)
      pages/         route-level pages (citizen/authority/admin/public)
  server/            Node + Express backend
    controllers/     request handlers
    models/          Mongoose schemas
    routes/          Express routers
    services/        aiService, locationService, routingService,
                      priorityService, notificationService,
                      cloudinaryService, analyticsService
    middleware/      auth (JWT), role guard, multer upload, error handler
    socket/          Socket.IO auth + room wiring
    data/boundaries/ ward/constituency GeoJSON (demo + docs for official data)
    seed/            demo data seeding script
```

## 3. Tech Stack

**Frontend**: React, Vite, Tailwind CSS v4, React Router, Axios, React
Leaflet + OpenStreetMap, Framer Motion, Recharts, Socket.IO client,
Lucide icons.

**Backend**: Node.js, Express, Socket.IO, JWT auth, bcryptjs, Multer.

**Database**: MongoDB via Mongoose (local or Atlas), 2dsphere geospatial
index on complaint locations.

**AI**: Google Gemini (`@google/generative-ai`), with a deterministic
local fallback analyzer when no API key is configured.

**Images**: Cloudinary, with an automatic local-disk fallback
(`/server/uploads`, served at `/uploads/...`) when Cloudinary
credentials are absent.

**Geo**: Browser Geolocation API, OpenStreetMap/Nominatim reverse
geocoding, `@turf/turf` for point-in-polygon ward/constituency
resolution against GeoJSON boundary files.

---

## 4. Environment Variables

Copy `server/.env.example` to `server/.env` and fill in what you have:

```
PORT=5000
NODE_ENV=development
CLIENT_URL=http://localhost:5173

MONGO_URI=mongodb://127.0.0.1:27017/civicai
JWT_SECRET=replace_with_a_long_random_secret
JWT_EXPIRES_IN=7d

GEMINI_API_KEY=
GEMINI_MODEL=gemini-1.5-flash

CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=

NOMINATIM_URL=https://nominatim.openstreetmap.org/reverse
```

- **No `GEMINI_API_KEY`?** The AI pipeline still runs end-to-end using a
  filename/heuristic-based mock classifier (`services/aiService.js`).
  Every response is tagged `aiSource: "mock"` and the UI shows a visible
  "demo mode" notice — it is never presented as a real AI verdict.
- **No Cloudinary credentials?** Images are stored locally under
  `server/uploads` and served at `/uploads/<file>`. The rest of the app
  (DB records, map, complaint detail) works identically either way.
- **Nominatim** needs no API key but is a shared public service — be
  reasonable with request volume; swap in a paid provider by editing
  only `services/locationService.js`.

The client has its own `.env.example`, but in local dev the Vite proxy
(`vite.config.js`) forwards `/api`, `/uploads` and `/socket.io` to the
backend, so no client env var is required to run locally.

---

## 5. Installation & Running Locally

### Prerequisites
- Node.js 18+
- A MongoDB instance — either local (`mongod` running on
  `localhost:27017`) or a free MongoDB Atlas cluster.

### Backend

```bash
cd server
cp .env.example .env      # then edit MONGO_URI / JWT_SECRET at minimum
npm install
npm run seed               # creates demo accounts, departments & complaints
npm run dev                # starts the API on http://localhost:5000
```

### Frontend

```bash
cd client
npm install
npm run dev                 # starts the app on http://localhost:5173
```

Open `http://localhost:5173`.

### Demo accounts (created by `npm run seed`, all use password `Demo@123`)

| Role                | Email                          |
|---------------------|---------------------------------|
| Admin               | admin@civicai.demo              |
| Authority (Roads)   | roads.officer@civicai.demo      |
| Authority (Sanit.)  | sanitation.officer@civicai.demo |
| Authority (Water)   | water.officer@civicai.demo      |
| Citizen             | citizen1@civicai.demo           |
| Citizen             | citizen2@civicai.demo           |

---

## 6. MongoDB Setup

**Local**: install MongoDB Community Server and run `mongod`; the
default `MONGO_URI` in `.env.example` already points at
`mongodb://127.0.0.1:27017/civicai`.

**Atlas**: create a free cluster at mongodb.com/atlas, add a database
user, allow your IP (or `0.0.0.0/0` for a hackathon demo), and paste the
connection string (with password filled in) into `MONGO_URI`.

## 7. Gemini Setup

1. Get a key at https://aistudio.google.com/app/apikey.
2. Set `GEMINI_API_KEY` in `server/.env`.
3. Optionally change `GEMINI_MODEL` (defaults to `gemini-1.5-flash`,
   which supports image input).

## 8. Cloudinary Setup

1. Create a free account at https://cloudinary.com.
2. From the console dashboard, copy Cloud Name, API Key, API Secret
   into `server/.env`.
3. Leave them blank to use the local-disk fallback instead.

## 9. Ward / Constituency GeoJSON Setup

See `server/data/boundaries/README.md`. In short: `bengaluru-wards-demo.geojson`
is clearly-labeled **demo** data for the hackathon. To use real
boundaries, drop an official GeoJSON `FeatureCollection` (each feature's
`properties` including `ward`, `constituency`, `source: "official"`)
into `server/data/boundaries/` and point `WARD_BOUNDARY_FILE` at it — no
code changes needed. Complaints store a `boundarySource` field
(`official` / `demo` / `unresolved`) so the UI never mislabels demo data
as authoritative.

---

## 10. API Documentation (summary)

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/auth/register` | Create a citizen or authority account |
| POST | `/api/auth/login` | Log in, returns JWT |
| GET | `/api/auth/me` | Current user |
| POST | `/api/ai/analyze` | Upload + analyze an image (preview, before submit) |
| POST | `/api/location/reverse-geocode` | Lat/lng → address/city |
| POST | `/api/location/identify-area` | Lat/lng → address + ward + constituency |
| POST | `/api/complaints` | Create complaint (runs the full pipeline) |
| GET | `/api/complaints` | List (role-aware; supports filters) |
| GET | `/api/complaints/:id` | Detail + status history |
| PUT | `/api/complaints/:id` | Edit (owner, pre-assignment) |
| DELETE | `/api/complaints/:id` | Admin only |
| POST | `/api/complaints/:id/assign` | Admin reassignment |
| PUT | `/api/complaints/:id/status` | Authority/admin status update |
| POST | `/api/complaints/:id/resolve` | Shortcut to set RESOLVED |
| POST\|DELETE | `/api/complaints/:id/upvote` | Toggle upvote |
| POST | `/api/comments` | Add a comment |
| GET | `/api/comments/:complaintId` | List comments |
| GET | `/api/notifications` | List + unread count |
| PUT | `/api/notifications/:id/read` \| `/read-all` | Mark read |
| GET | `/api/analytics/*` | overview, categories, departments, status, wards, priority, over-time, heatmap |
| GET\|POST\|PUT\|DELETE | `/api/admin/*` | users, departments, wards management |

### Socket.IO events
`complaint:created`, `complaint:assigned`, `complaint:statusUpdated`,
`complaint:upvoted`, `complaint:commented`, `notification:new`.

---

## 11. Deployment

- **Frontend → Vercel**: `vercel.json` not required for a plain Vite
  app; set the project root to `client/`, build command `npm run
  build`, output `dist`. Set `VITE_API_URL` to your deployed backend and
  update `src/api/client.js`'s `baseURL` (or configure a rewrite) since
  Vercel won't run the local dev proxy.
- **Backend → Render**: create a Web Service pointed at `server/`,
  build command `npm install`, start command `npm start`, and add all
  variables from `.env.example` in the Render dashboard.
- **Database → MongoDB Atlas**: as above; whitelist Render's egress IPs
  or `0.0.0.0/0` for the hackathon.
- Update `CLIENT_URL` on the backend and the CORS/Socket.IO origin to
  your deployed frontend URL.

---

## 12. Known Limitations

- Ward/constituency boundaries shipped in this repo are **demo data**
  for central Bengaluru only — not official BBMP/government boundaries.
- The AI mock fallback is a simple filename heuristic, not real
  computer vision; it exists solely so the pipeline can be demoed
  without a Gemini key.
- Authority-to-ward assignment during seeding is manual/simple
  (first-match), not a full on-call/load-balancing system.
- Email notifications are not implemented (in-app + Socket.IO only);
  the architecture (`notificationService.js`) makes adding an email
  provider a small, isolated change.
- No automated test suite is included; validation was done via direct
  module-level checks and a full frontend production build.

## 13. Future Improvements

- Official ward/constituency GeoJSON ingestion tooling.
- Email/SMS notifications alongside Socket.IO.
- Duplicate-complaint detection/merging near the same coordinates.
- Authority workload-based auto-assignment.
- Offline-first PWA support for citizens with poor connectivity.
- Automated tests (backend integration tests, frontend component tests).

