# CrisisSync – Use Case Specifications

**Course:** Software Engineering Methods | **Assignment 1 – Part A** | **Group:** G<GroupNum>

This document holds the full specification of every use case in the CrisisSync Use Case model
(`CrisisSync_UseCase.puml`). The text is written so that it can be pasted directly into the
Visual Paradigm use case editor (*right-click use case → Open Use Case Details…*):

| Section in this document | Where it goes in Visual Paradigm 18.1 |
|---|---|
| Primary / Supporting actors, Trigger, Description | **Description** tab (documentation text) |
| Level, Complexity | **Details** tab → *Level*, *Complexity* |
| Pre-conditions | **Details** tab → *Preconditions* |
| Post-conditions (success + failure guarantee) | **Details** tab → *Post-conditions* |
| Basic Flow (numbered steps) | **Flow of Events** tab → one row per step |
| Extensions `Na` / `Nb` | **Flow of Events** tab → right-click step *N* → add extension; the extension name is the condition text, its sub-steps are the numbered lines below it |

Numbering convention: step **3** of the basic flow may have extensions **3a**, **3b**, … Each extension
is labelled *(Alternative flow)* when the use case can still succeed, or *(Exception flow)* when it
describes an error, failure or time-out and how the system recovers.

---

## 0. Shared definitions (apply to every use case)

**Actors**

| Actor | Type | Notes |
|---|---|---|
| System User | Human (abstract) | Generalisation of the five concrete user types; every user must log in with a username and password. |
| System Administrator | Human | Manages accounts and permissions, runs the data-loading utility, browses the audit history, views reports. |
| Logistics & Resources Officer | Human | Maintains the equipment and personnel pool. |
| Dispatcher | Human | Opens/cancels/closes incidents, builds task trees, reserves resources, monitors incidents live. |
| Field Commander | Human **and** a personnel resource | Sees and completes only the tasks he/she commands; can be reserved to at most one task at a time. |
| Field Crew Member | Human **and** a personnel resource | Reports shift readiness, views the assigned task, reports equipment failures. |
| National Clearing & Billing System | External system | Receives the cost summary of a closed incident and returns an approval + transaction token, or a rejection. |
| Month-End Scheduler | Timer (non-human) | Fires at the end of every calendar month (server clock) and triggers the automatic monthly reports. |

**Resource statuses** – Personnel: *Available*, *Reserved to task*, *Unavailable* (leave / sickness / off shift).
Equipment: *Available*, *Reserved to task*, *Faulty / Under repair*. A resource is atomic: at any moment it is
reserved to **at most one** task. A faulty item cannot be reserved.

**Closed lists** – Personnel roles: Paramedic, Heavy Mechanical Equipment Operator, Firefighter, Hazmat
Specialist, Rescuer, Field Commander. Equipment types: Ambulance, Fire Truck, Bulldozer, Decontamination Tent,
Generator. Incident types: Building Collapse, Fire, Road Accident, Hazmat Leak, Natural Disaster.

**Task life cycle** – `Blocked → Ready → Active → Completed`, with `Cancelled` reachable from every non-final
state. *Completed* and *Cancelled* are final. Prerequisites can be edited only while the task is *Blocked*.

**Incident statuses** – *Open*, *Closed* (billing approved), *Cancelled*.

**Global rules**

1. The relational SQL database is the single source of truth; every state change is executed in a database transaction.
2. Every task state transition, every resource update and every incident open / close / settle / cancel is written to
   the read-only **audit log** with the exact timestamp and the user (or the scheduler) that performed it.
3. Every change of a task or resource state is **pushed** by the server to all stations that display the affected
   incident (no client polling).
4. Authorisation is enforced on the server for every request, according to the role of the logged-in user.

---

## UC-01 Log In

| | |
|---|---|
| **Primary actor** | System User (any of the five user types) |
| **Supporting actors** | – |
| **Trigger** | The user launches a client station and submits a username and password. |
| **Level / Complexity** | User goal / Low |

**Description.** Every entry into CrisisSync requires a username and password. After a successful login the
station shows the interface that belongs to the user's role (one of five interfaces).

**Pre-conditions**
1. A CrisisSync client station is running and configured with the server address.

**Post-conditions**
- *Success guarantee:* an authenticated session bound to the user's role and station exists on the server; the
  login is recorded in the audit log; the role-specific home screen is displayed.
- *Failure guarantee:* no session is created; the failed attempt is recorded; nothing else in the system changes.

**Basic Flow**
1. The user enters a username and a password on the login screen and confirms.
2. The client sends the credentials to the server over the OCSF connection.
3. The server verifies the username and the password hash against the user database.
4. The server verifies that the account is active (not deactivated by the System Administrator).
5. The server creates an authenticated session bound to the user's role and to the requesting station.
6. The server records the login (user, station, timestamp) in the audit log.
7. The client displays the home screen of the user's role (Administrator, Logistics, Dispatcher, Field Commander or Field Crew Member).

**Extensions**
- **2a (Exception flow) – The server cannot be reached.**
  1. The client displays "Server unreachable – check the network connection" and offers *Retry*.
  2. The use case ends without a session.
- **3a (Exception flow) – The username or password is wrong.**
  1. The server rejects the request and records the failed attempt.
  2. The client displays "Invalid username or password" (without revealing which one is wrong).
  3. After 5 failed attempts within 10 minutes the server locks the account for 15 minutes and records the lockout. The use case ends.
- **4a (Exception flow) – The account is deactivated.**
  1. The server rejects the request; the client displays "Account disabled – contact the System Administrator". The use case ends.
- **5a (Alternative flow) – The user already has an open session on another station.**
  1. The server terminates the older session and notifies that station ("Session ended – logged in elsewhere").
  2. The flow continues at step 6.

---

## UC-02 Manage User Accounts

| | |
|---|---|
| **Primary actor** | System Administrator |
| **Supporting actors** | – |
| **Trigger** | The Administrator opens the *Users & Permissions* screen. |
| **Level / Complexity** | User goal / Medium |

**Description.** The Administrator creates, edits and deactivates user accounts and assigns each account exactly
one of the five roles (permission sets).

**Pre-conditions**
1. The user is logged in with the role System Administrator.

**Post-conditions**
- *Success guarantee:* the account is created / updated / deactivated in the database and the change is written to the audit log.
- *Failure guarantee:* no account is changed; the Administrator is told why.

**Basic Flow**
1. The Administrator opens the *Users & Permissions* screen; the system lists all accounts (username, full name, role, status).
2. The Administrator chooses *Create account* (or selects an existing account and chooses *Edit* / *Deactivate*).
3. The Administrator enters the username, full name, role (closed list of five roles) and an initial password, and confirms.
4. The system validates that the username is unique and that all mandatory fields are present.
5. The system stores the account (password stored as a salted hash) and writes the change to the audit log.
6. The system displays the updated account list with a confirmation message.

**Extensions**
- **4a (Exception flow) – The username already exists.**
  1. The system rejects the request and displays "Username already in use"; the form keeps the entered data. The flow returns to step 3.
- **4b (Exception flow) – A mandatory field is missing or the password does not meet the policy (minimum 10 characters).**
  1. The system highlights the invalid fields and returns to step 3.
- **5a (Exception flow) – The Administrator tries to deactivate the last active Administrator account or his/her own account.**
  1. The system rejects the request: "At least one active System Administrator must remain". The use case ends.
- **5b (Exception flow) – The account belongs to a Field Commander or Crew Member who is currently reserved to an Active task.**
  1. The system rejects the deactivation and shows the task that holds the resource; the account can be deactivated after the task reaches a final state. The use case ends.

---

## UC-03 Load Initial Data

| | |
|---|---|
| **Primary actor** | System Administrator |
| **Supporting actors** | – |
| **Trigger** | The Administrator starts the *Data Loading* utility from the Administrator interface. |
| **Level / Complexity** | User goal / Medium |

**Description.** A utility module (not part of routine operation) loads initial data – users, crew members and
equipment items – from a data file (SQL script, JSON or CSV) into the database.

