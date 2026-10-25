# Training Attendance Operations Portal — Project Specification & Architecture

## 1. Project Overview & Problem Statement
* **Problem:** Employees attend internal and external trainings via **Microsoft Teams**, but frequently forget to manually log their attendance in the master SharePoint Excel tracker (`Internal_Training_Attendance.xlsx`). Because of this, the office Training Dashboard fails to reflect accurate training metrics.
* **Solution:** Build an automated, end-to-end **Training Attendance Operations Portal** inside **Microsoft Teams (Power Apps + Dataverse for Teams + Power Automate)** that registers trainings, captures MS Teams attendance automatically, filters attendees against an internal employee database, enriches their profile data, and syncs clean records to the SharePoint Excel workbook after the meeting ends.

---

## 2. User Roles & Permissions
1. **Training Coordinator (Single Primary Admin):**
   * The single authorized user who manages training operations.
   * Schedules new training meetings directly from the app OR registers existing/external training invites.
   * Monitors upcoming and live trainings.
   * Triggers or validates automated/manual attendance exports to the SharePoint Excel workbook.
2. **Trainer (Variable per Session):**
   * Conducts the training session on MS Teams (changes per training topic).
3. **Attendees:**
   * Internal team members (and occasionally external participants in cross-team meetings) joining the MS Teams meeting.

---

## 3. Core Functional Workflows

### Workflow 1: Create New Training Meeting (In-App Scheduling)
* Instead of syncing the Coordinator's entire Outlook calendar (which contains personal/unrelated 1-on-1 meetings), the Coordinator schedules new trainings directly from the Power App (`+ Add Training`).
* **Action:** The app uses `Office 365 Outlook` / `Microsoft Teams` connectors to create an online Teams meeting, sends calendar invites to the Trainer and Attendees, captures the internal `Meeting ID`, and saves the record in the `Registered_Trainings` table.

### Workflow 2: Register Existing / External Meeting
* Sometimes an external team or another organizer creates the training meeting and sends an invite to the Coordinator.
* **Action:** The Coordinator clicks `+ Register an upcoming training manually` (`Register Existing Training`) and inputs the existing `Meeting Link or ID`, `Training Topic`, `Trainer Name`, and `Training Date`.

### Workflow 3: Smart Attendance Capture & "Inner-Join" Employee Filtering (CRITICAL RULE)
* **Trigger:** Automatically runs after the meeting's scheduled `End Time` (via Power Automate using MS Graph API / Teams Attendance Report) OR via a manual fallback (Coordinator clicks Sync / uploads the Teams Attendance CSV if the external organizer did not make the Coordinator a Co-organizer).
* **Smart Filtering Logic (Internal Employees Only):**
  * External meetings often include participants from outside teams or organizations.
  * When attendance records (`Participant Name` / `Email` + `Total Attendance Duration`) are fetched, the system **MUST** cross-reference every participant against the internal `Employee_Master` database table.
  * **IF Match Found in `Employee_Master`:** Keep the attendee, fetch their `Attendee Emp ID`, `Attendee ZID`, `Attendee Dept`, `Attendee Team`, and `Product` from `Employee_Master`, convert their attendance duration from seconds/minutes into hours (`0.5`, `1`, `1.5`, etc.), and stage the record for Excel export.
  * **IF No Match Found in `Employee_Master`:** Completely **ignore/drop** the participant so external attendees never pollute the internal Excel tracker.

### Workflow 4: SharePoint Excel Sync (`Internal_Training_Attendance.xlsx`)
* **When to Update:** Only after the meeting is completed (either automatically post-meeting or when the Coordinator clicks the export button in `Reports & Export`).
* **Target Workbook Structure:**
  * **File Name:** `Internal_Training_Attendance.xlsx` (hosted on SharePoint).
  * **Worksheets:** Monthly tabs named `Jan_2026`, `Feb_2026`, `Mar_2026`, ..., `August_2026`, `Sep_2026`, `Oct_2026`.
  * **Columns (A to J):**

| Column | Header Name | Source | Example Value |
| :--- | :--- | :--- | :--- |
| **A** | `Training Topic` | `Registered_Trainings` | `Introduction to GIT work Tree` |
| **B** | `Trainer Name` | `Registered_Trainings` | `Joseph Dowendy` |
| **C** | `Attendee Name` | Teams Attendance + `Employee_Master` | `Sravya Sista` |
| **D** | `Attendee Emp ID` | `Employee_Master` (Auto-Lookup) | `5504934` |
| **E** | `Attendee ZID` | `Employee_Master` (Auto-Lookup) | `Z0133184` |
| **F** | `Attendee Dept` | `Employee_Master` (Auto-Lookup) | `CDAIW` |
| **G** | `Attendee Team` | `Employee_Master` (Auto-Lookup) | `CDAIW4` |
| **H** | `Product` | `Employee_Master` (Auto-Lookup) | `Braking` / `Steering` |
| **I** | `Training Date` | `Registered_Trainings` | `27-Aug-2026` |
| **J** | `Duration (Hr)` | Calculated from Teams Attendance | `1` / `0.5` / `1.5` |

---

## 4. Database Schema (Dataverse for Teams)

Both tables are already created inside the Power Apps (Dataverse for Teams) environment:

