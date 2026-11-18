# CrisisSync – Requirements Specification, Traceability Matrix & Acceptance Tests

**Group:** G18\
**Course:** Software Engineering Methods\
**System:** CrisisSync – Emergency Incident Management System\
**Assignment:** Assignment 1 – Part B (Requirements Engineering)\
**Submission date:** 19 November 2026

### Engineering Team

| # | Full Name | Student ID | Braude Email |
|---|-----------|------------|--------------|
| 1 | Bashar Haj | 214547374 | Bashar.Haj@e.braude.ac.il |
| 2 | Ehsan Shbetat | 325503514 | Ehsan.Shbetat@e.braude.ac.il |
| 3 | Silesty Khateb | 314986688 | Silesty.khateb@e.braude.ac.il |
| 4 | Ayman Serhan | 212814354 | ayman.serhan@e.braude.ac.il |

---

## 1. Scope and Notation

This document specifies the requirements and the acceptance tests of the two operations designated in
Assignment 1, Part B:

| Operation | Description | Realising use cases in the model |
|-----------|-------------|----------------------------------|
| **OP-1** | Reserving a resource package for a task – manually or by automatic allocation – including the Field Commander's report of the task's completion | Reserve Resource Package, Roll Back Conflicting Reservation, Auto-Allocate Resource Package, Retry with Next-Best Alternative, Report Task Completion |
| **OP-2** | Generating the SLA compliance report – automatically at the end of a month or on demand | Generate SLA Compliance Report, View Management Reports |

**Use case names.** Every use case named in this document is written exactly as it appears in the Visual
Paradigm model `G18_Assignment1.vpp`.

**Requirement syntax.** Every requirement is a single statement with the keyword **shall**, written in one
of three EARS templates:

| EARS pattern | Template | Used for |
|--------------|----------|----------|
| Ubiquitous | The *system* shall *response*. | Rules that always hold |
| Event-driven | **When** *trigger*, the *system* shall *response*. | Reactions to an event |
| Unwanted behaviour | **If** *undesired condition*, **then** the *system* shall *response*. | Errors, faults and conflicts |

**Terms.**

- *The server* is the CrisisSync server (business logic and database access). *A station* is a client
  workstation connected to the server over the local network.
- Task states: *Blocked*, *Ready*, *Active*, *Completed*, *Cancelled*. Resource statuses: *Available*,
  *Reserved to task*, *Unavailable* (personnel) and *Faulty / Under repair* (equipment).
- *A user action* is one mouse click or one keyboard shortcut.
- *Design assumption* marks a rule that the project story leaves open and that the team decided.

---

## 2. Functional Requirements (FR)

### 2.1 OP-1 – Reserve a resource package and report task completion