**Pre-conditions**
1. The user is logged in with the role System Administrator.
2. A data file in one of the supported formats is accessible from the Administrator's station.

**Post-conditions**
- *Success guarantee:* all records of the file are stored in the database in one transaction; the load is recorded in the audit log with the file name and record counts.
- *Failure guarantee:* the database is unchanged (full rollback); the Administrator receives the list of errors.

**Basic Flow**
1. The Administrator opens the *Data Loading* utility and selects the data file.
2. The system detects the file format (SQL / JSON / CSV) and parses the file.
3. The system validates every record: mandatory fields, roles and equipment types from the closed lists, unique identifiers, no identifier that already exists in the database.
4. The system displays a preview summary (number of users, crew members and equipment items to be loaded) and asks for confirmation.
5. The Administrator confirms the load.
6. The system inserts all records in a single database transaction and commits.
7. The system writes the load (file name, counts, user, timestamp) to the audit log and displays the result summary.

**Extensions**
- **2a (Exception flow) – The file is not in a supported format or cannot be parsed.**
  1. The system displays the parsing error with the line / record number. Nothing is loaded. The use case ends.
- **3a (Exception flow) – One or more records are invalid (unknown role or type, duplicate identifier, missing field).**
  1. The system lists every invalid record with its reason.
  2. The system does not load the file ("all or nothing"). The Administrator corrects the file and restarts at step 1.
- **5a (Alternative flow) – The Administrator cancels at the preview.**
  1. The system discards the parsed data. The use case ends without changes.
- **6a (Exception flow) – The database transaction fails (for example, connection lost).**
  1. The system rolls back the transaction, records the failure and displays "Load failed – no data was changed". The use case ends.

---

## UC-04 Browse Incident History

| | |
|---|---|
| **Primary actor** | System Administrator |
| **Supporting actors** | – |
| **Trigger** | The Administrator opens the *Incident History* screen. |
| **Level / Complexity** | User goal / Low |

**Description.** The Administrator browses the read-only audit history: the list of open, closed and cancelled
incidents, filtered by date range, incident type and status, and drills into one incident to see its tasks, the
resources reserved to each task, the billing approval and the complete timeline of actions.

**Pre-conditions**
1. The user is logged in with the role System Administrator.

**Post-conditions**
- *Success guarantee:* the requested history is displayed; no data is modified (the history is read-only).
- *Failure guarantee:* nothing changes; an explanatory message is displayed.

**Basic Flow**
1. The Administrator opens the *Incident History* screen.
2. The system displays all incidents (name, type, location, opening time, status, closing / cancellation time).
3. The Administrator sets filters: date range, incident type (closed list) and status (Open / Closed / Cancelled), and applies them.
4. The system displays the incidents matching all filters.
5. The Administrator selects an incident.
6. The system displays the incident's tasks with their final states, the resources that were reserved to each task, the billing approval (transaction token and time) and the full timeline: every state transition, resource update and incident action with the exact timestamp and the user who performed it.
7. The Administrator returns to the list or closes the screen.

**Extensions**
- **3a (Exception flow) – The date range is invalid (start after end, or in the future).**
  1. The system highlights the range and keeps the previous result. The flow returns to step 3.
- **4a (Alternative flow) – No incident matches the filters.**
  1. The system displays "No incidents match the selected filters". The flow returns to step 3.
- **6a (Alternative flow) – The incident was cancelled (no billing).**
  1. The system shows the cancellation time and the user instead of a billing approval. The flow continues at step 7.

---

## UC-05 View Management Reports

| | |
|---|---|
| **Primary actor** | System Administrator |
| **Supporting actors** | – |
| **Trigger** | The Administrator opens the *Reports* screen. |
| **Level / Complexity** | User goal / Low |

**Description.** The Administrator views the monthly reports generated by the system: the Resource Utilization
Report and the SLA Compliance Report of any past month.

**Pre-conditions**
1. The user is logged in with the role System Administrator.

**Post-conditions**
- *Success guarantee:* the selected report is displayed in its graphical form.
- *Failure guarantee:* nothing changes; the Administrator is offered to generate the missing report on demand.

**Basic Flow**
1. The Administrator opens the *Reports* screen.
2. The system lists the stored reports (month, report type, generation time, generated automatically / on demand).
3. The Administrator selects a report.
4. The system renders the report: utilization percentages as bar charts per equipment type and per role, or average incident duration per incident type (SLA).
5. The Administrator may export the report (PDF) or return to the list.

**Extensions**
- **2a (Alternative flow) – No report exists yet for the month the Administrator is interested in.**
  1. The Administrator selects the month and chooses *Generate now*; the system performs UC-06 / UC-07 on demand.
  2. The flow continues at step 4 with the newly generated report.
- **4a (Exception flow) – The stored report data is missing or corrupted.**
  1. The system displays "Report unavailable" and offers to regenerate it on demand (UC-06 / UC-07). The use case ends.

---

## UC-06 Generate Resource Utilization Report

| | |
|---|---|
| **Primary actor** | Month-End Scheduler (automatic) – or System Administrator (on demand for a selected past month) |
| **Supporting actors** | – |
| **Trigger** | The server clock reaches 00:00 on the first day of a new month, or the Administrator requests the report for a past month. |
| **Level / Complexity** | User goal / Medium |

**Description.** The system computes, for every equipment item and crew member, the share of the month during
which the resource was reserved to a task, and presents it graphically grouped by equipment type and by role.

**Pre-conditions**
1. Automatic generation: the calendar month has ended according to the server clock.
2. On demand: the user is logged in as System Administrator and the selected month is a completed past month.

**Post-conditions**
- *Success guarantee:* the report for the month is computed from the audit log, stored with its generation timestamp and (on demand) displayed.
- *Failure guarantee:* no partial report is stored; the failure is logged; on demand the Administrator is notified; automatic generation is retried.

**Basic Flow**
1. The Month-End Scheduler fires at the month boundary (or the Administrator selects a past month and chooses *Generate Resource Utilization Report*).
2. The system determines the reporting period (first to last day of the month).
3. The system reads from the audit log every reservation and release event of every resource within the period.
4. For each resource the system computes `reserved time within the month / total time of the month` as a percentage.
5. The system aggregates the percentages per equipment type and per personnel role (average and number of resources).
6. The system renders the results as bar charts and stores the report with the generation timestamp and the trigger (automatic / on demand).
7. On demand, the system displays the report to the Administrator; automatically generated reports become available in UC-05.

**Extensions**
- **1a (Exception flow) – The Administrator selects the current or a future month.**
  1. The system rejects the request: "Reports can be generated only for completed months". The use case ends.
- **3a (Alternative flow) – No reservation events exist in the period.**
  1. The system generates the report with 0 % utilization for every resource and marks it "no activity". The flow continues at step 6.
- **6a (Exception flow) – Generation fails (database unavailable).**
  1. The system discards partial results and logs the failure.
  2. Automatic generation: the scheduler retries up to 3 times at 10-minute intervals; after the third failure an alert is shown to the Administrator at the next login.
  3. On demand: the Administrator sees "Report generation failed – please retry". The use case ends.

---

## UC-07 Generate SLA Compliance Report

| | |
|---|---|
| **Primary actor** | Month-End Scheduler (automatic) – or System Administrator (on demand for a selected past month) |
| **Supporting actors** | – |
| **Trigger** | The server clock reaches 00:00 on the first day of a new month, or the Administrator requests the report for a past month. |
| **Level / Complexity** | User goal / Medium |

**Description.** The system computes, per incident type, the average time from the opening of an incident until
the completion of its last task, over the incidents whose last task was completed in the reported month.

**Pre-conditions**
1. Automatic generation: the calendar month has ended according to the server clock.
2. On demand: the user is logged in as System Administrator and the selected month is a completed past month.

**Post-conditions**
- *Success guarantee:* the SLA report for the month is computed, stored with its generation timestamp and (on demand) displayed.
- *Failure guarantee:* no partial report is stored; the failure is logged and retried (automatic) or reported to the Administrator (on demand).

