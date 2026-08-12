# Practice AI — Implementation Plan

Build the Practice AI module inside `EJET-ReviewAI-Wrapper-Services` with completely separate data storage, following the same DDD architecture as Review AI.

---

## System Architecture

```mermaid
graph TB
    subgraph Frontend["Frontend App"]
        LUI["Learner UI"]
        AIChat["AI Assistant Sidebar"]
    end

    subgraph ContentAI["Content AI Service"]
        CANotebooks["Notebooks"]
        CASources["Sources"]
        CAQuiz["Quiz Questions"]
        CACoding["Coding Questions"]
        CATech["Technical Questions"]
    end

    subgraph PracticeAIWrapper["Practice AI Wrapper Service (This Service)"]
        SyncAPI["Sync API\n/internal/practices/sync"]
        ProgressAPI["Progress API\n/practices/:id/progress/*"]
        QuizAPI["Quiz API\n/practices/:id/quiz/*"]
        TechAPI["Technical API\n/practices/:id/technical/*"]
        CodingAPI["Coding API\n/practices/:id/coding/*"]
        EvalAPI["Evaluations API\n/practices/:id/evaluations/*"]
        SourcesAPI["Sources API\n/practices/:id/sources"]
        Judge0["Judge0 Service\n(Code Execution)"]
        MongoDB[("MongoDB\npractice_* collections")]
    end

    subgraph PracticeEngine["Practice AI Engine (External - TBD)"]
        EngineEval["Evaluation APIs"]
        EngineAssist["AI Assistant APIs"]
    end

    subgraph External["External Services"]
        J0["Judge0 CE\n(Code Runner)"]
        AuthSvc["Auth Service\n(JWT Validation)"]
    end

    ContentAI -->|"POST /internal/practices/sync\n(x-api-key)"| SyncAPI
    SyncAPI --> MongoDB

    LUI -->|"JWT Auth"| ProgressAPI
    LUI -->|"JWT Auth"| QuizAPI
    LUI -->|"JWT Auth"| TechAPI
    LUI -->|"JWT Auth"| CodingAPI
    LUI -->|"JWT Auth"| EvalAPI
    LUI -->|"JWT Auth"| SourcesAPI

    AIChat -->|"Direct call\n(No wrapper needed)"| EngineAssist

    ProgressAPI --> MongoDB
    QuizAPI --> MongoDB
    TechAPI --> MongoDB
    CodingAPI --> MongoDB
    CodingAPI --> Judge0
    EvalAPI --> MongoDB
    SourcesAPI --> MongoDB

    Judge0 --> J0
    ProgressAPI -.->|"Future: trigger eval"| EngineEval
    EngineEval -.->|"Future: callback/poll"| EvalAPI

    AuthSvc -.->|"JWT verify"| PracticeAIWrapper

    style PracticeEngine fill:#fff3cd,stroke:#ffc107
    style PracticeAIWrapper fill:#d4edda,stroke:#28a745
    style ContentAI fill:#cce5ff,stroke:#007bff
    style Frontend fill:#f8d7da,stroke:#dc3545
```

---

## Data Flow Diagrams

### Flow 1: Practice Sync (Content AI → Practice AI)

```mermaid
sequenceDiagram
    participant CA as Content AI
    participant API as Sync API
    participant DB as MongoDB

    CA->>API: POST /internal/practices/sync
    Note over CA,API: Headers: x-api-key
    Note over CA,API: Body: { id, title, level,<br/>quizQuestions[], codingQuestions[],<br/>technicalQuestions[], sources[] }

    API->>DB: Upsert practices collection
    API->>DB: Upsert/Replace practice_quiz_sections
    API->>DB: Upsert/Replace practice_quiz_questions
    API->>DB: Upsert/Replace practice_technical_sections
    API->>DB: Upsert/Replace practice_technical_questions
    API->>DB: Upsert/Replace practice_coding_sections
    API->>DB: Upsert/Replace practice_coding_questions
    API->>DB: Upsert/Replace practice_sources

    API-->>CA: 200 { success: true, practiceId }
```

