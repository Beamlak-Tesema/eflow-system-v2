# eflow-system-v2

A full-stack document workflow system for submission, routing, approval, escalation, and audit tracking.

This project has:
- **Frontend**: React (Create React App) in `client/`
- **Backend**: Node.js + Express + PostgreSQL in `server/`
- **Workflow Engine**: graph-based routing with task, condition, email, delay, and parallel nodes
- **Document Processing**: uploads via `multer`, OCR/text extraction (`tesseract.js`, `pdf-parse`), and approved-file stamping (`pdf-lib`, `sharp`)

---

## 1) Repository Structure

```text
eflow-system-v2/
├── client/
│   ├── src/
│   │   ├── components/      # Workflow builder, upload UI, modals, role manager
│   │   ├── context/         # Auth context + local storage session handling
│   │   ├── pages/           # Login, Dashboard, StudentPortal, Profile
│   │   └── api.js           # Axios API client (currently localhost-based)
│   └── package.json
├── server/
│   ├── src/
│   │   ├── routes/          # auth, workflows, documents, approvals, admin, users
│   │   ├── controllers/     # auth controller
│   │   ├── middleware/      # JWT auth middleware
│   │   ├── cron/            # SLA monitor + auto escalation
│   │   └── index.js         # Express app bootstrap
│   ├── eflow_db_2_schema.sql
│   ├── migrate-rbac.js
│   ├── fix-db.js
│   └── uploads/             # local uploaded files (served statically)
└── fix-db.js
```

---

## 2) Architecture Overview

### Frontend
- `App.js` protects `/dashboard` and `/profile` routes through `ProtectedRoute`.
- `AuthContext` stores user + JWT in `localStorage`.
- `Dashboard.jsx` handles staff/admin review, OTP approval flow, rejection, resubmission, checklist locking, and role/admin views.
- `StudentPortal.jsx` presents student-facing service cards and uploads documents tied to selected workflows.
- `WorkflowBuilder.jsx` creates workflow graphs (`nodes` + `edges`) and saves them to backend `workflows.flow_structure`.

### Backend
- `server/src/index.js` mounts API routes under `/api/*`, serves `/uploads/*`, and starts SLA monitor.
- Route modules:
  - `authRoutes.js`: register/login/profile
  - `workflowRoutes.js`: CRUD for workflow definitions
  - `documentRoutes.js`: upload/resubmit/list/history/tagging
  - `approvalRoutes.js`: OTP request, approve/reject, dynamic workflow traversal
  - `adminRoutes.js`: departments, dynamic roles, role impact/sealing, users, stats, audit logs
  - `userRoutes.js`: out-of-office/delegation settings
- Database: PostgreSQL with schema in `server/eflow_db_2_schema.sql`.

---

## 3) Prerequisites

- **Node.js**: recommended LTS (18 or 20)
- **npm**
- **PostgreSQL**
- SMTP credentials for email notifications

> Note: this repo uses native module `sharp`. If your platform/Node version is unusual, install/resolve `sharp` accordingly (see troubleshooting).

---

## 4) Environment Variables

Create `server/.env` (the backend reads from `.env` in `server/`):

```env
PORT=5000

DB_USER=postgres
DB_HOST=localhost
DB_NAME=eflow_db_2
DB_PASSWORD=your_password
DB_PORT=5432

JWT_SECRET=replace_with_a_strong_secret

SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=your_smtp_user
SMTP_PASS=your_smtp_password
FROM_EMAIL=no-reply@example.com
```

---

## 5) Database Setup

1. Create database (example):
   ```bash
   createdb eflow_db_2
   ```
2. Apply schema:
   ```bash
   psql -d eflow_db_2 -f /home/runner/work/eflow-system-v2/eflow-system-v2/server/eflow_db_2_schema.sql
   ```
3. If needed for existing DBs, run migration scripts:
   ```bash
   cd /home/runner/work/eflow-system-v2/eflow-system-v2/server
   node migrate-rbac.js
   node fix-db.js
   ```

---

## 6) Local Development

### Backend setup

```bash
cd /home/runner/work/eflow-system-v2/eflow-system-v2/server
npm install
npm run dev
# or: npm start
```

Backend default URL: `http://localhost:5000`

### Frontend setup