| ID | EARS pattern | Requirement | Source in project story |
|----|--------------|-------------|-------------------------|
| **FR-01** | Event-driven | When the Dispatcher submits a reservation request for a task in the *Ready* state, the server shall reserve all the resources selected for the task's package within a single SQL transaction. | Task life cycle – *Active*: the whole package is reserved in one transaction |
| **FR-02** | Unwanted behaviour | If any resource of the requested package is not in the *Available* status when the server checks availability inside the transaction, then the server shall roll back the whole reservation transaction so that no resource of the package is reserved. | Concurrency control – package reservation is all-or-nothing (rollback) |
| **FR-03** | Event-driven | When a reservation transaction is rolled back, the server shall send the requesting Dispatcher a failure message that identifies every unavailable resource of the package. | Concurrency control – the Dispatcher who failed receives a clear message |
| **FR-04** | Event-driven | When a reservation transaction is rolled back, the requesting Dispatcher's station shall display the current status of every resource of the requested package. | Concurrency control – the screen of the Dispatcher who failed is updated to the current state |
| **FR-05** | Event-driven | When a package reservation succeeds, the server shall change the state of the task from *Ready* to *Active*. | Task life cycle – *Active* |
| **FR-06** | Unwanted behaviour | If a reservation request refers to a task that is not in the *Ready* state, then the server shall reject the request without modifying any resource. | Dependencies – a task cannot reserve resources until all its prerequisites are completed |
| **FR-07** | Unwanted behaviour | If two or more reservation transactions request the same resource concurrently, then the server shall commit at most one of them. | Concurrency control – of two simultaneous reservations of the same resource only one succeeds |
| **FR-08** | Event-driven | When the Dispatcher requests automatic allocation for a *Ready* task with a selected allocation criterion, the server shall select for every package slot the best-ranked *Available* resource according to that criterion. | Automatic allocation – allocation of the package by a selected criterion |
| **FR-09** | Unwanted behaviour | If the reservation transaction of an automatic allocation is rolled back because a selected resource was reserved concurrently, then the server shall retry the reservation with the next-best *Available* alternative for the conflicting slot, up to 2 alternative attempts, without further Dispatcher input. | Automatic allocation – automatic continuation to the next-best available alternative (up to 2 alternative attempts) |
| **FR-10** | Event-driven | When the Field Commander reserved to an *Active* task reports the task's completion, the server shall change the state of the task to *Completed*. | Task life cycle – *Completed*: the Field Commander reports completion |
| **FR-11** | Unwanted behaviour | If a user who is not the Field Commander reserved to the task reports the task's completion, then the server shall reject the report. | Dual status of the Field Commander – may close only the tasks he/she commands |
| **FR-12** | Event-driven | When a task enters the *Completed* state, the server shall set the status of every resource reserved to that task to *Available*. | Task life cycle – *Completed*: the resources are released immediately |
| **FR-13** | Event-driven | When a task enters the *Completed* state, the server shall change every *Blocked* task whose prerequisite tasks are all *Completed* to the *Ready* state. | Task life cycle – *Completed*: the server advances the dependent tasks |
| **FR-14** | Ubiquitous | The server shall record every task state transition and every resource status change caused by a reservation or by a completion report in the audit log, together with the acting user and the server timestamp. | Incident history (audit log) |

### 2.2 OP-2 – SLA compliance report

| ID | EARS pattern | Requirement | Source in project story |
|----|--------------|-------------|-------------------------|
| **FR-15** | Event-driven | When the server clock passes the end of a calendar month, the server shall automatically generate the SLA Compliance Report for the month that has just ended. | Reports – the reports are generated automatically at the end of every month |
| **FR-16** | Event-driven | When the System Administrator requests the SLA Compliance Report for a selected completed past month, the server shall generate the report for that month on demand. | Implementation note 3 – on-demand report for a selected past month |
| **FR-17** | Ubiquitous | The server shall compute in the SLA Compliance Report, for each incident type, the average elapsed time from the incident's opening time to the completion time of its last task. | Reports – definition of the SLA report |
| **FR-18** | Ubiquitous | The server shall include in the SLA Compliance Report of a month only incidents that were not cancelled and whose last task reached the *Completed* state within that month. | Design assumption – an incident is counted in the month in which its last task was completed; a cancelled incident is not counted |
| **FR-19** | Unwanted behaviour | If no incident of a given incident type was completed in the reported month, then the server shall mark that incident type as "N/A" in the report instead of a numeric average. | Design assumption – an average over zero incidents is undefined |
| **FR-20** | Unwanted behaviour | If a report generation request refers to the current month or to a future month, then the server shall reject the request. | Implementation note 3 – on demand only for a past month |
| **FR-21** | Event-driven | When an SLA Compliance Report is generated, the server shall store the report together with its generation timestamp and its trigger (automatic or on demand). | Reports – the System Administrator views the generated reports |
| **FR-22** | Unwanted behaviour | If a user whose role is not System Administrator requests the generation of an SLA Compliance Report, then the server shall reject the request. | Users and permissions – management reports belong to the System Administrator |
| **FR-23** | Unwanted behaviour | If the generation of an SLA Compliance Report fails, then the server shall discard the partial results so that no partial report is stored. | Design assumption – a failed generation leaves no partial report |
| **FR-24** | Unwanted behaviour | If an on-demand generation of an SLA Compliance Report fails, then the server shall notify the requesting System Administrator that the generation failed. | Design assumption – the Administrator is told about a failed generation |