### Flow 2: Learner Practice Attempt

```mermaid
sequenceDiagram
    participant FE as Frontend
    participant Prog as Progress API
    participant Quiz as Quiz API
    participant Tech as Technical API
    participant Code as Coding API
    participant J0 as Judge0
    participant DB as MongoDB

    FE->>Prog: POST /practices/:id/progress/init
    Prog->>DB: Create/Resume practice_progress<br/>(status: IN_PROGRESS, started_at: now)
    Prog-->>FE: { progressId, status, timeSpentMs: 0 }

    Note over FE: ── Quiz Section ──
    FE->>Quiz: GET /practices/:id/quiz
    Quiz->>DB: Fetch practice_quiz_sections + questions
    Quiz-->>FE: { section, questions[] }

    FE->>Quiz: PATCH /practices/:id/quiz/answers
    Quiz->>DB: Upsert practice_quiz_answers
    Quiz-->>FE: { saved: true }

    FE->>Quiz: POST /practices/:id/quiz/submit
    Quiz->>DB: Compute score, set quiz_submitted_at
    Quiz-->>FE: { score, submittedAt }

    Note over FE: ── Technical Section ──
    FE->>Tech: GET /practices/:id/technical
    Tech->>DB: Fetch practice_technical_sections + questions
    Tech-->>FE: { section, questions[] }

    FE->>Tech: PUT /practices/:id/technical/questions/:qId/answer
    Note over FE,Tech: Body: { answerText: "..." }
    Tech->>DB: Upsert practice_technical_answers
    Tech-->>FE: { saved: true }

    FE->>Tech: POST /practices/:id/technical/submit
    Tech->>DB: Set technical_submitted_at
    Tech-->>FE: { submittedAt }

    Note over FE: ── Coding Section ──
    FE->>Code: GET /practices/:id/coding
    Code->>DB: Fetch practice_coding_sections + questions
    Code-->>FE: { section, questions[] }

    FE->>Code: PUT /practices/:id/coding/questions/:qId/answer
    Code->>DB: Upsert practice_coding_solutions
    Code-->>FE: { saved: true }

    FE->>Code: POST /practices/:id/coding/questions/:qId/run
    Code->>J0: Execute code (Judge0 CE)
    J0-->>Code: { stdout, stderr, exitCode }
    Code->>DB: Save to practice_coding_runs
    Code-->>FE: { stdout, stderr, exitCode, durationMs }

    FE->>Code: POST /practices/:id/coding/submit
    Code->>DB: Set coding_submitted_at
    Code-->>FE: { submittedAt }

    Note over FE: ── Final Submit ──
    FE->>Prog: POST /practices/:id/submit
    Prog->>DB: Set status: COMPLETED,<br/>submitted_at: now,<br/>calculate total time_spent_ms
    Prog-->>FE: { status: COMPLETED, timeSpentMs }
```

### Flow 3: Time Tracking Heartbeat

```mermaid
sequenceDiagram
    participant FE as Frontend
    participant API as Progress API
    participant DB as MongoDB

    loop Every 30s while learner is active
        FE->>API: PATCH /practices/:id/progress/time
        Note over FE,API: Body: { timeSpentMs: 125000 }
        API->>DB: Update time_spent_ms,<br/>last_active_at: now
        API-->>FE: { timeSpentMs, lastActiveAt }
    end
```

### Flow 4: Future Evaluation Flow (TBD)

```mermaid
sequenceDiagram
    participant API as Practice AI Wrapper
    participant Engine as Practice AI Engine
    participant DB as MongoDB

    Note over API,Engine: This flow will be implemented<br/>when Practice AI Engine is ready

    API->>Engine: POST /evaluate (quiz/technical/coding)
    Note over API,Engine: Payload: questions + answers

    Engine-->>API: Evaluation results
    API->>DB: Store in practice_evaluations
    
    Note over API: FE fetches results via
    Note over API: GET /practices/:id/evaluations/*
```

---

## ER Diagram