**Basic Flow**
1. The Month-End Scheduler fires at the month boundary (or the Administrator selects a past month and chooses *Generate SLA Compliance Report*).
2. The system determines the reporting period.
3. The system selects every incident whose last task reached the *Completed* state inside the period and that was not cancelled.
4. For each selected incident the system computes `completion time of the last task − incident opening time`.
5. The system groups the durations by incident type and computes the average, the number of incidents and the longest duration per type.
6. The system renders the result as a chart and a table and stores the report with the generation timestamp and the trigger.
7. On demand, the system displays the report to the Administrator.

**Extensions**
- **1a (Exception flow) – The Administrator selects the current or a future month.**
  1. The system rejects the request: "Reports can be generated only for completed months". The use case ends.
- **3a (Alternative flow) – No incident of a given type was completed in the period.**
  1. The system marks that incident type as "N/A" instead of a zero value and continues with steps 4–5 for the other incident types.
- **3b (Alternative flow) – An incident is still open at the end of the month.**
  1. The incident is excluded from this month's report and will be counted in the month in which its last task is completed.
- **6a (Exception flow) – Generation fails (database unavailable).**
  1. The system discards partial results and logs the failure.
  2. Automatic generation: the scheduler retries up to 3 times at 10-minute intervals and alerts the Administrator after the third failure.
  3. On demand: the Administrator sees "Report generation failed – please retry". The use case ends.

---

## UC-08 Update Equipment Status

| | |
|---|---|
| **Primary actor** | Logistics & Resources Officer |
| **Supporting actors** | – |
| **Trigger** | The Officer selects an equipment item in the *Resource Pool* screen and chooses *Update status*. |
| **Level / Complexity** | User goal / Low |

**Description.** The Officer sets the maintenance status of an equipment item to *Faulty / Under repair* or returns
a repaired item to *Available*.

**Pre-conditions**
1. The user is logged in with the role Logistics & Resources Officer.
2. The equipment item exists in the pool.

**Post-conditions**
- *Success guarantee:* the item's status is updated, the change is in the audit log and pushed to every station that displays the item.
- *Failure guarantee:* the status is unchanged; the Officer is told why.

**Basic Flow**
1. The Officer opens the *Resource Pool* screen and selects an equipment item; the system shows its type, location and current status.
2. The Officer chooses the new status (*Faulty / Under repair* or *Available*) and enters a short note.
3. The system verifies that the item is not currently reserved to a task.
4. The system updates the status in the database and writes the change (item, old status, new status, user, timestamp) to the audit log.
5. The system pushes the updated status to all stations displaying the pool or an incident that lists the item.
6. The system displays a confirmation.

**Extensions**
- **3a (Exception flow) – The item is reserved to an Active task.**
  1. The system rejects the manual change and shows the task that holds the item ("A fault during an active task must be reported from the field – UC-28"). The use case ends.
- **4a (Exception flow) – The database update fails.**
  1. The system rolls back, displays "Update failed – please retry" and leaves the status unchanged. The use case ends.

---

## UC-09 Transfer Equipment Between Stations

| | |
|---|---|
| **Primary actor** | Logistics & Resources Officer |
| **Supporting actors** | – |
| **Trigger** | The Officer selects an equipment item and chooses *Transfer*. |
| **Level / Complexity** | User goal / Low |

**Pre-conditions**
1. The user is logged in with the role Logistics & Resources Officer.
2. The item is not reserved to a task.

**Post-conditions**
- *Success guarantee:* the item's current location is the destination station; the move is in the audit log.
- *Failure guarantee:* the location is unchanged.

**Basic Flow**
1. The Officer selects the item; the system shows its current location.
2. The Officer selects the destination station from the list of stations and confirms.
3. The system verifies that the item is Available (not reserved) and that the destination differs from the current location.
4. The system updates the item's location and writes the transfer to the audit log.
5. The system pushes the new location to the stations displaying the pool and displays a confirmation.

**Extensions**
- **3a (Exception flow) – The item is reserved to an Active task.**
  1. The system rejects the transfer and shows the holding task. The use case ends.
- **3b (Exception flow) – The destination equals the current location.**
  1. The system displays "Item is already at this station". The flow returns to step 2.

---

## UC-10 Update Crew Availability

| | |
|---|---|
| **Primary actor** | Logistics & Resources Officer |
| **Supporting actors** | – |
| **Trigger** | The Officer selects a crew member and chooses *Update availability*. |
| **Level / Complexity** | User goal / Low |

**Description.** The Officer records an exceptional unavailability of a crew member (leave, sickness) or ends it.

**Pre-conditions**
1. The user is logged in with the role Logistics & Resources Officer.

**Post-conditions**
- *Success guarantee:* the crew member's status is *Unavailable* (with reason and period) or *Available*; the change is in the audit log and pushed.
- *Failure guarantee:* the status is unchanged.

**Basic Flow**
1. The Officer selects the crew member; the system shows role, location and current status.
2. The Officer chooses *Unavailable* with a reason (leave / sickness) and an expected end date, or *Available*, and confirms.
3. The system verifies that the crew member is not currently reserved to a task.
4. The system updates the status and writes the change to the audit log.
5. The system pushes the update and displays a confirmation.

**Extensions**
- **2a (Exception flow) – The end date is earlier than today.**
  1. The system rejects the period. The flow returns to step 2.
- **3a (Exception flow) – The crew member (or Field Commander) is reserved to an Active task.**
  1. The system rejects the change and shows the task; the Dispatcher must first release or replace the resource. The use case ends.

---

## UC-11 Open Incident

| | |
|---|---|
| **Primary actor** | Dispatcher |
| **Supporting actors** | – |
| **Trigger** | The Dispatcher chooses *Open Incident*. |
| **Level / Complexity** | User goal / Low |

**Pre-conditions**
1. The user is logged in with the role Dispatcher.

**Post-conditions**
- *Success guarantee:* an incident with status *Open*, the entered details and an automatically recorded opening time exists; a live status screen for it is available to all Dispatchers; the opening is in the audit log.
- *Failure guarantee:* no incident is created.

**Basic Flow**
1. The Dispatcher chooses *Open Incident*.
2. The Dispatcher enters the incident name, selects the type from the closed list (Building Collapse, Fire, Road Accident, Hazmat Leak, Natural Disaster) and enters the location, then confirms.
3. The system validates that all fields are present and the type is in the closed list.
4. The system creates the incident with status *Open* and records the opening time from the server clock.
5. The system writes the opening to the audit log (incident, user, timestamp).
6. The system creates the live status screen of the incident and pushes it to all Dispatcher stations.
7. The system opens the new incident's status screen for the Dispatcher, who continues with UC-12 (Define Task).

**Extensions**
- **3a (Exception flow) – A mandatory field is missing.**
  1. The system highlights the field. The flow returns to step 2.
- **4a (Exception flow) – The database transaction fails.**
  1. The system displays "Incident could not be opened – please retry"; nothing is stored. The use case ends.

---

## UC-12 Define Task

| | |
|---|---|
| **Primary actor** | Dispatcher |
| **Supporting actors** | – |
| **Trigger** | The Dispatcher chooses *Add Task* on the status screen of an open incident. |
| **Level / Complexity** | User goal / High |
| **Relationships** | Extended by UC-14 Reject Cyclic Dependency at extension point *Dependency validation* (step 6). |

**Description.** The Dispatcher adds a task to the incident's task tree: name, estimated duration, prerequisite
tasks and the required resource package, which always contains exactly one Field Commander.

**Pre-conditions**
1. The user is logged in with the role Dispatcher.
2. The incident is *Open*.

**Post-conditions**
- *Success guarantee:* the task exists in the incident's task graph in state *Blocked* (open prerequisites) or *Ready* (no open prerequisites); the graph is still a DAG; the creation is in the audit log and pushed to the live status screens.
- *Failure guarantee:* no task is created and the dependency graph is unchanged.

