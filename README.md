# atomquest-goal-portal
# AtomQuest Hackathon 1.0 — Goal Setting & Tracking Portal

---

## Problem Statement

Organizations relying on spreadsheets, emails, and offline review cycles face three core pain points: lack of real-time visibility, goal misalignment, and audit gaps at appraisal time.

This portal provides a single, structured system of record for the complete employee goal management lifecycle — from creation and approval through quarterly check-ins to final achievement reporting.

---

## Solution Overview

A web-based, role-aware Goal Setting & Tracking Portal supporting three user roles:

| Role | Key Capabilities |
|------|-----------------|
| **Employee** | Create goals, submit for approval, log quarterly achievements |
| **Manager (L1)** | Review & approve goals, conduct check-ins, add structured feedback |
| **Admin / HR** | Manage cycles, org hierarchy, audit logs, completion dashboards |

---

## Architecture

```
[ Browser / React SPA ]
        |
        | HTTPS
        |
[ Node.js + Express REST API ]
   - JWT Auth Middleware
   - Role-Based Access Control
   - Business Rule Validation (Joi)
        |
        | pg driver
        |
[ PostgreSQL Database ]
   - Goals, Users, Check-ins
   - Audit Log (append-only)
   - Org Hierarchy
```

### Hosting
| Layer | Service | Cost |
|-------|---------|------|
| Frontend | Vercel (free tier) | $0 |
| Backend | Render.com (free tier) | $0 |
| Database | Supabase PostgreSQL (free tier) | $0 |
| Email | SendGrid (100/day free) | $0 |

---

## Tech Stack

| Category | Technology |
|----------|-----------|
| Frontend | React.js 18 + Tailwind CSS |
| State Management | React Context + Hooks |
| Backend | Node.js + Express.js |
| Database | PostgreSQL 15 (Supabase) |
| ORM | node-postgres (pg) |
| Auth | JWT (jsonwebtoken) |
| Validation | Joi |
| Export | json2csv + ExcelJS |
| Charts | Recharts |
| SSO (Bonus) | MSAL.js (Azure AD / Entra ID) |
| Notifications (Bonus) | Nodemailer + SendGrid + Teams Webhook |
| Scheduler (Bonus) | node-cron |

---

## Database Schema

### Core Tables

```sql
-- Users
CREATE TABLE users (
  user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  role VARCHAR(20) CHECK (role IN ('employee', 'manager', 'admin')),
  created_at TIMESTAMP DEFAULT NOW()
);

-- Org Hierarchy
CREATE TABLE org_hierarchy (
  employee_id UUID REFERENCES users(user_id),
  manager_id UUID REFERENCES users(user_id),
  PRIMARY KEY (employee_id)
);

-- Goal Cycles
CREATE TABLE goal_cycles (
  cycle_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  year INT NOT NULL,
  goal_setting_open DATE,
  goal_setting_close DATE,
  q1_open DATE, q1_close DATE,
  q2_open DATE, q2_close DATE,
  q3_open DATE, q3_close DATE,
  q4_open DATE, q4_close DATE,
  is_active BOOLEAN DEFAULT FALSE
);

-- Goals
CREATE TABLE goals (
  goal_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id UUID REFERENCES users(user_id),
  cycle_id UUID REFERENCES goal_cycles(cycle_id),
  thrust_area VARCHAR(255),
  title VARCHAR(500) NOT NULL,
  description TEXT,
  uom_type VARCHAR(20) CHECK (uom_type IN ('min', 'max', 'timeline', 'zero')),
  target NUMERIC,
  weightage NUMERIC CHECK (weightage >= 10),
  status VARCHAR(20) DEFAULT 'draft',
  is_locked BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Shared Goals
CREATE TABLE shared_goals (
  shared_goal_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  source_goal_id UUID REFERENCES goals(goal_id),
  recipient_id UUID REFERENCES users(user_id),
  weightage_override NUMERIC
);

-- Check-ins
CREATE TABLE checkins (
  checkin_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  goal_id UUID REFERENCES goals(goal_id),
  quarter VARCHAR(5) CHECK (quarter IN ('Q1','Q2','Q3','Q4')),
  actual_value NUMERIC,
  status VARCHAR(20) CHECK (status IN ('not_started','on_track','completed')),
  progress_score NUMERIC,
  manager_comment TEXT,
  submitted_at TIMESTAMP DEFAULT NOW()
);

-- Audit Log (append-only)
CREATE TABLE audit_log (
  log_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  actor_id UUID REFERENCES users(user_id),
  table_name VARCHAR(100),
  record_id UUID,
  action VARCHAR(50),
  old_value JSONB,
  new_value JSONB,
  reason TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);
```

---

## Business Rules