```mermaid
erDiagram
    practices {
        string _id PK "Content AI notebook ID"
        string title
        string practice_name UK "Unique slug"
        string level "Basic | Intermediate | Advanced"
        string description
        string languages
        Date created_at
        Date updated_at
    }

    practice_progress {
        ObjectId _id PK
        number user_id "Learner ID"
        ObjectId practice_id FK
        string status "NOT_STARTED | IN_PROGRESS | COMPLETED"
        Date started_at
        Date submitted_at
        Date quiz_submitted_at
        Date technical_submitted_at
        Date coding_submitted_at
        number time_spent_ms "Count up from 0"
        Date last_active_at
        Date auto_saved_at
        ObjectId current_quiz_question_id
        ObjectId current_coding_question_id
        ObjectId current_technical_question_id
        Date created_at
        Date updated_at
    }

    practice_quiz_sections {
        ObjectId _id PK
        ObjectId practice_id FK
        string title
        number total_questions
        number time_limit_ms
        Date created_at
        Date updated_at
    }

    practice_quiz_questions {
        ObjectId _id PK
        ObjectId quiz_section_id FK
        number position
        string type "MCQ | MSQ"
        string question
        string[] options
        string[] valid_options "Server-only"
        string explanation
        string difficulty "basic | intermediate | advanced"
        Date created_at
        Date updated_at
    }

    practice_quiz_question_orders {
        ObjectId _id PK
        ObjectId practice_progress_id FK
        ObjectId quiz_question_id FK
        number position
        Date created_at
        Date updated_at
    }

    practice_quiz_question_option_orders {
        ObjectId _id PK
        ObjectId practice_progress_id FK
        ObjectId quiz_question_id FK
        number[] option_order "Shuffled indices"
        Date created_at
        Date updated_at
    }

    practice_quiz_answers {
        ObjectId _id PK
        ObjectId quiz_question_id FK
        ObjectId practice_progress_id FK
        string[] answer
        Date autosaved_at
        Date created_at
        Date updated_at
    }

    practice_quiz_review_flags {
        ObjectId _id PK
        ObjectId practice_progress_id FK
        ObjectId quiz_question_id FK
        Date flagged_at
        Date created_at
        Date updated_at
    }

    practice_technical_sections {
        ObjectId _id PK
        ObjectId practice_id FK
        string title
        number total_questions
        Date created_at
        Date updated_at
    }

    practice_technical_questions {
        ObjectId _id PK
        ObjectId technical_section_id FK
        number position
        string difficulty "basic | intermediate | advanced"
        string concept_question
        Date created_at
        Date updated_at
    }

    practice_technical_answers {
        ObjectId _id PK
        ObjectId technical_question_id FK
        ObjectId practice_progress_id FK
        string answer_text "Typed text answer"
        Date saved_at
        Date created_at
        Date updated_at
    }

    practice_coding_sections {
        ObjectId _id PK
        ObjectId practice_id FK
        string title
        number total_questions
        Date created_at
        Date updated_at
    }

    practice_coding_questions {
        ObjectId _id PK
        ObjectId coding_section_id FK
        number position
        string difficulty "basic | intermediate | advanced"
        string title
        string coding_problem
        string coding_instructions
        string starter_code
        string[] topics
        Date created_at
        Date updated_at
    }

    practice_coding_solutions {
        ObjectId _id PK
        ObjectId coding_question_id FK
        ObjectId practice_progress_id FK
        string code
        string language
        Date saved_at
        Date created_at
        Date updated_at
    }

    practice_coding_runs {
        ObjectId _id PK
        ObjectId coding_solution_id FK
        string exit_code
        string stdout
        string stderr
        string compile_error
        number duration_ms
        Date ran_at
        Date created_at
        Date updated_at
    }

    practice_evaluations {
        ObjectId _id PK
        string evaluation_id
        string client_id
        string practice_progress_id
        string evaluation_type "code | concept_text | quiz"
        Date submitted_at
        Date completed_at
        string question_text
        object code_evaluation "Nested evaluation data"
        object text_evaluation "Nested evaluation data"
        object quiz_evaluation "Nested evaluation data"
        Date created_at
        Date updated_at
    }

    practice_sources {
        ObjectId _id PK
        ObjectId practice_id FK
        string content_ai_source_id "Original source ID"
        string name
        string type "pdf | website | text | video | audio"
        string url
        string content
        string summary
        Date created_at
        Date updated_at
    }

    practices ||--o{ practice_progress : "learners attempt"
    practices ||--o| practice_quiz_sections : "has one"
    practices ||--o| practice_technical_sections : "has one"
    practices ||--o| practice_coding_sections : "has one"
    practices ||--o{ practice_sources : "has many"

    practice_quiz_sections ||--o{ practice_quiz_questions : "contains"
    practice_technical_sections ||--o{ practice_technical_questions : "contains"
    practice_coding_sections ||--o{ practice_coding_questions : "contains"

    practice_progress ||--o{ practice_quiz_question_orders : "shuffled for"
    practice_progress ||--o{ practice_quiz_question_option_orders : "shuffled for"
    practice_progress ||--o{ practice_quiz_answers : "answered by"
    practice_progress ||--o{ practice_quiz_review_flags : "flagged by"
    practice_progress ||--o{ practice_technical_answers : "answered by"
    practice_progress ||--o{ practice_coding_solutions : "coded by"
    practice_progress ||--o{ practice_evaluations : "evaluated for"

    practice_quiz_questions ||--o{ practice_quiz_answers : "answer for"
    practice_quiz_questions ||--o{ practice_quiz_question_orders : "ordered for"
    practice_quiz_questions ||--o{ practice_quiz_question_option_orders : "options shuffled"
    practice_quiz_questions ||--o{ practice_quiz_review_flags : "flagged"

    practice_technical_questions ||--o{ practice_technical_answers : "text answer for"
    practice_coding_questions ||--o{ practice_coding_solutions : "solution for"
    practice_coding_solutions ||--o{ practice_coding_runs : "run results"
```