**Basic Flow**
1. The Dispatcher chooses *Add Task* on the incident status screen.
2. The Dispatcher enters the task name and the estimated duration.
3. The Dispatcher selects zero or more prerequisite tasks from the incident's existing tasks.
4. The Dispatcher defines the required resource package: exactly one Field Commander plus the required roles and equipment types with quantities (for example: 1 Heavy Equipment Operator, 1 Bulldozer, 1 Paramedic).
5. The system validates the package: exactly one Field Commander, every role / type from the closed lists, every quantity ≥ 1.
6. The system validates that adding the task with its prerequisites keeps the incident's dependency graph acyclic (*extension point: Dependency validation*).
7. The system creates the task in state *Blocked* if at least one prerequisite is not *Completed*, otherwise directly in state *Ready*.
8. The system writes the creation and the initial state to the audit log and pushes the updated task tree to all stations displaying the incident.
9. The system shows the new task in the task tree.

**Extensions**
- **2a (Exception flow) – The estimated duration is missing or not positive.**
  1. The system highlights the field. The flow returns to step 2.
- **3a (Exception flow) – A selected prerequisite task is *Cancelled*.**
  1. The system rejects the selection ("A cancelled task can never be completed and cannot be a prerequisite"). The flow returns to step 3.
- **5a (Exception flow) – The package does not contain exactly one Field Commander.**
  1. The system rejects the package and explains the rule. The flow returns to step 4.
- **6a (Exception flow) – The new prerequisites would create a cycle.**
  1. UC-14 Reject Cyclic Dependency is performed; the task is not created. The flow returns to step 3.
- **7a (Exception flow) – The incident is no longer *Open* (cancelled or closed by another Dispatcher meanwhile).**
  1. The system rejects the creation and refreshes the screen with the incident's current status. The use case ends.

---

## UC-13 Modify Task Prerequisites

| | |
|---|---|
| **Primary actor** | Dispatcher |
| **Supporting actors** | – |
| **Trigger** | The Dispatcher selects a *Blocked* task and chooses *Edit prerequisites*. |
| **Level / Complexity** | User goal / Medium |
| **Relationships** | Extended by UC-14 Reject Cyclic Dependency at extension point *Dependency validation* (step 4). |

**Pre-conditions**
1. The user is logged in with the role Dispatcher.
2. The task is in state *Blocked* (dependencies are locked in *Ready*, *Active* and the final states).

**Post-conditions**
- *Success guarantee:* the task's prerequisite set is updated, the graph is still a DAG; if no open prerequisite remains the task is *Ready*; the change is in the audit log and pushed.
- *Failure guarantee:* the prerequisite set is unchanged.

**Basic Flow**
1. The Dispatcher selects a *Blocked* task and chooses *Edit prerequisites*.
2. The system shows the current prerequisites and the other tasks of the incident that may be selected.
3. The Dispatcher adds and/or removes prerequisite tasks and confirms.
4. The system verifies that the task is still *Blocked* and that the new prerequisite set keeps the graph acyclic (*extension point: Dependency validation*).
5. The system stores the new prerequisite set.
6. If every remaining prerequisite is *Completed*, the system moves the task to *Ready*.
7. The system writes the change (and the state transition, if any) to the audit log and pushes the updated tree.

**Extensions**
- **1a (Exception flow) – The task is not *Blocked*.**
  1. The system displays "Dependencies are locked for tasks in state Ready / Active / Completed / Cancelled". The use case ends.
- **3a (Exception flow) – A selected prerequisite is *Cancelled*.**
  1. The system rejects it (see UC-12 3a). The flow returns to step 3.
- **4a (Exception flow) – The new prerequisites create a cycle.**
  1. UC-14 Reject Cyclic Dependency is performed; nothing is stored. The flow returns to step 3.
- **4b (Exception flow) – The task left the *Blocked* state while the Dispatcher was editing (a prerequisite was completed and the server advanced the task to *Ready*).**
  1. The system rejects the change and refreshes the screen. The use case ends.

---

## UC-14 Reject Cyclic Dependency

| | |
|---|---|
| **Primary actor** | Dispatcher (receives the rejection) |
| **Supporting actors** | – |
| **Trigger** | During UC-12 or UC-13 the dependency validation detects a cycle. |
| **Level / Complexity** | Sub-function / Medium |
| **Relationships** | Extends UC-12 Define Task and UC-13 Modify Task Prerequisites (extension point *Dependency validation*). |

**Description.** Every addition or modification of a dependency is checked for cycles; an operation that would
create a cycle is rejected and the Dispatcher receives an explanatory message.

**Pre-conditions**
1. A task creation or prerequisite modification is being validated.
2. The requested prerequisite set would make a task depend (directly or indirectly) on itself.

**Post-conditions**
- *Success guarantee:* the operation is rejected, the dependency graph is unchanged and the Dispatcher sees the cycle.
- *Failure guarantee:* (same – the graph is never modified).

**Basic Flow**
1. The system performs a depth-first search over the incident's dependency graph including the requested change.
2. The system finds a path from the task back to itself (for example T3 → T1 → T2 → T3).
3. The system aborts the operation without writing anything to the database.
4. The system displays "Dependency rejected – it would create a cycle: T3 → T1 → T2 → T3".
5. The Dispatcher returns to the editing screen of the base use case with the previously entered data preserved.

**Extensions**
- **4a (Exception flow) – The Dispatcher's station disconnected before the message was delivered.**
  1. The message is queued and delivered with a full snapshot when the station reconnects; the graph remains unchanged. The use case ends.

---

## UC-15 Reserve Resource Package

| | |
|---|---|
| **Primary actor** | Dispatcher |
| **Supporting actors** | – |
| **Trigger** | The Dispatcher selects a *Ready* task on the live status screen and chooses *Reserve resources*. |
| **Level / Complexity** | User goal / High |
| **Relationships** | Included by UC-17 Auto-Allocate Resource Package. Extended by UC-16 Roll Back Conflicting Reservation at extension point *Availability re-check* (step 5). |

**Description.** The Dispatcher reserves the complete resource package of a *Ready* task in one atomic database
transaction. If any resource of the package is not available, nothing is reserved. Success moves the task to
*Active*.

**Pre-conditions**
1. The user is logged in with the role Dispatcher.
2. The incident is *Open* and the task is in state *Ready* (all prerequisites are *Completed*).
3. The task's required package is defined (exactly one Field Commander).

**Post-conditions**
- *Success guarantee:* every resource of the package has status *Reserved to task* with the task's identifier; the task is *Active* with a recorded start time; every change is in the audit log; all stations displaying the incident, the Field Commander and the crew members have been updated.
- *Failure guarantee:* no resource of the package is reserved (full rollback); the task remains *Ready*; the Dispatcher knows which resources were unavailable.

**Basic Flow**
1. The Dispatcher selects a *Ready* task on the incident's live status screen and chooses *Reserve resources*.
2. The system displays the package slots and, for each slot, the resources of the matching role / type whose status is *Available*, with their current location.
3. The Dispatcher selects exactly one resource per slot (including exactly one Field Commander) and confirms.
4. The system opens one SQL transaction and locks the rows of the selected resources with `SELECT … FOR UPDATE`, in ascending resource-ID order.
5. The system re-checks, inside the transaction, that every locked resource is still *Available* and matches its slot (*extension point: Availability re-check*).
6. The system sets every resource to *Reserved to task* with the task identifier and the reservation start time, and sets the task to *Active* with its start time.
7. The system writes an audit entry for each resource update and for the task transition (user, timestamp), and commits the transaction.
8. The system pushes the updated task and resource states to every station displaying the incident and notifies the assigned Field Commander and crew members.
9. The system shows the task as *Active* with its reserved resources on the Dispatcher's screen.

**Extensions**
- **1a (Exception flow) – The task is no longer *Ready* (advanced, cancelled or reserved by another Dispatcher meanwhile).**
  1. The system rejects the request and refreshes the screen with the task's current state. The use case ends.
- **3a (Exception flow) – A slot is left empty or a selected resource does not match the slot.**
  1. The system does not accept the confirmation and highlights the slot. The flow returns to step 3.