---

## 3. Non-Functional Requirements (NFR)

Every non-functional requirement states a measurable numeric threshold (in bold).

| ID | Category | EARS pattern | Requirement (with measurable threshold) |
|----|----------|--------------|------------------------------------------|
| **NFR-01** | Performance | Ubiquitous | The server shall complete a package reservation transaction – from receipt of the request to commit – within **2.0 seconds** for **95 %** of the requests while **50** Dispatcher stations are connected. |
| **NFR-02** | Performance | Ubiquitous | The server shall generate the SLA Compliance Report of a month that contains up to **10,000** incidents within **30 seconds**. |
| **NFR-03** | Security | Ubiquitous | The server shall reject **100 %** of the reservation, completion-report and SLA-report-generation requests that are sent by a session without the required role or identity, including requests that bypass the client GUI. |
| **NFR-04** | Reliability | Ubiquitous | The server shall reserve a resource to at most one of two concurrent reservation transactions that request it, with **0** double reservations in **1,000** tested pairs of transactions. |
| **NFR-05** | Reliability | Unwanted behaviour | If a lock required by a reservation transaction is not acquired within **5 seconds**, then the server shall abort the transaction so that every lock it holds is released. |
| **NFR-06** | Reliability | Ubiquitous | The server shall commit a task state transition or a resource status change only together with its audit entry, so that **100 %** of the committed changes have an audit entry. |
| **NFR-07** | Reliability | Unwanted behaviour | If the automatic month-end generation of the SLA Compliance Report fails, then the server shall retry the generation up to **3** times at **10-minute** intervals. |
| **NFR-08** | Usability | Ubiquitous | The system shall allow the Dispatcher to reserve the complete package of a *Ready* task from the live status screen with at most **N + 3** user actions, where N is the number of package slots: select the task, choose *Reserve resources*, one selection per slot, and confirm. |
| **NFR-09** | Usability | Event-driven | When a reservation transaction is rolled back, the requesting Dispatcher's station shall display the failure message within **1.0 second** of the rollback. |

---

## 4. Requirements Traceability Matrix (RTM)

Every requirement is mapped to the use case that realises it in the model and to the acceptance test of
Section 5 that verifies it. The Test IDs are identical in this matrix and in Section 5.