---

## Proposed Changes

### Component 1: Environment & Config

#### [MODIFY] [.env](file:///d:/BL_Project/EJET-ReviewAI-Wrapper-Services/.env)

```env
# Practice AI Engine (TBD - Engine not ready yet)
PRACTICE_AI_API_URL=http://localhost:5000/api/v1
PRACTICE_AI_API_KEY=practice-ai-engine-key
```

#### [NEW] `src/config/practice-ai.config.ts`

Register config namespace `practiceAi` with `apiUrl` and `apiKey`.

---

### Component 2: Folder Structure

```
src/practice-ai/
├── practice-ai.module.ts                  (root module)
│
├── practices/                              (sync + practice snapshot)
│   ├── domain/practice.ts
│   ├── dto/sync-practice.dto.ts
│   ├── infrastructure/persistence/document/
│   │   ├── entities/practice.schema.ts
│   │   ├── mappers/practice.mapper.ts
│   │   ├── repositories/practice-document.repository.ts
│   │   └── document-persistence.module.ts
│   ├── practices.controller.ts
│   ├── practices.service.ts
│   └── practices.module.ts
│
├── practice-progress/                      (progress + time tracking)
│   ├── domain/practice-progress.ts
│   ├── enums/practice-status.enum.ts
│   ├── infrastructure/persistence/document/
│   │   ├── entities/practice-progress.schema.ts
│   │   ├── mappers/practice-progress.mapper.ts
│   │   ├── repositories/practice-progress-document.repository.ts
│   │   └── document-persistence.module.ts
│   ├── practice-progress.controller.ts
│   ├── practice-progress.service.ts
│   └── practice-progress.module.ts
│
├── practice-quiz/                          (quiz section)
│   ├── domain/practice-quiz.ts
│   ├── infrastructure/persistence/document/
│   │   ├── entities/practice-quiz.schema.ts
│   │   ├── mappers/practice-quiz.mapper.ts
│   │   ├── repositories/practice-quiz-document.repository.ts
│   │   └── document-persistence.module.ts
│   ├── practice-quiz.controller.ts
│   ├── practice-quiz.service.ts
│   └── practice-quiz.module.ts
│
├── practice-technical/                     (technical - TEXT answers only)
│   ├── domain/practice-technical.ts
│   ├── infrastructure/persistence/document/
│   │   ├── entities/practice-technical.schema.ts
│   │   ├── mappers/practice-technical.mapper.ts
│   │   ├── repositories/practice-technical-document.repository.ts
│   │   └── document-persistence.module.ts
│   ├── practice-technical.controller.ts
│   ├── practice-technical.service.ts
│   └── practice-technical.module.ts
│
├── practice-coding/                        (coding - with FULL run + console)
│   ├── domain/practice-coding.ts
│   ├── infrastructure/persistence/document/
│   │   ├── entities/practice-coding.schema.ts
│   │   ├── mappers/practice-coding.mapper.ts
│   │   ├── repositories/practice-coding-document.repository.ts
│   │   └── document-persistence.module.ts
│   ├── practice-coding.controller.ts
│   ├── practice-coding.service.ts
│   └── practice-coding.module.ts
│
├── practice-evaluations/                   (read-only for now)
│   ├── domain/practice-evaluations.ts
│   ├── infrastructure/persistence/document/
│   │   ├── entities/practice-evaluation.schema.ts
│   │   ├── mappers/practice-evaluation.mapper.ts
│   │   ├── repositories/practice-evaluation-document.repository.ts
│   │   └── document-persistence.module.ts
│   ├── practice-evaluations.controller.ts
│   ├── practice-evaluations.service.ts
│   └── practice-evaluations.module.ts
│
└── practice-sources/                       (sources - synced from Content AI)
    ├── domain/practice-source.ts
    ├── infrastructure/persistence/document/
    │   ├── entities/practice-source.schema.ts
    │   ├── mappers/practice-source.mapper.ts
    │   ├── repositories/practice-source-document.repository.ts
    │   └── document-persistence.module.ts
    ├── practice-sources.controller.ts
    ├── practice-sources.service.ts
    └── practice-sources.module.ts
```