### Table 1: `Employee_Master`
Stores unique internal team members for smart filtering and profile enrichment.
* `Attendee Name` (Text - Primary Column)
* `Employee Email` (Text - Used for reliable matching with MS Teams identity)
* `Attendee Emp ID` (Text)
* `Attendee ZID` (Text)
* `Attendee Dept` (Text)
* `Attendee Team` (Text)
* `Product` (Text)

### Table 2: `Registered_Trainings`
Stores all trainings created or registered by the Coordinator.
* `Training Topic` (Text - Primary Column)
* `Trainer Name` (Text)
* `Training Date` (Text / DateTime)
* `Meeting Link or ID` (Text)
* `Sync Status` (Text — Allowed values: `Scheduled`, `In progress`, `Completed`, `Pending validation`, `Synced`, `Needs attention`)

---

## 5. UI / UX Design Specification (5 Screens — Ultra-Dark Theme)

* **Visual Theme:** Sleek Ultra-Dark Mode (`#080808` main canvas background, `#121212` card containers, `#262626` 1px rounded borders with `12px` border radius, `#FFFFFF` primary text, `#9CA3AF` secondary muted text).
* **Global Header:**
  * Left: Calendar badge icon + Title: **Training Attendance** | Subtitle: `Training operations portal`
  * Right: Role: **Coordinator** | Subtitle: `Authorized user`
* **Global Left Navigation Sidebar:** Vertical icon rail with 5 navigation tabs:
  1. **Dashboard** (Grid Icon)
  2. **Upcoming Trainings** (Calendar Icon)
  3. **Live Attendance** (People/Group Icon)
  4. **Reports & Export** (Table/Grid Icon)
  5. **Admin Management** (Shield Check Icon)

### Screen-by-Screen Layout Details:

#### Screen 1: Dashboard
* **2x2 KPI Metric Cards Grid:**
  1. `Upcoming` — Dynamic count of scheduled trainings (`08 Training meetings`).
  2. `Live` — Dynamic count of active meetings (`01 In progress`).
  3. `Completed` — Dynamic count of trainings completed this month (`14 This month`).
  4. `Needs attention` — Dynamic count of failed/pending exports (`02 Export / attendance issues`).
* **Upcoming Trainings Preview Container:** Shows top 3 scheduled trainings (`Training Topic`, `Trainer Name • Date • Time`, `Scheduled` badge) with a `View all` link on the top right.
* **Quick Actions (2 Bottom Cards):**
  * Left Card: `Upcoming Trainings` (`Review registered meetings`) -> Navigates to Screen 2.
  * Right Card: `Reports & Export` (`Review or export records`) -> Navigates to Screen 4.

#### Screen 2: Upcoming Trainings
* **Header Action (Top Right):** Light pill button `+ Add Training` (opens form/modal to create a new MS Teams training meeting & auto-register it).
* **Main List Card:** Vertical gallery of scheduled trainings showing calendar icon, `Training Topic`, `Trainer Name • Scheduled • Outlook`, `Date • Start–End Time`, and `Attendance capture: Not started`.
* **Bottom Action Bar:** Full-width light pill button `+ Register an upcoming training manually` (opens form/modal to register an existing external meeting link/ID).

#### Screen 3: Live Attendance
* **Live Meeting Monitor Card:**
  * Displays active training title, `Meeting status: In progress • Trainer Name`, and a `Demo` / `Live` status pill.
  * Divider line followed by two large numeric counters:
    * Left Counter: `Participants seen` (Total unique participants who joined).
    * Right Counter: `Currently present` (Participants currently in the call).

#### Screen 4: Reports & Export
* **Completed Training Reports Card:**
  * Instruction text: *"Filter by training and review export status before updating the SharePoint Excel workbook."*
  * **Dropdown Control (`Training`):** Bound to completed trainings in `Registered_Trainings`.
  * **Status Label (`Export status`):** Displays current validation/sync state (e.g., `Pending validation` or `Synced to SharePoint`).
  * **Primary Action Button:** Full-width light pill button `Define export workflow` / `Export to SharePoint Excel` (triggers the Power Automate flow that filters attendees via `Employee_Master` and appends rows to `Internal_Training_Attendance.xlsx`).

#### Screen 5: Admin Management
* **Authorized App Administrators Card:**
  * Displays current active administrator (`Training Coordinator` | `Primary admin • Active` | `Admin` badge).
  * Full-width light pill button: `Define admin management`.

---

## 6. Implementation Strategy for Copilot AI
* **UI Approach:** Use a **Hybrid YAML + HTML Text** architecture in Power Apps Studio for Teams.
  * Use native Power Apps Layout Containers, Galleries, Dropdowns, Forms, and Buttons (via YAML) so all interactions (`OnSelect`, `Navigate`, `SubmitForm`, Power Automate triggers) work fluidly.
  * Use `HtmlText` controls for complex dark-mode metric cards with custom SVG icons and thin rounded borders (`border: 1px solid #262626; border-radius: 12px; background: #111111;`).
* **Power Fx Formulas Needed:**
  * `CountRows(Filter(Registered_Trainings, 'Sync Status' = "Scheduled"))` for KPI cards.
  * `Patch(Registered_Trainings, Defaults(Registered_Trainings), {...})` for creating/registering meetings.
  * Power Automate flow trigger `SyncTrainingAttendance.Run(SelectedMeetingID)` for filtering attendees against `Employee_Master` and writing to SharePoint Excel.