| Req ID | Requirement (short form) | Type | Use Case in the model | Test ID |
|--------|--------------------------|:----:|-----------------------|:-------:|
| FR-01 | Package reserved in a single SQL transaction | FR | Reserve Resource Package | TEST-01 |
| FR-02 | Unavailable resource ⇒ full rollback, nothing reserved | FR | Roll Back Conflicting Reservation | TEST-02 |
| FR-03 | Failure message identifies every unavailable resource | FR | Roll Back Conflicting Reservation | TEST-03 |
| FR-04 | Dispatcher's screen shows the current resource statuses after a rollback | FR | Roll Back Conflicting Reservation | TEST-04 |
| FR-05 | Successful reservation ⇒ task Ready → Active | FR | Reserve Resource Package | TEST-05 |
| FR-06 | Reservation rejected for a task that is not Ready | FR | Reserve Resource Package | TEST-06 |
| FR-07 | Concurrent requests for one resource ⇒ at most one commits | FR | Reserve Resource Package; Roll Back Conflicting Reservation | TEST-07 |
| FR-08 | Auto-allocation selects the best-ranked available resource per slot | FR | Auto-Allocate Resource Package | TEST-08 |
| FR-09 | Conflict ⇒ automatic retry with the next-best alternative (≤ 2) | FR | Retry with Next-Best Alternative | TEST-09 |
| FR-10 | Field Commander's report ⇒ task Completed | FR | Report Task Completion | TEST-10 |
| FR-11 | Completion report by another user rejected | FR | Report Task Completion | TEST-11 |
| FR-12 | Completed ⇒ all resources of the task Available | FR | Report Task Completion | TEST-12 |
| FR-13 | Completed ⇒ dependent tasks with all prerequisites completed become Ready | FR | Report Task Completion | TEST-13 |
| FR-14 | Transitions and status changes recorded in the audit log with user and timestamp | FR | Reserve Resource Package; Report Task Completion | TEST-14 |
| FR-15 | SLA report generated automatically at month end | FR | Generate SLA Compliance Report | TEST-15 |
| FR-16 | SLA report generated on demand for a past month | FR | Generate SLA Compliance Report | TEST-16 |
| FR-17 | Average time from opening to last task completion, per incident type | FR | Generate SLA Compliance Report | TEST-17 |
| FR-18 | Only non-cancelled incidents completed within the month are included | FR | Generate SLA Compliance Report | TEST-18 |
| FR-19 | Incident type without completed incidents marked "N/A" | FR | Generate SLA Compliance Report | TEST-19 |
| FR-20 | Current or future month rejected | FR | Generate SLA Compliance Report | TEST-20 |
| FR-21 | Report stored with generation timestamp and trigger | FR | Generate SLA Compliance Report; View Management Reports | TEST-21 |
| FR-22 | Generation request by a non-Administrator rejected | FR | Generate SLA Compliance Report | TEST-22 |
| FR-23 | Failed generation ⇒ no partial report stored | FR | Generate SLA Compliance Report | TEST-23 |
| FR-24 | Failed on-demand generation ⇒ Administrator notified | FR | Generate SLA Compliance Report | TEST-24 |
| NFR-01 | Reservation transaction ≤ 2.0 s for 95 % of requests with 50 stations | NFR | Reserve Resource Package | TEST-25 |
| NFR-02 | SLA report for 10,000 incidents ≤ 30 s | NFR | Generate SLA Compliance Report | TEST-26 |
| NFR-03 | 100 % of unauthorised requests rejected on the server | NFR | Reserve Resource Package; Report Task Completion; Generate SLA Compliance Report | TEST-27 |
| NFR-04 | 0 double reservations in 1,000 concurrent pairs | NFR | Reserve Resource Package; Roll Back Conflicting Reservation | TEST-28 |
| NFR-05 | Lock not acquired within 5 s ⇒ transaction aborted | NFR | Reserve Resource Package | TEST-29 |
| NFR-06 | 100 % of committed changes have an audit entry | NFR | Reserve Resource Package; Report Task Completion | TEST-30 |
| NFR-07 | Failed scheduled generation retried up to 3 × at 10-minute intervals | NFR | Generate SLA Compliance Report | TEST-31 |
| NFR-08 | Package reserved in at most N + 3 user actions | NFR | Reserve Resource Package | TEST-32 |
| NFR-09 | Failure message displayed ≤ 1.0 s after the rollback | NFR | Roll Back Conflicting Reservation | TEST-33 |

---

## 5. BDD Acceptance Tests (Given – When – Then)

**Test data.** The incident "Carmel Fire" (type *Fire*, Haifa) has the tasks T1 "Evacuation" and
T2 "Debris removal"; T2 depends on T1. The package of T2 has four slots, filled in the tests by the Field
Commander FC-3, the Heavy Equipment Operator EO-7, the Bulldozer D-07 and the Paramedic PM-2. A second
incident, "Tel Aviv Collapse" (type *Building Collapse*), is used in the concurrency tests. ST-A and ST-B
are two Dispatcher stations.

**Scenario coverage.**