---

### Component 3: API Specifications

#### 3.1 Sync API (Internal — Content AI → Practice AI)

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| `POST` | `/internal/practices/sync` | ApiKeyGuard | Upserts practice + all questions + sources |

#### 3.2 Practice Progress APIs

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| `POST` | `/practices/:practiceId/progress/init` | JWT | Start or resume practice |
| `GET` | `/practices/:practiceId/progress/state` | JWT | Get progress, time spent, positions |
| `PATCH` | `/practices/:practiceId/progress/time` | JWT | Heartbeat: update `time_spent_ms` |
| `POST` | `/practices/:practiceId/submit` | JWT | Submit full practice (all sections) |
| `DELETE` | `/practices/:practiceId/delete-progress` | JWT | Clear all progress |
| `GET` | `/practices/:practiceId/submission-status` | Public | Check submission status |

#### 3.3 Quiz APIs

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| `GET` | `/practices/:practiceId/quiz` | JWT | Fetch quiz section & questions |
| `GET` | `/practices/:practiceId/quiz/state` | JWT | Fetch saved answers, flags |
| `PATCH` | `/practices/:practiceId/quiz/answers` | JWT | Autosave one answer |
| `PATCH` | `/practices/:practiceId/quiz/review` | JWT | Toggle review flag |
| `POST` | `/practices/:practiceId/quiz/submit` | JWT | Submit & score quiz |

#### 3.4 Technical APIs (Text Answers)

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| `GET` | `/practices/:practiceId/technical` | JWT | Fetch questions |
| `GET` | `/practices/:practiceId/technical/state` | JWT | Fetch saved text answers |
| `PUT` | `/practices/:practiceId/technical/questions/:questionId/answer` | JWT | Save text answer |
| `POST` | `/practices/:practiceId/technical/submit` | JWT | Submit technical section |