| Rule | Implementation |
|------|---------------|
| Total weightage = 100% | API recalculates sum before every insert/update |
| Min weightage per goal = 10% | Joi schema validation on request body |
| Max 8 goals per employee | COUNT query before insert; returns 409 if exceeded |
| Goals locked after approval | `is_locked` flag checked in middleware; 403 if violated |
| Check-in windows enforced | API checks current date against `goal_cycles` window dates |
| Audit trail on locked edits | Admin unlocks trigger append-only write to `audit_log` |

---

## Progress Score Formulas

| UoM Type | Formula |
|----------|---------|
| Min (higher is better, e.g. Sales Revenue) | `actual ÷ target` |
| Max (lower is better, e.g. TAT, Cost) | `target ÷ actual` |
| Timeline (date-based) | Completion date vs. Deadline |
| Zero (zero = success, e.g. Safety incidents) | `actual == 0 → 100%, else 0%` |

---

## Check-in Schedule

| Period | Window Opens | Action |
|--------|-------------|--------|
| Phase 1 — Goal Setting | 1st May | Goal Creation, Submission & Approval |
| Q1 Check-in | July | Progress Update — Planned vs. Actual |
| Q2 Check-in | October | Progress Update — Planned vs. Actual |
| Q3 Check-in | January | Progress Update — Planned vs. Actual |
| Q4 / Annual | March / April | Final Achievement Capture |

---

## API Endpoints

### Auth
```
POST   /api/auth/login
POST   /api/auth/refresh
```

### Goals
```
GET    /api/goals                  # Employee: own goals; Manager: team goals
POST   /api/goals                  # Create new goal (employee)
PUT    /api/goals/:id              # Update goal (pre-approval only)
DELETE /api/goals/:id              # Delete goal (draft only)
POST   /api/goals/:id/submit       # Submit for manager approval
POST   /api/goals/:id/approve      # Manager approves (locks goal)
POST   /api/goals/:id/return       # Manager returns for rework
POST   /api/goals/shared           # Admin/Manager pushes shared KPI
```

### Check-ins
```
GET    /api/checkins/:goalId        # Get check-in history for a goal
POST   /api/checkins               # Submit achievement update
PUT    /api/checkins/:id/comment   # Manager adds check-in comment
```

### Admin
```
GET    /api/cycles                 # List goal cycles
POST   /api/cycles                 # Create new cycle
PUT    /api/cycles/:id             # Update cycle / window dates
POST   /api/goals/:id/unlock       # Admin unlocks a locked goal
GET    /api/reports/achievement    # Export achievement report (CSV/XLSX)
GET    /api/reports/completion     # Check-in completion dashboard
GET    /api/audit                  # Audit log viewer
```

---

## Bonus Features

### Microsoft Entra ID (Azure AD) SSO
- Frontend uses `@azure/msal-browser` for SSO redirect
- Backend exchanges Azure ID token for system JWT
- Org hierarchy auto-populated from `manager` attribute in Azure AD
- Role assignment from Azure AD group membership

### Email & Teams Notifications
- Nodemailer + SendGrid for transactional emails (submission, approval, rejection, reminders)
- Teams incoming webhook with adaptive cards
- Deep-link support from Teams notification to specific goal sheet

### Escalation Module
- `node-cron` daily job evaluates configurable rules
- Escalation chain: Employee → Manager → Skip-level/HR
- All escalations logged in `escalation_log` table

### Analytics Module
- QoQ trend charts (Recharts)
- Org-wide completion heatmap
- Goal distribution by Thrust Area and UoM type
- Manager effectiveness dashboard

---

## Demo Credentials

| Role | Email | Password |
|------|-------|----------|
| Employee | employee@demo.com | Demo@1234 |
| Manager | manager@demo.com | Demo@1234 |
| Admin | admin@demo.com | Demo@1234 |

---

## Project Structure

```
atomquest-goal-portal/
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── auth/
│   │   │   ├── employee/
│   │   │   │   ├── GoalForm.jsx
│   │   │   │   ├── GoalList.jsx
│   │   │   │   └── CheckinForm.jsx
│   │   │   ├── manager/
│   │   │   │   ├── TeamDashboard.jsx
│   │   │   │   ├── GoalApproval.jsx
│   │   │   │   └── CheckinView.jsx
│   │   │   └── admin/
│   │   │       ├── AdminConsole.jsx
│   │   │       ├── CycleManager.jsx
│   │   │       └── AuditLog.jsx
│   │   ├── context/
│   │   │   └── AuthContext.jsx
│   │   ├── pages/
│   │   └── App.jsx
│   └── package.json
├── backend/
│   ├── src/
│   │   ├── routes/
│   │   │   ├── auth.js
│   │   │   ├── goals.js
│   │   │   ├── checkins.js
│   │   │   └── admin.js
│   │   ├── middleware/
│   │   │   ├── auth.js
│   │   │   ├── roleGuard.js
│   │   │   └── validate.js
│   │   ├── db/
│   │   │   ├── schema.sql
│   │   │   └── pool.js
│   │   └── index.js
│   └── package.json
└── README.md
```

---