```bash
cd /home/runner/work/eflow-system-v2/eflow-system-v2/client
npm install
npm start
```

Frontend default URL: `http://localhost:3000`

---

## 7) File Upload Flow

1. User selects a workflow and file in `DocumentUpload.jsx` or `StudentPortal.jsx`.
2. Frontend creates `FormData` with `title`, `document`, and `workflow_id`.
3. Frontend sends `POST /api/documents/upload` (`multipart/form-data`).
4. Backend (`documentRoutes.js`) stores file via `multer` in `server/uploads/`.
5. Backend extracts text (`pdf-parse` for PDF, `tesseract.js` for images).
6. Backend stores record in `documents` table and initializes workflow assignment state.

Uploaded files are served by backend at `/uploads/*` through Express static hosting.

---

## 8) Workflow Routing Concept

Workflows are persisted as graph JSON (`flow_structure`) with `nodes` and `edges`.

### At upload/resubmit
- Backend finds the **start node** (node with no incoming edge).
- If start node is task:
  - `specific_user` → assign `current_assignee_id`
  - `role_based` + routing (`ANY`, `SPECIFIC`, `INITIATOR_DEPT`) → assign role/department fields
- Initializes SLA clock (`original_sla_deadline`) from node SLA config.

### At approval
`POST /api/approvals/approve` walks the workflow graph:
- **Condition node**: chooses `true`/`false` edge by comparing node condition value with `documents.metadata_tag`.
- **Email node**: sends templated email and continues traversal.
- **Delay node**: currently treated as pass-through.
- **Parallel node**: fans out into multiple branches and waits until all branches approve.
- **Task node**: next human approval step.
- Final step sets document status to `Approved` and stamps the uploaded PDF/image.

### Escalation / SLA monitor
A cron job in `server/src/cron/slaMonitor.js` checks pending docs every 5 minutes and auto-escalates breached items by workflow path or fallback role.

---

## 9) SMTP / Email Behavior

Email is used for:
- reviewer notifications
- submitter notifications (approval/rejection)
- workflow email nodes
- SLA urgent escalation fallback notifications

Configured through `SMTP_*` + `FROM_EMAIL` environment variables.

---

## 10) Deployment Notes (Production vs Localhost)

The app will not behave exactly like localhost unless these are addressed:

1. **Hardcoded localhost URLs exist in frontend code**
   - `client/src/api.js` uses `http://localhost:5000/api`
   - `client/src/pages/Login.jsx` posts to `http://localhost:5000/api/auth/login`
   - `client/src/components/DocumentDetailsModal.jsx` builds file URLs with `http://localhost:5000/...`
   - For production, centralize and replace with environment-based API/base file URL values.

2. **Uploads are stored on local disk (`server/uploads`)**
   - On many hosts, local disk is ephemeral.
   - Use persistent disk volumes or object storage (e.g., S3-compatible storage).

3. **CORS and security hardening**
   - Current backend uses permissive `cors()`.
   - Restrict origins and enforce HTTPS in production.

4. **Set all required environment variables in hosting platform**
   - DB, JWT, SMTP, FROM_EMAIL, PORT.

5. **Database must be hosted and reachable from backend runtime**.

---

## 11) Troubleshooting

### `react-scripts: not found`
Install frontend dependencies:
```bash
cd /home/runner/work/eflow-system-v2/eflow-system-v2/client
npm install
```

### Sharp runtime error (`Could not load the "sharp" module`)
Common with unsupported Node/platform combos:
```bash
cd /home/runner/work/eflow-system-v2/eflow-system-v2/server
npm install --include=optional sharp
# or
npm install --os=linux --cpu=x64 sharp
```
Use an LTS Node version (18/20) to reduce native module issues.

### Emails not sending
- Verify `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `FROM_EMAIL`
- Check backend startup logs for SMTP verification errors.

### Uploaded file not viewable
- Ensure backend is running and `/uploads` is publicly served.
- Confirm the file exists in `server/uploads/`.
- Verify frontend file URL base matches deployed backend domain.

---

## 12) Suggested Production Improvements

- Move all frontend URL configuration to environment variables.
- Move uploads to persistent/object storage.
- Add backend startup validation for required env vars.
- Add automated tests around workflow traversal edge cases (condition and parallel branches).

