# CrisisSync — Assignment 1: Requirements and Acceptance Tests

**Group:** G18\
**Deliverable:** Part B — Requirements, Requirements Traceability Matrix (RTM), and BDD Acceptance Tests

## Team members

| Full name  | Student ID | College email |
| ---------- | ---------- | ------------- |
| bashar haj | 214547374       | Bashar.Haj@e.braude.ac.il       |
| ehsan shbetat | 325503514       | Ehsan.Shbetat@e.braude.ac.il       |
| silesty khateb | 314986688       | Silesty.khateb@e.braude.ac.il       |
| ayman serhan | 212814354      | ayman.serhan@e.braude.ac.il       |

## 1. Scope and assumptions

This document covers **only** the two operations designated in Assignment 1, Part B:

1. Reserving an entire resource package for a task, including reporting task completion.
2. Generating an SLA compliance report.

The source of functional behavior is the CrisisSync project story. Quantitative non-functional thresholds below are **proposed, testable design targets**, not numerical limits specified by the project story. They must be confirmed by the project team before submission.

Relevant Use Case names in the existing model: `Reserve Resource Package`, `Report Task Completion`, and `Generate SLA Compliance Report`. The `System Administrator` may generate a report on demand for a past month, while the `Month-End Scheduler` triggers monthly generation automatically.

## 2. Functional Requirements (FR)

### Resource reservation and task completion

| ID        | Requirement                                                                                                                                                                                                                                                                                       |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **FR-01** | The system **shall** allow the Dispatcher to request reservation of a required resource package only for a task in the `Ready` state, after all its prerequisites are completed.                                                                                                                  |
| **FR-02** | The system **shall** validate that all required personnel and equipment are available, including exactly one Field Commander for the task, before confirming a reservation.                                                                                                                       |
| **FR-03** | The system **shall** reserve every resource in the requested package within a single database transaction; if any resource is unavailable, it **shall** roll back the entire reservation, leave the task in `Ready`, and inform the Dispatcher.                                                   |
| **FR-04** | Following a successful reservation, the system **shall** mark the task `Active` and **shall** prevent any reserved resource from being assigned to another active task.                                                                                                                           |
| **FR-05** | The system **shall** allow only the Field Commander assigned to an active task to report its completion. It **shall** then mark the task `Completed`, immediately release its resources, and change each dependent `Blocked` task to `Ready` when all of that task's prerequisites are completed. |

### SLA compliance report

| ID        | Requirement                                                                                                                                                                                                   |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **FR-06** | The system **shall** automatically generate an SLA compliance report at the end of each calendar month and **shall** allow the System Administrator to generate a report for a selected past month on demand. |
| **FR-07** | The system **shall** calculate, separately for each incident type, the average elapsed time between incident opening and completion of the last task for the incidents included in the reporting month.       |
| **FR-08** | The system **shall** make the generated SLA compliance report available for the System Administrator to view, with a clear reporting month and incident-type grouping.                                        |

## 3. Non-Functional Requirements (NFR)

All performance targets apply to the local-area-network deployment described in the project story and to the test setup stated below. The numerical values are proposed acceptance criteria.

| ID         | Quality attribute         | Measurable requirement                                                                                                                                                                                                                                                          |
| ---------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **NFR-01** | Performance               | Under a test load of 20 simultaneous client sessions and 1,000 resources, the system **shall** respond to 95% of resource-reservation requests within **3 seconds**, measured from request submission to the success or failure response.                                       |
| **NFR-02** | Reliability / concurrency | In **100** repeated tests where two Dispatchers simultaneously request the same available resource for distinct tasks, the system **shall** produce **zero** double reservations, and every rejected package transaction **shall** leave **zero** partially reserved resources. |
| **NFR-03** | Security                  | The system **shall** reject **100%** of test attempts by a user without the required role to reserve resources, complete another commander's task, or request an SLA report; no rejected request **shall** change persistent data.                                              |
| **NFR-04** | Usability                 | In a moderated test with **5** users trained on the Dispatcher workflow, at least **4 of 5** users **shall** correctly identify a failed reservation and its unavailable resource within **30 seconds** of receiving the response, without assistance.                          |
| **NFR-05** | Report performance        | With **10,000** incident records in the test database, an on-demand SLA report for one past month **shall** be generated and displayed within **10 seconds** in at least **95%** of 20 measured runs.                                                                           |

## 4. Requirements Traceability Matrix (RTM)

Each requirement maps to an existing Use Case and at least one acceptance test in Section 5. Multiple requirements may share a test only where its assertions explicitly cover both.

| Requirement ID | Use Case in Visual Paradigm                                                      | Acceptance test ID(s) |
| -------------- | -------------------------------------------------------------------------------- | --------------------- |
| FR-01          | Reserve Resource Package                                                         | AT-01, AT-02          |
| FR-02          | Reserve Resource Package                                                         | AT-01, AT-03          |
| FR-03          | Reserve Resource Package                                                         | AT-03, AT-04          |
| FR-04          | Reserve Resource Package                                                         | AT-01, AT-04          |
| FR-05          | Report Task Completion                                                           | AT-05, AT-06          |
| FR-06          | Generate SLA Compliance Report                                                   | AT-07, AT-08 , AT-13  |
| FR-07          | Generate SLA Compliance Report                                                   | AT-07                 |
| FR-08          | Generate SLA Compliance Report                                                   | AT-07, AT-08          |
| NFR-01         | Reserve Resource Package                                                         | AT-09                 |
| NFR-02         | Reserve Resource Package                                                         | AT-04                 |
| NFR-03         | Reserve Resource Package; Report Task Completion; Generate SLA Compliance Report | AT-06, AT-10          |
| NFR-04         | Reserve Resource Package                                                         | AT-11                 |
| NFR-05         | Generate SLA Compliance Report                                                   | AT-12                 |