| Operation | Happy-path scenarios | Exception, error and time-out scenarios |
|-----------|----------------------|------------------------------------------|
| OP-1 | TEST-01, TEST-05, TEST-08, TEST-10, TEST-12, TEST-13, TEST-14 | TEST-02, TEST-03, TEST-04, TEST-06, TEST-07, TEST-09, TEST-11, TEST-29, TEST-30, TEST-33 |
| OP-2 | TEST-15, TEST-16, TEST-17, TEST-18, TEST-21 | TEST-19, TEST-20, TEST-22, TEST-23, TEST-24, TEST-31 |

The remaining tests (TEST-25, TEST-26, TEST-27, TEST-28 and TEST-32) measure non-functional thresholds.
System recovery after a failure is checked in TEST-04, TEST-09 and TEST-29 for OP-1 and in TEST-31 for OP-2.

### 5.1 OP-1 – Reserve a resource package and report task completion

#### Acceptance Test [TEST-01]: Package reserved in one transaction (Happy Path)
* **Given:** T2 is in state *Ready* and FC-3, EO-7, D-07 and PM-2 are all *Available*.
* **When:** The Dispatcher at ST-A selects T2, chooses *Reserve resources*, selects the four resources and confirms.
* **Then:** The server executes exactly one database transaction that updates all four resource records.
* **And:** After the commit each of the four resources has the status *Reserved to task* T2.

#### Acceptance Test [TEST-02]: Resource becomes unavailable – full rollback (Exception)
* **Given:** T2 is *Ready* and the reservation screen at ST-A lists FC-3, EO-7, D-07 and PM-2 as *Available*; before the Dispatcher confirms, the Logistics & Resources Officer sets D-07 to *Faulty / Under repair*.
* **When:** The Dispatcher at ST-A confirms the package that contains D-07.
* **Then:** The server rolls back the reservation transaction, and none of FC-3, EO-7, PM-2 and D-07 is reserved.
* **And:** T2 remains in state *Ready*.

#### Acceptance Test [TEST-03]: Failure message names the unavailable resources (Exception)
* **Given:** After the reservation screen at ST-A listed them as *Available*, D-07 was reserved to the task "Access road clearing" and PM-2 was set to *Unavailable* (sickness); the Dispatcher at ST-A then confirmed the package of T2, and the reservation transaction is rolled back.
* **When:** The rollback completes.
* **Then:** The Dispatcher receives the message "Reservation failed – the following resources are no longer available: Bulldozer D-07 (reserved to task 'Access road clearing'), Paramedic PM-2 (unavailable)".
* **And:** The message names neither FC-3 nor EO-7.

#### Acceptance Test [TEST-04]: Screen refreshed with the current availability (Exception and Recovery)
* **Given:** The reservation of T2 submitted from ST-A is rolled back because D-07 was reserved concurrently from ST-B.
* **When:** The rollback completes.
* **Then:** The status screen at ST-A shows D-07 as *Reserved to task* and FC-3, EO-7 and PM-2 as *Available*.
* **And:** The screen still shows T2 as *Ready*, so the Dispatcher can choose another bulldozer and reserve again.

#### Acceptance Test [TEST-05]: Task becomes Active when the reservation succeeds (Happy Path)
* **Given:** T2 is *Ready* and the Dispatcher has confirmed a package of four *Available* resources.
* **When:** The reservation transaction commits.
* **Then:** T2 is in state *Active*.
* **And:** The Dispatcher's status screen shows T2 as *Active*.

#### Acceptance Test [TEST-06]: Reservation request for a task that is not Ready (Exception)
* **Given:** T2 is *Blocked* because its prerequisite T1 is still *Active*; FC-3, EO-7, D-07 and PM-2 are *Available*.
* **When:** A reservation request for T2 with these four resources reaches the server.
* **Then:** The server rejects the request.
* **And:** FC-3, EO-7, D-07 and PM-2 are still *Available* and T2 is still *Blocked*.