- **4a (Exception flow) – A lock cannot be obtained within 5 seconds (another station holds the rows).**
  1. The system aborts the transaction (nothing is reserved) and displays "Resources are being reserved by another station – please retry". The flow returns to step 2.
- **5a (Exception flow) – At least one resource is no longer *Available* (reserved concurrently by another Dispatcher, marked faulty, or marked unavailable).**
  1. UC-16 Roll Back Conflicting Reservation is performed: the whole transaction is rolled back.
  2. The Dispatcher may retry from step 2 with the refreshed availability.
- **7a (Exception flow) – The commit fails (database error).**
  1. The system rolls back the transaction, leaves the task *Ready* and displays "Reservation failed – no resource was reserved. Please retry". The use case ends.
- **8a (Exception flow) – A station that should receive the push is disconnected.**
  1. The server records the station as stale; when the station reconnects it requests a full snapshot of the incident and discards its cached state. The flow continues at step 9.

---

## UC-16 Roll Back Conflicting Reservation

| | |
|---|---|
| **Primary actor** | Dispatcher (receives the failure) |
| **Supporting actors** | – |
| **Trigger** | During UC-15 step 5 at least one locked resource is not *Available*. |
| **Level / Complexity** | Sub-function / Medium |
| **Relationships** | Extends UC-15 Reserve Resource Package (extension point *Availability re-check*). |

**Description.** Package reservation is "all or nothing": if two Dispatchers try to reserve the same bulldozer at the
same moment (one for an incident in Haifa, one for Tel Aviv) only one transaction commits; the other is rolled
back completely, and the losing Dispatcher is told clearly which resources were unavailable.

**Pre-conditions**
1. A package reservation transaction is open and holds row locks on the requested resources.
2. At least one of the locked resources is not in status *Available*.

**Post-conditions**
- *Success guarantee:* the transaction is rolled back; no resource of the package is reserved; the task remains *Ready*; the Dispatcher's screen shows the current availability; the rejected attempt is recorded.
- *Failure guarantee:* (same – no partial reservation can ever exist).

**Basic Flow**
1. The system identifies every resource of the package whose status is not *Available* and its current holder (task or status reason).
2. The system rolls back the whole transaction, releasing all row locks; no resource status was changed.
3. The system records the rejected attempt (task, resources, user, timestamp).
4. The system sends the Dispatcher a clear message: "Reservation failed – the following resources are no longer available: Bulldozer D-07 (reserved to task 'Debris removal', incident Haifa)".
5. The system refreshes the Dispatcher's status screen with the current state of every resource.
6. The Dispatcher returns to UC-15 step 2 to choose other resources, or starts UC-17 Auto-Allocate.

**Extensions**
- **4a (Exception flow) – The Dispatcher's station disconnected before the message was delivered.**
  1. The message is queued; on reconnection the station receives the message together with a full snapshot. The use case ends.

---

## UC-17 Auto-Allocate Resource Package

| | |
|---|---|
| **Primary actor** | Dispatcher |
| **Supporting actors** | – |
| **Trigger** | The Dispatcher selects a *Ready* task and chooses *Auto-allocate* with an allocation criterion. |
| **Level / Complexity** | User goal / High |
| **Relationships** | Includes UC-15 Reserve Resource Package (the atomic reservation, steps 4–8). Extended by UC-18 Retry with Next-Best Alternative at extension point *Reservation conflict* (step 5). |

**Description.** Instead of choosing every resource manually, the Dispatcher asks the system to fill the package
according to a selected criterion (for example: the available resources geographically closest to the incident
location). The criteria are pluggable so that new criteria can be added in the future.

**Pre-conditions**
1. The user is logged in with the role Dispatcher.
2. The incident is *Open* and the task is *Ready*.
3. At least one allocation criterion is configured.

**Post-conditions**
- *Success guarantee:* the complete package is reserved atomically (as in UC-15) with resources chosen by the criterion; the task is *Active*; the Dispatcher sees which resources were chosen and why.
- *Failure guarantee:* no resource is reserved; the task remains *Ready*; the Dispatcher knows which slot could not be filled.

**Basic Flow**
1. The Dispatcher selects a *Ready* task, chooses *Auto-allocate* and selects the criterion (default: nearest available resources to the incident location).
2. For every slot of the package the system builds a ranked candidate list of *Available* resources of the matching role / type, ordered by the criterion (for example by distance to the incident location).
3. The system picks the best-ranked candidate of every slot, including exactly one Field Commander.
4. The system performs Reserve Resource Package (**include UC-15**) with the resources selected in step 3 in place of the Dispatcher's manual selection.
5. The system verifies that the reservation committed (*extension point: Reservation conflict*).
6. The system pushes the updated states (UC-15 step 8) and shows the Dispatcher the allocated package with the criterion value of every resource (for example the distance in km).

**Extensions**
- **2a (Exception flow) – No *Available* candidate exists for at least one slot.**
  1. The system stops without reserving anything and reports the unfilled slot(s) ("No available Bulldozer in the pool"). The task remains *Ready*. The use case ends.
- **5a (Exception flow) – A selected resource was taken by a concurrent reservation (the included transaction was rolled back).**
  1. UC-18 Retry with Next-Best Alternative is performed automatically (up to 2 alternative attempts).
  2. If an alternative attempt commits, the flow continues at step 6.
- **5b (Exception flow) – The alternatives are exhausted (no further *Available* candidate for the conflicting slot, or 2 alternative attempts failed).**
  1. The system reports "Automatic allocation failed – no available alternative for slot Bulldozer"; nothing is reserved and the task remains *Ready*. The use case ends.

---

## UC-18 Retry with Next-Best Alternative

| | |
|---|---|
| **Primary actor** | Dispatcher (no interaction required) |
| **Supporting actors** | – |
| **Trigger** | The reservation transaction of an automatic allocation was rolled back because of a concurrent reservation. |
| **Level / Complexity** | Sub-function / Medium |
| **Relationships** | Extends UC-17 Auto-Allocate Resource Package (extension point *Reservation conflict*). |

**Description.** When two automatic allocations for different incidents select the same resource at the same time,
only one obtains it. The other continues automatically to the next-best available alternative (up to 2 alternative
attempts) without the Dispatcher repeating the request and without delaying the server for other users.

**Pre-conditions**
1. An automatic allocation is in progress and its reservation transaction was rolled back due to a conflict.
2. Fewer than 2 alternative attempts have been made for this request.

**Post-conditions**
- *Success guarantee:* the package is reserved with the alternative resource(s); the Dispatcher is informed which resources were substituted.
- *Failure guarantee:* no resource is reserved; the task remains *Ready*; the Dispatcher is informed that no alternative was available.

**Basic Flow**
1. The system identifies the slot(s) whose resource caused the conflict (from the rollback result).
2. The system keeps the non-conflicting selections and replaces each conflicting slot with the next candidate in its ranked list that is currently *Available*.
3. The system increments the attempt counter for this request.
4. The system re-executes the atomic reservation (UC-15 steps 4–7) in the same server worker thread, so that other client requests continue to be served.
5. The reservation commits; the system marks the substituted resources in the result shown to the Dispatcher ("Bulldozer D-07 was taken – D-12 allocated instead").

**Extensions**
- **2a (Exception flow) – The candidate list of the conflicting slot is exhausted.**
  1. The system stops; nothing is reserved; the Dispatcher is informed (UC-17 5b). The use case ends.
- **5a (Exception flow) – The alternative reservation is also rolled back due to a new conflict.**
  1. If the attempt counter is below 2, the flow returns to step 1.
  2. Otherwise the system stops; nothing is reserved; the Dispatcher is informed (UC-17 5b). The use case ends.

---

## UC-19 Assign Replacement Equipment

| | |
|---|---|
| **Primary actor** | Dispatcher |
| **Supporting actors** | – |
| **Trigger** | A *Replacement required* alert arrives after an equipment failure was reported from the field (UC-28). |
| **Level / Complexity** | User goal / Medium |

