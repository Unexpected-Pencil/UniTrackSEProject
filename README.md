# UniTrack — Degree Progress System

A full-stack degree planning application for Midwestern State University Texas Computer Science students. The C++ backend serves a course catalog and audits a student's progress against the CS degree plan. A lightweight HTML/JavaScript frontend (no framework) provides four views: a dashboard summarizing degree progress, a personal schedule view with what-if grade projections, a course registration page with real-time conflict detection, and an alerts page surfacing prerequisite violations and other issues.

Built as a team project for CMPS 3013 Software Engineering, Spring 2026.

---

## Highlights

- **Embedded HTTP server** — single-process C++ server using [cpp-httplib](https://github.com/yhirose/cpp-httplib) on `localhost:8080`, no external dependencies
- **Real-time conflict detection** — both client and server validate that newly added courses don't overlap existing in-progress sections
- **Projected GPA** — students can preview the impact of hypothetical grades on in-progress courses without affecting their official GPA
- **Persistent alerts** — dismissed alerts survive page navigation, server restarts, and are scoped per-student via fingerprint hashing
- **Self-contained data layer** — student state auto-saves to a human-readable text file; no database required
- **Hardcoded MSU Texas CS degree plan** — rigid courses, category requirements (math/science electives), and flexible gen-ed categories (010B Communications, 030N Lab Science, 040N Language Philosophy, etc.) all encoded faithfully from the official catalog

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | C++17, cpp-httplib |
| Frontend | HTML5, CSS3, vanilla JavaScript (no framework) |
| Data | CSV catalog, pipe-delimited text persistence |
| Build | Visual Studio 2019+ |
| Platform | Windows 10/11 |

---

## Architecture

```
┌─────────────────┐    HTTP    ┌──────────────────┐    File I/O    ┌──────────────────┐
│  Browser (HTML) │ ─────────▶ │  C++ Server      │ ─────────────▶ │  CSV + TXT files │
│                 │            │  (localhost:8080)│                │                  │
│  - home.html    │ ◀───────── │                  │ ◀───────────── │  - catalog       │
│  - schedule     │   JSON     │  - REST routes   │                │  - student state │
│  - registration │            │  - DegreeAudit   │                │  - dismissed     │
│  - alerts       │            │  - Conflict chk  │                │                  │
└─────────────────┘            └──────────────────┘                └──────────────────┘
```

**Backend classes:**
- `Student` — name, major, records, GPA calculation (both official and projected)
- `AcademicRecord` — single course + grade + semester + completion status
- `Course` — catalog entry with embedded `Section` (CRN, days, times, room, instructor)
- `DegreePlan` — hardcoded MSU Texas CS curriculum (rigid + category + flex requirements)
- `DegreeAudit` — runs after every state change; produces categorized progress and alerts
- `Alert` — typed warning emitted by the audit (prereq violations, GPA, completion)

**REST endpoints:**
- `GET /student`, `POST /student`, `PUT /student`
- `GET /courses` — full catalog
- `GET /audit` — degree progress by category
- `GET /alerts`, `POST /alerts/dismiss`, `POST /alerts/restore`
- `POST /record`, `PUT /record`, `DELETE /record` — academic records
- `POST /flex` — assign a course to a flex gen-ed category

---

## Project Structure

```
UniTrack/
├── UniTrack.sln              ← open in Visual Studio
├── UniTrack.vcxproj
├── README.md
├── .gitignore
├── src/                      ← C++ source
│   ├── Server.cpp            ← entry point
│   ├── Student.cpp / .h
│   ├── AcademicRecord.cpp / .h
│   ├── Course.cpp / .h
│   ├── Section.cpp / .h
│   ├── Alert.cpp / .h
│   ├── DegreeAudit.cpp / .h
│   ├── DegreePlan.cpp / .h
│   ├── Structs.cpp / .h
│   └── httplib.h             ← header-only HTTP library
├── temp/                     ← frontend, served by the C++ server
│   ├── home.html             ← dashboard with degree progress
│   ├── schedule.html         ← current / future / history tabs
│   ├── registration.html     ← course catalog + section selection
│   ├── alerts.html           ← warnings and dismissal
│   ├── api.js                ← fetch wrappers for every endpoint
│   └── nav.js                ← sidebar navigation
└── data/                     ← CSV catalog and persisted student state
    ├── Schedule_with_Credits.csv
    └── student_data.txt
```

---

## Build & Run

### Requirements
- **Visual Studio 2019 or later** with **Desktop Development with C++** workload
- **Windows 10 or 11**

### Setup

1. Clone the repository:
   ```bash
   git clone https://github.com/YOUR-USERNAME/UniTrack.git
   ```
2. Open `UniTrack.sln` in Visual Studio
3. Set the **Working Directory** (one-time setup):
   - Right-click the project → **Properties**
   - **Configuration Properties → Debugging → Working Directory** → `$(ProjectDir)`
   - **Apply** and **OK**
4. Ensure the toolbar shows **Debug | x64**
5. Build with **Ctrl+Shift+B**
6. Run with **Ctrl+F5**

The server starts on `localhost:8080` and your default browser opens automatically.

### What you should see

In the console:
```
Catalog loaded: 124 courses
[load] reading data/student_data.txt...
[load] name set to 'Alex Johnson'
[load] major set to 'Computer Science'
[load]   ADDED CMPS1044 (A)
[load]   ADDED CMPS1063 (B)
...
Server running at http://localhost:8080/
```

In the browser, the dashboard shows Alex Johnson's current GPA, credit progress, and degree requirement categories with status indicators.

---

## Usage

**Dashboard (`home.html`):** Shows the student's official GPA (completed courses only), credits toward degree, and progress across all requirement categories. Expand any category to see which courses satisfy it.

**Schedule (`schedule.html`):** Three tabs — Current Schedule (in-progress courses for this term), Future Schedule (planned for later terms), and Course History (completed). On the current tab, each course has a "What if?" dropdown that lets you project a grade; the projected GPA pill in the header updates in real time.

**Registration (`registration.html`):** Browse the catalog grouped by department. Courses with multiple sections (e.g., BIOL1013 online vs in-person) display each section's CRN, times, and instructor under a single course header. Recommended courses at the top are derived from prerequisite eligibility. Time conflicts are detected both before submission (warning dialog) and on the server (HTTP 409 rejection).

**Alerts (`alerts.html`):** Categorized into critical (prereq violations, GPA warnings), warnings, and info. Dismissed alerts persist across sessions and are scoped per-student.

---

## Notable Implementation Details

- **GPA splits.** Two values computed in a single pass: `gpa` (completed records only — the official transcript value) and `projectedGpa` (includes in-progress courses with what-if grades set). The home page shows the official value; the schedule page shows the projection.

- **Time conflict detection.** Sections are compared by shared meeting days and overlapping time windows. Times are parsed from display strings ("1:00 PM") back to minutes since midnight for arithmetic comparison. Sections without parseable times never trigger conflicts.

- **CSV parser.** Quote-aware splitter handles course titles containing commas (e.g., `"Mechanics, Wave Motion & Heat"`) by tracking in-quotes state and only splitting on commas outside quoted fields.

- **Alert dismissal fingerprinting.** Each alert is identified by `studentName|type|courseCode|message`. Dismissals persist to `dismissed_alerts.txt` and are filtered from subsequent `GET /alerts` responses. New students start with a clean slate.

- **Section CRN propagation.** Multi-section courses share a course code but each section has a unique CRN. Registration sends the CRN to the server so the correct section is attached to the student's record — important for distinguishing online sections (no meeting times) from in-person sections.

---

## Limitations

- Single-user — no authentication or multi-tenancy
- Windows-only due to `system("start ...")` browser launch in `Server.cpp` (easily portable, but not currently)
- Catalog and degree plan are hardcoded; adding a new major requires C++ changes to `DegreePlan.cpp`
- Course records persist by code+timing, not CRN — records created before CRN-aware registration may have ambiguous section data on legacy data

---

## Acknowledgments

Course catalog and degree plan based on the published MSU Texas Computer Science requirements. Built for CMPS 3013 Software Engineering, Spring 2026.