#### Acceptance Test [TEST-07]: Two Dispatchers reserve the same bulldozer (Race Condition)
* **Given:** D-07 is *Available*; within the same 100 ms the Dispatcher at ST-A confirms a package that contains D-07 for the incident "Carmel Fire" and the Dispatcher at ST-B confirms a package that contains D-07 for the incident "Tel Aviv Collapse".
* **When:** The server executes both reservation transactions.
* **Then:** Exactly one of the two transactions commits.
* **And:** D-07 is reserved to exactly one task.

#### Acceptance Test [TEST-08]: Automatic allocation by nearest resources (Happy Path)
* **Given:** T2 is *Ready*; two *Available* bulldozers exist – D-07 at 4 km and D-12 at 15 km from the incident location – and every other slot of the package has at least one *Available* candidate.
* **When:** The Dispatcher chooses *Auto-allocate* for T2 with the criterion "nearest available resources".
* **Then:** The server selects D-07 for the Bulldozer slot and, for every other slot, the *Available* candidate nearest to the incident location.
* **And:** D-12 is not selected and remains *Available*.

#### Acceptance Test [TEST-09]: Automatic allocation retries with the next-best alternative (Recovery)
* **Given:** Automatic allocations for "Carmel Fire" and for "Tel Aviv Collapse" both rank the bulldozer D-07 first and D-12 second; D-12 is *Available*.
* **When:** Both allocations run at the same time and the reservation for "Tel Aviv Collapse" is rolled back because D-07 was reserved for "Carmel Fire".
* **Then:** The server re-executes the reservation for "Tel Aviv Collapse" with D-12 in the Bulldozer slot, without any further Dispatcher input.
* **And:** The reservation commits at the first alternative attempt, within the limit of 2 alternative attempts.

#### Acceptance Test [TEST-10]: Field Commander reports completion (Happy Path)
* **Given:** T2 is *Active*, FC-3 is the Field Commander reserved to it, and FC-3 is logged in.
* **When:** FC-3 selects T2 on the incident status screen and chooses *Report completion*.
* **Then:** T2 is in state *Completed*.

#### Acceptance Test [TEST-11]: Completion report by a user who is not the task's Field Commander (Exception)
* **Given:** T2 is *Active* with FC-3 as its Field Commander; the Field Commander FC-5 is logged in and is not reserved to T2.
* **When:** FC-5 sends a completion report for T2.
* **Then:** The server rejects the report with the message "You may complete only tasks you command".
* **And:** T2 remains *Active* and its resources remain reserved.

#### Acceptance Test [TEST-12]: Resources released on completion (Happy Path)
* **Given:** T2 is *Active* with FC-3, EO-7, D-07 and PM-2 reserved to it.
* **When:** FC-3 reports the completion of T2.
* **Then:** FC-3, EO-7, D-07 and PM-2 all have the status *Available*.
* **And:** All four appear as *Available* when a Dispatcher opens the reservation of another *Ready* task.

#### Acceptance Test [TEST-13]: Dependent tasks advance to Ready (Happy Path)
* **Given:** T1 is *Active*; T2 is *Blocked* with T1 as its only prerequisite; T3 is *Blocked* with the prerequisites T1 and T4, and T4 is still *Ready*.
* **When:** The Field Commander of T1 reports its completion.
* **Then:** T2 changes from *Blocked* to *Ready*.
* **And:** T3 remains *Blocked* because T4 is not *Completed*.

#### Acceptance Test [TEST-14]: Audit log records the transitions (Happy Path)
* **Given:** The Dispatcher "dana" is logged in at ST-A; T2 is *Ready* and FC-3, EO-7, D-07 and PM-2 are *Available*.
* **When:** "dana" reserves the package of T2 and the transaction commits.
* **Then:** The audit log contains one entry for the transition *Ready → Active* of T2 and one entry for the status change of each of the four resources.
* **And:** Each of the five entries carries the user "dana" and the server timestamp of the change.

### 5.2 OP-2 – SLA compliance report