**Description.** When an equipment item breaks down during an active task, the item immediately becomes *Faulty* and
the Dispatcher must allocate a replacement item of the same type to the task.

**Pre-conditions**
1. The user is logged in with the role Dispatcher.
2. The task is *Active* and one of its equipment slots is marked *unfilled* because its item became *Faulty*.

**Post-conditions**
- *Success guarantee:* a replacement item of the same type is reserved to the task; the slot is filled; the change is in the audit log and pushed; the Field Commander is notified.
- *Failure guarantee:* no item is reserved; the slot remains flagged *unfilled* on the status screen.

**Basic Flow**
1. The Dispatcher opens the alert; the system shows the task, the failed item and the unfilled slot.
2. The system lists the *Available* items of the same equipment type, nearest to the incident first.
3. The Dispatcher selects an item (or chooses *Auto-select nearest*) and confirms.
4. The system opens a transaction, locks the item row (`SELECT … FOR UPDATE`), verifies that it is still *Available* and that the task is still *Active*.
5. The system sets the item to *Reserved to task*, links it to the slot, writes the audit entry and commits.
6. The system pushes the update to the stations displaying the incident and notifies the task's Field Commander and crew.
7. The system clears the alert and shows the filled slot.

**Extensions**
- **2a (Exception flow) – No *Available* item of the required type exists.**
  1. The system displays "No available replacement"; the slot stays flagged and highlighted on the status screen; the Dispatcher may wait for an item to be released or cancel the task (UC-21). The use case ends.
- **4a (Exception flow) – The selected item was reserved concurrently by another Dispatcher.**
  1. The system rolls back and refreshes the list. The flow returns to step 2.
- **4b (Exception flow) – The task reached a final state meanwhile (completed or cancelled).**
  1. The system rolls back, clears the alert and informs the Dispatcher. The use case ends.

---

## UC-20 Monitor Live Incident Status

| | |
|---|---|
| **Primary actor** | Dispatcher (all open incidents) / Field Commander (incidents in which he/she commands a task) |
| **Supporting actors** | – |
| **Trigger** | The user opens the live status screen of an open incident. |
| **Level / Complexity** | User goal / High |

**Description.** Every open incident has a live status screen. Every change of a task or resource is pushed from
the server to all relevant stations immediately (server push, no polling). The Dispatcher performs the incident
actions from this screen; the Field Commander completes his/her tasks from it.

**Pre-conditions**
1. The user is logged in as Dispatcher, or as the Field Commander of at least one task of the incident.
2. The incident is *Open*.

**Post-conditions**
- *Success guarantee:* the screen shows the current state of the incident and keeps reflecting every change within the push latency; no data is changed by viewing.
- *Failure guarantee:* the user is informed that the screen is not live (disconnected) and the display is marked stale.

**Basic Flow**
1. The user opens the status screen of an open incident.
2. The system verifies the user's authorisation for this incident and sends a full snapshot: incident name, type, location, elapsed time since opening; progress indicator (completed tasks out of all tasks); the task tree with its dependencies; and for each task its state (visually distinct for the five states), its Field Commander, its reserved resources, and the elapsed execution time versus the estimated duration.
3. The client renders the snapshot and starts the local elapsed-time counters.
4. Whenever a task or resource of the incident changes, the server pushes the change to the station and the client updates the affected element in place.
5. When a task's elapsed execution time exceeds its estimated duration, the client highlights the task as overdue.
6. The user performs actions from the screen: the Dispatcher reserves resources (UC-15 / UC-17), cancels tasks (UC-21), cancels or closes the incident (UC-23 / UC-24); the Field Commander reports completion of his/her tasks (UC-27).
7. The user closes the screen; the server stops pushing this incident to the station.

**Extensions**
- **2a (Exception flow) – The Field Commander is not assigned to any task of the incident.**
  1. The system denies access and shows only the incidents in which the user commands a task. The use case ends.
- **4a (Exception flow) – The connection to the server is lost.**
  1. The client shows a "Disconnected – data may be stale" banner and disables all actions.
  2. The client reconnects automatically (retry every 3 seconds); after reconnection it requests a fresh snapshot, discards the cached state and removes the banner. The flow continues at step 4.
- **4b (Alternative flow) – The incident is closed or cancelled while being viewed.**
  1. The system pushes the final status; the client marks the screen read-only with the final status and time. The use case ends.

---

## UC-21 Cancel Task

| | |
|---|---|
| **Primary actor** | Dispatcher |
| **Supporting actors** | – |
| **Trigger** | The Dispatcher selects a task in a non-final state and chooses *Cancel task*. |
| **Level / Complexity** | User goal / Medium |
| **Relationships** | Extended by UC-22 Cascade-Cancel Dependent Tasks at extension point *Dependent tasks found* (step 2). |

**Pre-conditions**
1. The user is logged in with the role Dispatcher.
2. The task is in state *Blocked*, *Ready* or *Active*.

**Post-conditions**
- *Success guarantee:* the task is *Cancelled*; its resources are *Available* again; every non-final task that depended on it directly or indirectly is also *Cancelled* with its resources released; everything is in the audit log and pushed.
- *Failure guarantee:* no task state or resource status is changed.

**Basic Flow**
1. The Dispatcher selects the task and chooses *Cancel task*.
2. The system computes the set of tasks that depend on this task, directly or indirectly, and are not in a final state (*extension point: Dependent tasks found*).
3. The system asks the Dispatcher to confirm the cancellation of the task (and of the dependent tasks, see UC-22).
4. The Dispatcher confirms.
5. The system, in one transaction, verifies that the task is still in a non-final state, sets it to *Cancelled*, releases every resource reserved to it (status *Available*, reservation end time recorded) and writes the audit entries.
6. The system commits and pushes the updated states to all stations displaying the incident, and notifies the released Field Commander and crew members.
7. The system shows the task as *Cancelled*.

**Extensions**
- **2a (Alternative flow) – At least one non-final dependent task exists.**
  1. UC-22 Cascade-Cancel Dependent Tasks is performed as part of steps 3–6.
- **4a (Alternative flow) – The Dispatcher declines the confirmation.**
  1. Nothing changes. The use case ends.
- **5a (Exception flow) – The task reached a final state meanwhile (for example its Field Commander reported completion).**
  1. The system rolls back, refreshes the screen with the current state and informs the Dispatcher. The use case ends.
- **5b (Exception flow) – The transaction fails.**
  1. The system rolls back; no task is cancelled; the Dispatcher is asked to retry. The use case ends.

---

## UC-22 Cascade-Cancel Dependent Tasks

| | |
|---|---|
| **Primary actor** | Dispatcher |
| **Supporting actors** | – |
| **Trigger** | A task being cancelled has non-final dependent tasks. |
| **Level / Complexity** | Sub-function / Medium |
| **Relationships** | Extends UC-21 Cancel Task (extension point *Dependent tasks found*). |

**Description.** Tasks that depend on a cancelled task (directly or indirectly) and have not reached a final state
are cancelled as well and their resources are released, after the Dispatcher approves a message that states how
many tasks will be cancelled.

**Pre-conditions**
1. UC-21 is in progress and the dependent-task set is not empty.

**Post-conditions**
- *Success guarantee:* every task of the set is *Cancelled* and its resources are released, in the same transaction as the base task.
- *Failure guarantee:* nothing is cancelled (the whole transaction, including the base task, is rolled back).

**Basic Flow**
1. The system displays the message "Cancelling this task will also cancel N dependent tasks: T4, T5, T7" and asks for approval.
2. The Dispatcher approves.
3. Inside the cancellation transaction of UC-21, the system sets every task of the set to *Cancelled* and releases every resource reserved to them.
4. The system writes an audit entry for every cancelled task and every released resource.
5. The transaction is committed with the base task (UC-21 step 6) and the whole sub-tree is pushed as *Cancelled*.

**Extensions**
- **2a (Alternative flow) – The Dispatcher does not approve.**
  1. The base task is not cancelled either; nothing changes. The use case ends (together with UC-21).