## 5. BDD Acceptance Tests (Given–When–Then)

### AT-01 — Successful resource-package reservation

**Given** a `Ready` task whose prerequisite tasks are all `Completed`\
**And** its required package contains exactly one Field Commander and all other required resources\
**And** every requested resource is available\
**When** an authorized Dispatcher reserves the package\
**Then** all package resources are reserved for that task\
**And** the task becomes `Active`\
**And** the Dispatcher receives a successful reservation confirmation.

### AT-02 — Reservation rejected for blocked task

**Given** a task in `Blocked` with an unfinished prerequisite\
**When** an authorized Dispatcher attempts to reserve its package\
**Then** the system rejects the request and explains that the task is not `Ready`\
**And** the task remains `Blocked`\
**And** no resource reservation is created.

### AT-03 — Entire package rolls back on unavailable resource

**Given** a `Ready` task requires one Field Commander, one medic, and one bulldozer\
**And** the bulldozer is already reserved for another task\
**When** an authorized Dispatcher attempts to reserve that package\
**Then** the system rejects the entire package reservation\
**And** neither the commander nor the medic is reserved for this task\
**And** the task remains `Ready`\
**And** the Dispatcher receives a message identifying the unavailable bulldozer and can retry later.

### AT-04 — Competing reservations do not double-book a resource

**Given** two `Ready` tasks in different incidents require the same available bulldozer\
**And** two authorized Dispatchers initiate their package requests simultaneously\
**When** the server processes both requests\
**Then** exactly one transaction reserves the bulldozer successfully\
**And** the other transaction fails and releases every resource it attempted to reserve\
**And** the losing Dispatcher receives the current resource status\
**And** repeating this test 100 times results in zero double reservations and zero partial reservations.

### AT-05 — Successful task completion and resource release

**Given** an `Active` task is assigned to Field Commander C\
**And** its resource package is reserved\
**And** a dependent task is `Blocked` solely by this active task\
**When** C reports the task as completed\
**Then** the system marks the task `Completed`\
**And** it releases the task's personnel and equipment immediately\
**And** it moves the dependent task to `Ready`\
**And** it records the completion in the audit history.

### AT-06 — Unauthorized task completion is rejected

**Given** an `Active` task is assigned to Field Commander C\
**And** Field Commander D is authenticated but is not assigned to that task\
**When** D attempts to report its completion\
**Then** the system rejects the operation\
**And** the task remains `Active`\
**And** its resources remain reserved.

### AT-07 — Correct on-demand SLA report

**Given** the System Administrator selects a past reporting month\
**And** the test dataset contains two qualifying `Fire` incidents with opening-to-last-task-completion durations of 60 and 120 minutes\
**And** it contains one qualifying `Road Accident` incident with a duration of 30 minutes\
**When** the administrator requests the SLA compliance report\
**Then** the report displays the selected month\
**And** it shows an average of **90 minutes** for `Fire`\
**And** it shows an average of **30 minutes** for `Road Accident`\
**And** it separates results by incident type.

### AT-08 — Scheduled generation and empty-month handling

**Given** the system clock reaches the end of the reporting month\
**And** the month contains no qualifying incidents\
**When** the Month-End Scheduler triggers report generation\
**Then** the system completes the generation without an exception\
**And** it produces an empty-results indication rather than a fabricated average\
**And** the System Administrator can view the report for that month.

### AT-09 — Resource-reservation response time

**Given** 20 active client sessions and 1,000 resources in the database\
**When** the test client submits 100 valid reservation requests and measures end-to-end response times\
**Then** at least 95 requests return a success or failure response within **3 seconds**.

### AT-10 — Role restrictions on protected operations

**Given** test accounts without the required roles are authenticated\
**When** each account attempts resource reservation, another commander's task completion, and SLA report generation\
**Then** 100% of the unauthorized attempts are rejected\
**And** no attempt changes persistent task, resource, or report data.

### AT-11 — Reservation-failure message usability

**Given** five trained participants perform the Dispatcher workflow\
**And** each participant receives a reservation rejection caused by a named unavailable resource\
**When** each participant is asked to identify the failure and the unavailable resource\
**Then** at least four participants identify both correctly within **30 seconds**, without assistance.

### AT-12 — SLA report response time

**Given** a test database of 10,000 incidents and an authorized System Administrator\
**When** the administrator generates a report for a selected past month 20 times and the elapsed time is measured\
**Then** at least 19 runs generate and display the report within **10 seconds**\
**And** a failure or timeout does not show a partial report as complete.

### AT-13 — SLA Report Timeout and Recovery

**Given** an authorized System Administrator requests an SLA compliance report for a past month\
**And** a temporary server failure causes the report generation to time out\
**When** the system detects the timeout\
**Then** the system shall notify the administrator that the report generation failed\
**And** the system shall not display or save an incomplete report as a completed report\
**And** existing report data shall remain unchanged\
**When** the server becomes available again and the administrator retries the request\
**Then** the system shall generate the complete SLA compliance report successfully.