#### Acceptance Test [TEST-15]: Automatic month-end generation (Happy Path)
* **Given:** The server clock shows 31 October 2026, 23:59:59, and no SLA Compliance Report exists for October 2026.
* **When:** The server clock passes 1 November 2026, 00:00:00.
* **Then:** The server generates the SLA Compliance Report for October 2026 without any user action.

#### Acceptance Test [TEST-16]: On-demand generation for a past month (Happy Path)
* **Given:** The System Administrator is logged in and the current month is November 2026.
* **When:** The Administrator selects "August 2026" and chooses *Generate SLA Compliance Report*.
* **Then:** The server generates the SLA Compliance Report for August 2026.
* **And:** The report is displayed to the Administrator.

#### Acceptance Test [TEST-17]: Average time per incident type (Happy Path)
* **Given:** In October 2026 two *Fire* incidents were completed – one opened at 08:00 whose last task was completed at 11:00 (3 h), and one opened at 20:00 whose last task was completed at 01:00 on the next day (5 h) – and one *Road Accident* incident was completed, opened at 14:00 with its last task completed at 14:30 (30 min).
* **When:** The SLA Compliance Report for October 2026 is generated.
* **Then:** The report shows an average of 4 h 00 min for the type *Fire*.
* **And:** The report shows an average of 0 h 30 min for the type *Road Accident*.

#### Acceptance Test [TEST-18]: Only non-cancelled incidents completed in the month are counted (Happy Path)
* **Given:** Three *Fire* incidents exist: the last task of incident A was completed on 30 October 2026 and A was closed on the same day; incident B was opened on 29 October 2026 and its last task was completed on 2 November 2026; incident C was cancelled on 15 October 2026.
* **When:** The SLA Compliance Report for October 2026 is generated.
* **Then:** The *Fire* average of October 2026 is computed from incident A only.
* **And:** Incidents B and C do not contribute to the report of October 2026 (B belongs to the report of November 2026).

#### Acceptance Test [TEST-19]: Incident type without completed incidents (Exception)
* **Given:** No *Hazmat Leak* incident was completed in October 2026, and two *Fire* incidents were completed in October 2026.
* **When:** The SLA Compliance Report for October 2026 is generated.
* **Then:** The report shows "N/A" for the type *Hazmat Leak*.
* **And:** The report shows a numeric average for the type *Fire*.

#### Acceptance Test [TEST-20]: Request for the current month (Exception)
* **Given:** The current month is November 2026 and the System Administrator is logged in.
* **When:** The Administrator requests the SLA Compliance Report for November 2026.
* **Then:** The server rejects the request with the message "Reports can be generated only for completed months".
* **And:** No SLA Compliance Report is stored for November 2026.

#### Acceptance Test [TEST-21]: Report stored with timestamp and trigger (Happy Path)
* **Given:** The System Administrator generated the SLA Compliance Report for September 2026 on demand on 8 November 2026 at 09:30.
* **When:** The Administrator opens the *Reports* screen.
* **Then:** The list shows the SLA Compliance Report of September 2026 with the generation time 8 November 2026, 09:30 and the trigger "on demand".
* **And:** Selecting the entry displays the stored report.

#### Acceptance Test [TEST-22]: Generation request by a Dispatcher (Exception)
* **Given:** A user with the role Dispatcher is logged in.
* **When:** The user's station sends a request to generate the SLA Compliance Report for October 2026.
* **Then:** The server rejects the request.
* **And:** The request generates and stores no report.

#### Acceptance Test [TEST-23]: Failed generation leaves no partial report (Exception)
* **Given:** No SLA Compliance Report is stored for August 2026, and a fault injector makes the database unavailable while the server computes the report for August 2026.
* **When:** The generation fails.
* **Then:** No SLA Compliance Report is stored for August 2026.
* **And:** The reports that were stored before the failure are unchanged.

#### Acceptance Test [TEST-24]: Administrator notified of a failed on-demand generation (Exception)
* **Given:** The System Administrator requests the SLA Compliance Report for August 2026, and the database becomes unavailable during the computation.
* **When:** The on-demand generation fails.
* **Then:** The Administrator's station displays the message "Report generation failed – please retry".
* **And:** The action *Generate SLA Compliance Report* stays available, so the Administrator can repeat the request.