- **3a (Exception flow) – One of the dependent tasks reached a final state meanwhile.**
  1. The system removes it from the set (a *Completed* task stays *Completed*) and continues with the remaining tasks.

---

## UC-23 Cancel Incident

| | |
|---|---|
| **Primary actor** | Dispatcher |
| **Supporting actors** | – |
| **Trigger** | The Dispatcher chooses *Cancel incident* on an open incident (for example when a report turns out to be false). |
| **Level / Complexity** | User goal / Medium |

**Pre-conditions**
1. The user is logged in with the role Dispatcher.
2. The incident is *Open*.

**Post-conditions**
- *Success guarantee:* every task not yet in a final state is *Cancelled* and its resources are released; the incident is *Cancelled* with an automatically recorded cancellation time; everything is in the audit log and pushed; no billing request is sent.
- *Failure guarantee:* the incident and its tasks are unchanged.

**Basic Flow**
1. The Dispatcher chooses *Cancel incident* and enters the reason.
2. The system shows the number of non-final tasks that will be cancelled and the resources that will be released, and asks for confirmation.
3. The Dispatcher confirms.
4. The system, in one transaction, sets every non-final task to *Cancelled*, releases their resources, sets the incident to *Cancelled* and records the cancellation time from the server clock.
5. The system writes the audit entries (incident cancellation with the reason, every task transition and resource release) and commits.
6. The system pushes the final status to all stations displaying the incident, notifies the released Field Commanders and crew members, and removes the incident from the open-incidents list.

**Extensions**
- **1a (Exception flow) – The incident is already *Closed* or *Cancelled*.**
  1. The system rejects the request and shows the final status. The use case ends.
- **3a (Alternative flow) – The Dispatcher declines the confirmation.**
  1. Nothing changes. The use case ends.
- **4a (Exception flow) – The transaction fails.**
  1. The system rolls back everything; the incident remains *Open*; the Dispatcher is asked to retry. The use case ends.

---

## UC-24 Close Incident

| | |
|---|---|
| **Primary actor** | Dispatcher |
| **Supporting actors** | National Clearing & Billing System (through UC-25) |
| **Trigger** | The Dispatcher chooses *Close incident* on an open incident whose tasks are all in a final state. |
| **Level / Complexity** | User goal / High |
| **Relationships** | Includes UC-25 Settle Incident Billing. Extended by UC-26 Handle Billing Rejection at extension point *Settlement result* (step 7). |

**Description.** Only the Dispatcher may close an incident, and only after every task is *Completed* or
*Cancelled*. Closing computes the operating costs, sends a billing and budget-reimbursement request to the
external clearing system and, upon approval, records the transaction token and closes the incident.

**Pre-conditions**
1. The user is logged in with the role Dispatcher.
2. The incident is *Open* and every task is in a final state (*Completed* or *Cancelled*).

**Post-conditions**
- *Success guarantee:* the billing approval and transaction token are stored with the incident; the incident is *Closed* with an automatically recorded closing time; the closure and the settlement are in the audit log and pushed.
- *Failure guarantee:* the incident remains *Open*; no token is stored; the cost summary and the rejection / error reason are available to the Dispatcher for correction and retry; no duplicate billing request is created.

**Basic Flow**
1. The Dispatcher chooses *Close incident* on the incident's status screen.
2. The system verifies that every task of the incident is in a final state.
3. The system computes the operating costs: personnel hours (reservation time of every crew member × role rate), engine hours of the engineering and medical equipment (reservation time × type rate) and the consumables recorded at task completion.
4. The system displays the cost summary and asks for confirmation.
5. The Dispatcher confirms the closure.
6. The system sends the billing and reimbursement request to the National Clearing & Billing System (**include UC-25**).
7. The system receives the settlement result (*extension point: Settlement result*): approval with a transaction token.
8. The system, in one transaction, stores the approval and the token, sets the incident to *Closed*, records the closing time from the server clock and writes the audit entries.
9. The system pushes the final status to all stations displaying the incident and removes it from the open-incidents list.

**Extensions**
- **2a (Exception flow) – At least one task is not in a final state.**
  1. The system rejects the closure and lists the open tasks ("Complete or cancel these tasks first"). The use case ends.
- **5a (Alternative flow) – The Dispatcher declines the confirmation.**
  1. Nothing changes. The use case ends.
- **6a (Exception flow) – The clearing system does not answer within 10 seconds or a communication error occurs.**
  1. The incident remains *Open*; the system records the attempt and displays "Settlement pending – the clearing system did not respond. Retry later".
  2. A retry reuses the same request identifier (incident ID + attempt number), so the clearing system cannot bill the incident twice. The use case ends.
- **7a (Exception flow) – The clearing system rejects the settlement.**
  1. UC-26 Handle Billing Rejection is performed; the incident remains *Open*. The use case ends.
- **8a (Exception flow) – The database transaction fails after the approval was received.**
  1. The system rolls back and keeps the approval message in a pending queue; the closure is completed automatically on the next retry with the same token (no second billing request is sent). The use case ends.

---

## UC-25 Settle Incident Billing

| | |
|---|---|
| **Primary actor** | Dispatcher (indirectly – included by UC-24) |
| **Supporting actors** | National Clearing & Billing System |
| **Trigger** | UC-24 reaches the settlement step. |
| **Level / Complexity** | Sub-function / Medium |
| **Relationships** | Included by UC-24 Close Incident. |

**Description.** The system sends the incident's cost summary to the national clearing and billing system – an
external party representing the local authorities, the insurance companies and the Ministry of Finance – through
an adapter, and returns the result (approval + transaction token, or rejection) to the closing use case. In the
semester project the adapter is a stub that simulates the external system and can also simulate a rejection.

**Pre-conditions**
1. The cost summary of the incident is computed and confirmed by the Dispatcher.
2. The clearing adapter is configured.

**Post-conditions**
- *Success guarantee:* the settlement result (approval with token, or rejection with reason) is returned to UC-24 and recorded in the audit log.
- *Failure guarantee:* a time-out or communication error is returned to UC-24; no result is recorded as an approval.

**Basic Flow**
1. The system builds the billing request: incident identifier, type, opening and closing times, the cost breakdown (personnel hours, engine hours, consumables) and a unique request identifier (incident ID + attempt number).
2. The system sends the request through the clearing adapter.
3. The National Clearing & Billing System validates the request and returns an approval with a transaction token.
4. The system validates the response format and records the settlement (token, time) in the audit log.
5. The system returns the approval and the token to UC-24.

**Extensions**
- **3a (Exception flow) – The clearing system returns a rejection with a reason code.**
  1. The system records the rejection and returns it to UC-24 (which performs UC-26). The use case ends.
- **3b (Exception flow) – No response within 10 seconds.**
  1. The adapter returns a time-out to UC-24 (6a). The use case ends.
- **4a (Exception flow) – The response is malformed (no token).**
  1. The system treats the response as a communication error and returns it to UC-24 (6a). The use case ends.

---

## UC-26 Handle Billing Rejection

| | |
|---|---|
| **Primary actor** | Dispatcher |
| **Supporting actors** | National Clearing & Billing System |
| **Trigger** | The clearing system rejects the settlement of an incident. |
| **Level / Complexity** | Sub-function / Low |
| **Relationships** | Extends UC-24 Close Incident (extension point *Settlement result*). |

**Pre-conditions**
1. A settlement request was sent and a rejection was received.

**Post-conditions**
- *Success guarantee:* the incident remains *Open*; the rejection reason is stored and shown; the Dispatcher can correct the data and retry the closure.
- *Failure guarantee:* (same – a rejected incident is never closed).

**Basic Flow**
1. The system stores the rejection (reason code, message, time) with the incident and writes it to the audit log.
2. The system leaves the incident in status *Open*.
3. The system displays an alert to the Dispatcher: "Settlement rejected by the clearing system – reason: <reason>".
4. The Dispatcher reviews the cost summary, corrects what the reason requires (for example missing consumables data) and restarts UC-24.

