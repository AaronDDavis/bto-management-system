# BTO Management System - Technical Documentation

![Java](https://img.shields.io/badge/Java-SE%208+-orange?style=for-the-badge&logo=openjdk&logoColor=white)
![Architecture](https://img.shields.io/badge/Architecture-ECB-blue?style=for-the-badge)
![Pattern](https://img.shields.io/badge/Pattern-Two--Pass%20Hydration-blueviolet?style=for-the-badge)
![Persistence](https://img.shields.io/badge/Persistence-In--Memory%20Graph-green?style=for-the-badge)

> **A Console-Based HDB Build-To-Order Management System with Two-Phase Reference Resolution**

**File-persisted application demonstrating Entity-Control-Boundary architecture, role-based state machines, and in-memory relational graph construction from CSV.**

This project simulates the end-to-end workflow of Singapore’s BTO housing process, modelling applicants, officers, and managers under real-world policy constraints. The system emphasises architectural clarity over external dependencies, demonstrating how complex relational state can be managed in-memory using structured design patterns.

---

## 📖 Table of Contents

- [✨ Core Engineering Contributions](#-core-engineering-contributions)
- [🏗️ Architecture: Entity-Control-Boundary](#️-architecture-entity-control-boundary)
- [💾 Two-Pass Hydration Model](#-two-pass-hydration-model)
- [🔄 State Machines](#-state-machines)
- [👥 Authority Model](#-authority-model)
- [🛠️ Tech Stack](#️-tech-stack)
- [🧱 Design Patterns](#-design-patterns)
- [💻 How to Run](#-how-to-run)
- [🧾 License](#-license)
- [📬 Contact](#-contact)

---

## ✨ Core Engineering Contributions

### 1. Entity-Control-Boundary (ECB) Layering

Clear separation of responsibilities across packages:

- **Boundary**: HTTP-like request/response via console I/O
- **Control**: Business logic and role-based data access
- **Entity**: Domain models with state machines

**Constraint**: Domain entities have zero dependencies on Control or Boundary layers.

### 2. Two-Pass In-Memory Relational Graph

CSV files store only IDs; object references constructed in two phases:

**Phase 1 - Entity Hydration**: Load all entities with primitive fields only  
**Phase 2 - Reference Resolution**: Re-parse files to link objects via ID lookups

**Result**: In-memory object graph with bidirectional navigation.

---

## 🏗️ Architecture: Entity-Control-Boundary

```
┌─────────────────────────────────────────────────────────────┐
│ BOUNDARY LAYER                                              │
│ • userinterface/ - Console menu navigation                  │
│ • display/       - Output formatting (no business logic)    │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ CONTROL LAYER                                               │
│ • userctrl/      - Role-specific business logic             │
│ • databasemgr/   - Filtered queries (role-based visibility) │
│ • reader/writer/ - Persistence orchestration                │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ ENTITY LAYER                                                │
│ • user/          - User hierarchy with state machines       │
│ • project/       - Project entities with lifecycle          │
│ • application/   - Application types with FSM               │
│ • enquiry/       - Enquiry domain model                     │
│ • database/      - Generic in-memory repository             │
└─────────────────────────────────────────────────────────────┘
```

<details>
<summary><h3 style='display:inline'>🔍 Layer Enforcement<h3></summary>

**Entity Layer Isolation**:

- No imports from `userctrl`, `databasemgr`, `userinterface`, `display`
- Only dependencies: `misc` (utilities)
- **Verified In**: Import statements in `user/`, `project/`, `application/`, `enquiry/` packages

**Control Layer Responsibilities**:

- `userctrl/`: State transitions, eligibility validation, business rules
- `databasemgr/`: Role-based filtering (`CheckType.isHDBManager()`, `CheckType.isHDBOfficer()`)
- **Example**: `ProjectDatabaseMgr.getData()` returns all projects for managers, only visible non-prohibited for officers

**Boundary Layer Restrictions**:

- No business logic in views
- All calculations delegated to Control layer
- **Example**: `ApplicantInterface.applyForProject()` calls `ApplicantMgr.applyForProject()` for validation

</details>

---

## 💾 Two-Pass Hydration Model

### ⚙️ The Problem

CSV stores object references as string IDs. Naive loading would require:

1. Load all entities
2. For each reference field, perform O(n) lookup
3. Risk of forward references (child loaded before parent)

### 💡 The Solution: Phased Loading

```
┌──────────────────────────────────────────────────────────────┐
│ PHASE 1: ENTITY HYDRATION (Basic Attributes Only)            │
├──────────────────────────────────────────────────────────────┤
│ Users → Projects → Applications → Enquiries                  │
│                                                              │
│ • All entities loaded with primitive fields                  │
│ • Reference fields remain null                               │
│ • O(1) ID → Object map construction                          │
└──────────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────────┐
│ PHASE 2: REFERENCE RESOLUTION (Graph Construction)           │
├──────────────────────────────────────────────────────────────┤
│ Applicant Updates → Officer Updates                          │
│                                                              │
│ • Re-parse files for reference columns only                  │
│ • Resolve IDs via database managers                          │
│ • Build bidirectional navigation                             │
└──────────────────────────────────────────────────────────────┘
```

> **Enforced By**: `BTOManagementSystem.loadData()` execution order

<details>
<summary><h3 style='display:inline-block'>⚙️ Phase 1: Entity Hydration Details</h3></summary>

#### Step 1: Load Users

```
ApplicantReader.read() → List<Applicant>
HDBOfficerReader.read() → List<HDBOfficer>
HDBManagerReader.read() → List<HDBManager>
Add all to userDatabase
```

#### Step 2: Load Projects

```
ProjectReader.read(userDatabase) → List<Project>
```
- Resolves manager/officers via `userMgr.getUser(userDatabase, ID)`
- Skips project if manager null

#### Step 3: Load Applications

```
ApplicationReader.read(userDatabase, projectDatabase) → List<Application>
```
- Resolves user and project by ID
- Skips if either null

#### Step 4: Load Enquiries

```
EnquiryReader.read(userDatabase, projectDatabase) → List<Enquiry>
```

- Resolves filer and project by ID
- Skips if either null

</details>

<details>
<summary ><h3 style='display:inline-block'>🧩 Phase 2: Reference Resolution Details</h3></summary>

#### Step 5: Applicant Reference Updates

```java
ApplicantReader.updateApplicants(userDatabase, applicationDatabase, projectDatabase)
```

- Re-reads `ApplicantFile.txt` line by line
- Resolves:
  - `appliedProject` via `projMgr.getData(projectDatabase, projectID)`
  - `projectApplication` via `appMgr.getData(applicationDatabase, applicationID)`
  - `withdrawalApplication` via `appMgr.getData(applicationDatabase, applicationID)`
- Sets references even if null (prints error to stderr)

#### Step 6: Officer Reference Updates

```java
HDBOfficerReader.updateHDBOfficers(userDatabase, applicationDatabase, projectDatabase)
```

- Re-reads `HDBOfficerFile.txt` line by line
- Resolves semicolon-separated ID lists:
  - `joinedProjects` (List<Project>)
  - `registeredProjects` (List<Project>)
  - `projectRegistrations` (List<Application>)
- Builds `prohibitedProjects` = joinedProjects + registeredProjects

</details>

---

## 🔄 State Machines

### 1. Application Finite State Machine

**Status Enum**: `PENDING`, `SUCCESSFUL`, `UNSUCCESSFUL`, `BOOKED`, `WITHDRAWN`

<details>
<summary><h4 style='display:inline'>Transition Rules by Application Type</h4></summary>

#### 1. BTO Application

| From | To | Guard | Enforced By |
| :- | :- | :- | :- |
| `PENDING` | `SUCCESSFUL` | None | `BTOApplication.updateStatus()` |
| `PENDING` | `UNSUCCESSFUL` | None | `BTOApplication.updateStatus()` |
| `PENDING` | `BOOKED` | None | `BTOApplication.updateStatus()` |
| `PENDING` | `WITHDRAWN` | None | `BTOApplication.updateStatus()` |

<br>

#### 2. Project Registration

| From | To | Guard | Enforced By |
| :- | :- | :- | :- |
| `PENDING` | `SUCCESSFUL` | None | `ProjectRegistration.updateStatus()` |
| `PENDING` | `UNSUCCESSFUL` | None | `ProjectRegistration.updateStatus()` |
| `PENDING` | `WITHDRAWN` | None | `ProjectRegistration.updateStatus()` |
| `*` | `BOOKED` | **Rejected** | Returns false |

<br>

#### 3. Withdrawal Application

| From | To | Guard | Enforced By |
| :- | :- | :- | :- |
| `PENDING` | `SUCCESSFUL` | None | `WithdrawalApplication.updateStatus()` |
| `PENDING` | `UNSUCCESSFUL` | None | `WithdrawalApplication.updateStatus()` |
| `*` | `BOOKED` | **Rejected** | Returns false |
| `*` | `WITHDRAWN` | **Rejected** | Returns false |

</details>
<br>

### 2. Applicant State Machine

**State Representation**:

```java
boolean canApply, isWithdrawing, isReceiptReady;
Project appliedProject;
Application projectApplication, withdrawalApplication;
```

<details>
<summary><h4 style='display:inline'>Transition Rules</h4></summary>
<br>

**Initial State**: `canApply=true`, all others false/null

**Key Transitions:**

| Trigger | State Changes | Enforced By |
| :- | :- | :- |
| Apply for project | `canApply: true→false`<br>`isWithdrawing: *→false` | `Applicant.setAppliedProject()` |
| Submit withdrawal | `isWithdrawing: false→true` | `Applicant.setWithdrawalApplication()` |
| Withdrawal successful | `appliedProject: Project→null`<br>`canApply: false→true`<br>`isWithdrawing: true→false` | `UserMgr.updateStatus()` |
| BTO booked | `canApply: *→false`<br>`isReceiptReady: false→true` | `UserMgr.updateStatus()` |

**Invalid States Prevented**:

- `appliedProject != null` AND `canApply == true`
- `projectApplication != null` AND `canApply == true`
- `withdrawalApplication != null` AND `isWithdrawing == false`

</details>
<br>

### 3. HDB Officer State Machine

**Additional State**: Lists of `joinedProjects`, `registeredProjects`, `prohibitedProjects`, `projectRegistration`

<details>
<summary><h4 style='display:inline'>Transition Rules</h4></summary>
<br>

**Key Transitions**:

| Trigger | State Changes | Enforced By |
| :- | :- | :- |
| Register for project | Add to `registeredProjects`, `prohibitedProjects`, `projectRegistration` | `HDBOfficerMgr.registerForProject()` |
| Registration approved | Move from `registeredProjects` to `joinedProjects`<br>Add to `project.officers` | `UserMgr.updateStatus()` |

**Registration Eligibility**:

- Cannot register if new project's dates overlap with any joined project
- **Check**: `!(registeredProject.applicationStartDate < newProject.applicationStartDate AND registeredProject.applicationEndDate < newProject.applicationEndDate)`
- **Enforced By**: `HDBOfficerMgr.checkJoinEligibility()`

**Invariant**: `prohibitedProjects` = `joinedProjects` ∪ `registeredProjects` (maintained during load only)

</details>
<br>

---

## 👥 Authority Model

### 👤 Role Hierarchy

```
User
├── Applicant
│   └── HDBOfficer (also implements HDBOfficial)
└── HDBManager (also implements HDBOfficial)
```

### 🛡️ Permission Matrix

| Operation | Applicant | Officer | Manager | Enforcement |
| :- | :- | :- | :- | :- |
| View visible projects | ✓ (filtered by age/marital) | ✓ (exclude prohibited) | ✓ (all) | `ProjectDatabaseMgr.getData()` |
| Apply for project | ✓ (eligibility checked) | ✓ (as Applicant) | ✗ | `ApplicantMgr.applyForProject()` |
| Register for project | ✗ | ✓ (date overlap checked) | ✗ | `HDBOfficerMgr.registerForProject()` |
| Book applicant flat | ✗ | ✓ (joined projects only) | ✗ | `HDBOfficerMgr.bookApplicantFlat()` |
| Create/edit/delete project | ✗ | ✗ | ✓ | `HDBManagerMgr` methods |
| Update application status | ✗ | ✗ | ✓ | `HDBManagerMgr.updateStatus()` |
| View all applications | ✗ | ✓ (joined projects) | ✓ (all) | `ApplicationDatabaseMgr.getData()` |
| View enquiries | ✓ (own) | ✓ (assigned projects) | ✓ (all or assigned) | `EnquiryDatabaseMgr.getData()` |
| Reply to enquiries | ✗ | ✓ (assigned projects) | ✓ (assigned projects) | `HDBOfficialMgr.replyTo()` |

<details>
<summary><h3 style='display:inline-block'>🔐 Eligibility Constraints</h3></summary>

**Age and Marital Status for BTO**:

- Age ≥ 35 and single: only 2-Room projects
- Age ≥ 21 and married: all room types
- Other combinations: no projects visible
- **Enforced By**: `ApplicantMgr.getProjects()`, `applyForProject()`

**Officer Project Restrictions**:

- Cannot view/register projects in `prohibitedProjects`
- Cannot register if dates overlap with `joinedProjects`
- **Enforced By**: `ProjectDatabaseMgr.getData()`, `HDBOfficerMgr.checkJoinEligibility()`

**Applicant State Guards**:

- Can apply only if `canApply == true`
- Can submit withdrawal only if `!isWithdrawing`
- Can get receipt only if `isReceiptReady`
- **Enforced By**: `ApplicantInterface` menu logic

</details>

---

## 🛠️ Tech Stack

| Component | Technology | Description |
| :- | :- | :- |
| **Language** | Java SE | Core application |
| **Persistence** | BufferedReader/Writer | CSV-based file I/O |
| **Architecture** | ECB (Entity-Control-Boundary) | Clear layer separation |
| **Date Handling** | `java.time.LocalDate` | DD-MM-YYYY format |
| **ID Generation** | `java.util.UUID` | With type-specific prefixes |

---

## 🧱 Design Patterns

| Pattern | Implementation | Provable In |
| :- | :- | :- |
| **Factory** | `ApplicationMgr.create()` | Switch on `APPLICATION_TYPE` enum |
| **Template Method** | `ItemDisplayer<T>` | `display(List<T>)` calls abstract `display(T)` |
| **Strategy** | `Application.updateStatus()` | Different logic per subclass |
| **Facade** | `BTOManagementSystem` load/save | Coordinates 6 entity types across phases |
| **Manager/Controller** | Per-role managers | `ApplicantMgr`, `HDBOfficerMgr`, `HDBManagerMgr` |
| **Marker Interface** | `HDBOfficial` | Empty interface for type identification |
| **Repository** | `Database<T>` + managers | Role-based filtered access |

---

## 💻 How to Run

### ⚙️ Prerequisites

- Java JDK 8+
- Create `data/` directory in project root

---

### 🚀 Compilation & Execution

```bash
javac -d bin src/**/*.java
java -cp bin main.BTOManagementSystem
```

---

### 🧩 Test Workflows

**Applicant Flow**:

1. Sign up with valid NRIC (e.g., S1234567A, 9 chars, starts S/T, 7 digits + 1 letter)
2. Browse projects (filtered by age/marital status)
3. Apply for eligible project
4. View application status

**Officer Flow**:

1. Sign up as Officer
2. Register for projects (date overlap validation)
3. Book applicant flats for joined projects

**Manager Flow**:

1. Login as Manager
2. Create project (assign manager/officers)
3. Update application statuses
4. Generate reports

---

## 🧾 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

You’re welcome to use, modify, or build upon this project for learning or non-commercial purposes.

---

## 📬 Contact

Developed by **Aaron Davis**

Email: [aaronddavis001@gmail.com]

LinkedIn: [https://linkedin.com/in/aaron-daniel-davis]