### 5.3 Non-functional acceptance tests

#### Acceptance Test [TEST-25]: Reservation time under load (Performance)
* **Given:** 50 Dispatcher stations are connected and a load generator submits 1,000 valid package reservation requests.
* **When:** The duration of every reservation transaction, from receipt of the request to commit, is measured on the server.
* **Then:** At least 95 % of the measured durations are at most 2.0 seconds.

#### Acceptance Test [TEST-26]: SLA report generation time (Performance)
* **Given:** The database contains 10,000 incidents whose last task was completed in March 2026.
* **When:** The System Administrator requests the SLA Compliance Report for March 2026.
* **Then:** The server generates the report within 30 seconds.

#### Acceptance Test [TEST-27]: Server-side authorisation (Security)
* **Given:** A test client with an authenticated Field Crew Member session sends 100 request messages directly to the server, bypassing the client GUI: package reservation requests, completion reports for T2 and requests to generate the SLA Compliance Report.
* **When:** The messages reach the server.
* **Then:** The server rejects 100 of the 100 requests.
* **And:** No task, resource or report record is changed.

#### Acceptance Test [TEST-28]: No double reservation under concurrency (Reliability)
* **Given:** A test harness runs 1,000 pairs of concurrent reservation transactions; the two transactions of each pair request the same *Available* resource.
* **When:** All 2,000 transactions have ended.
* **Then:** In each of the 1,000 pairs the resource is reserved to exactly one of the two tasks – 0 double reservations.

#### Acceptance Test [TEST-29]: Lock wait time-out (Reliability – Time-out and Recovery)
* **Given:** A test transaction holds the lock on the record of D-07 for 20 seconds without committing.
* **When:** The Dispatcher at ST-B confirms a package that contains D-07.
* **Then:** The server aborts the transaction of ST-B 5 seconds after it requested the lock, and none of the resources selected at ST-B is reserved.
* **And:** The other resources selected at ST-B can be reserved from a third station immediately after the abort.

#### Acceptance Test [TEST-30]: No state change without an audit entry (Reliability – Fault)
* **Given:** T2 is *Active*, and a fault injector makes the writing of audit entries fail.
* **When:** FC-3 reports the completion of T2.
* **Then:** No change is committed: T2 remains *Active* and its resources remain reserved.
* **And:** FC-3 is asked to repeat the report.

#### Acceptance Test [TEST-31]: Scheduled generation failure and recovery (Reliability – Recovery)
* **Given:** The database is unavailable from 31 October 2026, 23:59 until 1 November 2026, 00:15.
* **When:** The server clock passes 1 November 2026, 00:00:00.
* **Then:** The automatic generation fails at 00:00, the first retry at 00:10 fails, and the second retry at 00:20 generates the SLA Compliance Report for October 2026.
* **And:** Exactly one SLA Compliance Report is stored for October 2026.

#### Acceptance Test [TEST-32]: Reservation in a bounded number of actions (Usability)
* **Given:** The Dispatcher views the live status screen of "Carmel Fire"; T2 is *Ready* and its package has 4 slots.
* **When:** A usability tester counts every user action that the Dispatcher needs to reserve the package of T2.
* **Then:** The count is at most 7 (N + 3 with N = 4): select T2, choose *Reserve resources*, one selection for each of the 4 slots, and confirm.
* **And:** The Dispatcher opens no other screen during the reservation.

#### Acceptance Test [TEST-33]: Prompt failure message (Usability – Exception)
* **Given:** The Dispatcher at ST-A confirms a package that contains a resource reserved meanwhile from ST-B, and the server logs the time of the rollback.
* **When:** The server rolls back the transaction of ST-A.
* **Then:** The failure message appears on the screen of ST-A within 1.0 second of the logged rollback time.