#### 3.5 Coding APIs (Full Run + Console)

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| `GET` | `/practices/:practiceId/coding` | JWT | Fetch coding questions |
| `GET` | `/practices/:practiceId/coding/state` | JWT | Fetch saved code & runs |
| `PUT` | `/practices/:practiceId/coding/questions/:questionId/answer` | JWT | Save code |
| `POST` | `/practices/:practiceId/coding/questions/:questionId/run` | JWT | Run code (Judge0) |
| `POST` | `/practices/:practiceId/coding/submit` | JWT | Submit coding section |

#### 3.6 Evaluations APIs (Read-Only)

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| `GET` | `/practices/:practiceId/evaluations` | JWT | All evaluation results |
| `GET` | `/practices/:practiceId/evaluations/quiz` | Public | Quiz evaluation |
| `GET` | `/practices/:practiceId/evaluations/technical` | Public | Technical evaluation |
| `GET` | `/practices/:practiceId/evaluations/coding` | Public | Coding evaluation |

#### 3.7 Sources & User History APIs

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| `GET` | `/practices/:practiceId/sources` | JWT | Get all practice sources |
| `GET` | `/user-practices` | JWT | Get all practices for learner |

---

### Component 4: Shared Infrastructure (Reused, Not Duplicated)

| Component | Location | Used For |
|---|---|---|
| `Judge0Service` | `modules/coding/judge0.service.ts` | Code execution |
| `JwtAuthGuard` | `common/guards/jwt-auth.guard.ts` | JWT auth |
| `ApiKeyGuard` | `common/guards/api-key.guard.ts` | Sync API auth |
| `CurrentUser` | `common/decorators/current-user.decorator.ts` | Extract userId |
| `@Public()` | `common/decorators/public.decorator.ts` | Public endpoints |

---

## Implementation Phases

```mermaid
gantt
    title Practice AI Implementation Roadmap
    dateFormat  YYYY-MM-DD
    axisFormat  %b %d

    section Phase 1 - Foundation
    Config + Env vars           :p1a, 2026-08-12, 1d
    Practice schema + Sync API  :p1b, after p1a, 2d

    section Phase 2 - Progress
    Progress schema + APIs      :p2, after p1b, 2d

    section Phase 3 - Quiz
    Quiz schemas + Full CRUD    :p3, after p2, 2d

    section Phase 4 - Technical
    Technical schemas + Text answers :p4, after p3, 1d

    section Phase 5 - Coding
    Coding schemas + Run + Console :p5, after p4, 2d

    section Phase 6 - Sources
    Sources schema + Read API   :p6, after p5, 1d

    section Phase 7 - Evaluations
    Evaluation schema + GET APIs :p7, after p6, 1d

    section Phase 8 - Integration
    User practices + Testing    :p8, after p7, 1d
```

| Phase | What | Details |
|---|---|---|
| **Phase 1** | Foundation | Config, env vars, `practices` schema, `SyncPracticeDto`, sync controller/service |
| **Phase 2** | Progress | `practice_progress` schema, `PracticeStatus` enum, init/state/time/submit/delete APIs |
| **Phase 3** | Quiz | All 6 quiz collections, shuffle logic, answer CRUD, local scoring on submit |
| **Phase 4** | Technical | 3 technical collections, **text answer** save/read/submit (simplest section) |
| **Phase 5** | Coding | 4 coding collections, save code, **Judge0 run** integration, submit |
| **Phase 6** | Sources | `practice_sources` schema, synced during practice sync, GET read API |
| **Phase 7** | Evaluations | `practice_evaluations` schema, **GET-only** endpoints (no trigger) |
| **Phase 8** | Integration | `user-practices` listing API, end-to-end testing of full flow |

---

## Verification Plan

### Automated Tests
- `npm run build` — no compilation errors after each phase
- `npm run start:dev` — server starts clean

### Manual Verification
- Swagger docs (`/api/docs`) — test each endpoint
- MongoDB Compass — verify `practice_*` collections created separately from `review_*`
- Full flow test: sync → init → quiz answers → technical text → code run → submit
- Verify Judge0 code execution works through practice coding endpoints