**Extensions**
- **4a (Alternative flow) – The rejection cannot be resolved by the Dispatcher.**
  1. The Dispatcher escalates to the System Administrator; the incident remains *Open* and visible in the history with its rejection record. The use case ends.

---

## UC-27 Report Task Completion

| | |
|---|---|
| **Primary actor** | Field Commander |
| **Supporting actors** | – |
| **Trigger** | The Field Commander selects an *Active* task that he/she commands and chooses *Report completion*. |
| **Level / Complexity** | User goal / High |

**Description.** The Field Commander reports that the task is finished. The task becomes *Completed*, all its
resources are released immediately (back to *Available*) and the server advances every dependent task whose
prerequisites are now all completed from *Blocked* to *Ready*.

**Pre-conditions**
1. The user is logged in with the role Field Commander.
2. The task is *Active* and the logged-in user is the Field Commander reserved to this task.

**Post-conditions**
- *Success guarantee:* the task is *Completed* with a recorded completion time; every resource of the task is *Available* (reservation end time recorded); every dependent task whose prerequisites are all *Completed* is *Ready*; all transitions are in the audit log and pushed to the relevant stations.
- *Failure guarantee:* the task remains *Active* with its resources; the Field Commander is told why.

**Basic Flow**
1. The Field Commander opens the incident status screen and selects the *Active* task that he/she commands.
2. The Field Commander chooses *Report completion*, optionally records the consumables used and a short note, and confirms.
3. The system verifies that the task is *Active* and that the reporting user is the Field Commander reserved to it.
4. The system, in one transaction, sets the task to *Completed* with the completion time, records the consumables, and sets every resource reserved to the task (including the Field Commander) to *Available* with the reservation end time.
5. In the same transaction the system examines every task that depends on the completed task and moves each one whose prerequisites are now all *Completed* from *Blocked* to *Ready*.
6. The system writes an audit entry for the completion, every resource release and every *Blocked → Ready* transition, and commits.
7. The system pushes the updated states to every station displaying the incident and notifies the Dispatchers that new tasks are *Ready*.
8. The Field Commander sees the task marked *Completed* and is free to be reserved to another task.

**Extensions**
- **3a (Exception flow) – The reporting user is not the Field Commander assigned to the task.**
  1. The system rejects the report ("You may complete only tasks you command"), records the denied attempt and leaves the task unchanged. The use case ends.
- **3b (Exception flow) – The task is no longer *Active* (cancelled by the Dispatcher meanwhile).**
  1. The system rejects the report and refreshes the screen with the current state. The use case ends.
- **3c (Alternative flow) – The task was already reported *Completed* by this Field Commander (duplicate report, e.g. after a reconnection).**
  1. The system acknowledges without changing anything (idempotent report). The use case ends.
- **6a (Exception flow) – The transaction fails.**
  1. The system rolls back; the task remains *Active* with all its resources; the Field Commander is asked to retry. The use case ends.
- **7a (Exception flow) – A Dispatcher station is disconnected.**
  1. The station receives a fresh snapshot of the incident when it reconnects. The flow continues at step 8.

---

## UC-28 Report Equipment Failure

| | |
|---|---|
| **Primary actor** | Field Crew Member or Field Commander |
| **Supporting actors** | – |
| **Trigger** | A crew member or the Field Commander of an *Active* task selects a reserved equipment item and chooses *Report failure*. |
| **Level / Complexity** | User goal / Medium |

**Description.** When an item breaks down during an active task, the field reports it in real time; the item moves
immediately to *Faulty / Under repair* and the Dispatcher is required to allocate a replacement (UC-19).

**Pre-conditions**
1. The user is logged in as Field Crew Member or Field Commander and is reserved to the *Active* task.
2. The item is reserved to that task.

**Post-conditions**
- *Success guarantee:* the item is *Faulty / Under repair*; the task's slot is flagged *unfilled*; the failure is in the audit log; all stations displaying the incident are updated; every Dispatcher receives a *Replacement required* alert.
- *Failure guarantee:* the item's status is unchanged; the user is told why.

**Basic Flow**
1. The user opens the assigned task and selects the equipment item.
2. The user chooses *Report failure*, enters a short description and confirms.
3. The system verifies that the user is reserved to the task and that the item is reserved to the same task.
4. The system, in one transaction, sets the item to *Faulty / Under repair* immediately (the reservation link is kept for history and cost calculation, but the slot is flagged *unfilled*), and writes the audit entry.
5. The system pushes the update to all stations displaying the incident and sends a *Replacement required* alert to every Dispatcher station (UC-19).
6. The system confirms the report to the user.

**Extensions**
- **3a (Exception flow) – The item is not reserved to the user's task.**
  1. The system rejects the report. The use case ends.
- **3b (Alternative flow) – The item was already reported *Faulty* by another member of the same task.**
  1. The system acknowledges the report, adds the description to the existing failure record and does not send a second alert. The use case ends.
- **3c (Exception flow) – The task is no longer *Active*.**
  1. The system rejects the report and refreshes the screen. The use case ends.
- **5a (Exception flow) – The user's station loses the connection right after confirming.**
  1. The report was committed on the server; on reconnection the client receives the updated snapshot showing the item as *Faulty*. The use case ends.

---

## UC-29 Report Shift Readiness

| | |
|---|---|
| **Primary actor** | Field Crew Member |
| **Supporting actors** | – |
| **Trigger** | The crew member opens *My shift* and reports the start of a shift and readiness (available / not available). |
| **Level / Complexity** | User goal / Low |

**Pre-conditions**
1. The user is logged in with the role Field Crew Member.

**Post-conditions**
- *Success guarantee:* the crew member's status is *Available* (on shift) or *Unavailable*, with a timestamp; the change is in the audit log and pushed to the stations displaying the pool.
- *Failure guarantee:* the status is unchanged.

**Basic Flow**
1. The crew member opens *My shift*; the system shows the current status and location.
2. The crew member chooses *Start shift – available* (or *Not available*) and confirms the current location.
3. The system verifies that the change is allowed (see extensions).
4. The system updates the status and location, records the time and writes the audit entry.
5. The system pushes the update and displays a confirmation.

**Extensions**
- **3a (Exception flow) – The crew member was marked *Unavailable* by the Logistics Officer (leave / sickness).**
  1. The system rejects the change: "Your availability is managed by Logistics – contact the Logistics Officer". The use case ends.
- **3b (Exception flow) – The crew member is reserved to an *Active* task.**
  1. The system keeps the status *Reserved to task*: a *Not available* report is rejected until the task releases the resource, and an *Available* report is acknowledged without changing the status. The use case ends.

---

## UC-30 View Assigned Task

| | |
|---|---|
| **Primary actor** | Field Crew Member |
| **Supporting actors** | – |
| **Trigger** | The crew member opens *My task*. |
| **Level / Complexity** | User goal / Low |

**Pre-conditions**
1. The user is logged in with the role Field Crew Member.

**Post-conditions**
- *Success guarantee:* the details of the task to which the crew member is reserved are displayed and kept up to date by server push.
- *Failure guarantee:* the crew member is informed that he/she has no current assignment.

**Basic Flow**
1. The crew member opens *My task*.
2. The system finds the *Active* task to which the crew member is reserved.
3. The system displays the task details: incident name, type and location; task name and estimated duration; the Field Commander; the other crew members and the equipment of the package; the elapsed time.
4. The system pushes every change of the task (new replacement item, completion, cancellation) and the client updates the screen.
5. The crew member closes the screen.

**Extensions**
- **2a (Alternative flow) – The crew member is not reserved to any task.**
  1. The system displays "No current assignment". The use case ends.
- **4a (Alternative flow) – The task is completed or cancelled while being viewed.**
  1. The system shows "Task <name> is Completed / Cancelled – you are released" and clears the assignment. The use case ends.
- **4b (Exception flow) – The connection is lost.**
  1. The client shows a *Disconnected* banner, reconnects automatically and reloads the task. The flow continues at step 4.
