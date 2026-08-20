# Practice AI — API Documentation

> **Base URL**: `http://localhost:4000/api/v1`
> **Swagger UI**: `http://localhost:4000/api/docs/practice-ai`
> **Version**: 1.0.0

---

## Authentication

All learner-facing endpoints require a **JWT Bearer token** in the `Authorization` header.

```
Authorization: Bearer <jwt_token>
```

The internal sync endpoint uses an **API Key** via the `x-api-key` header instead.

---

## Common Error Responses

All error responses follow this shape:

```json
{
  "statusCode": 404,
  "message": "Progress not found",
  "error": "Not Found"
}
```

| Status | Meaning |
|--------|---------|
| `400` | Bad Request — validation failed or missing required fields |
| `401` | Unauthorized — missing or invalid JWT token |
| `404` | Not Found — resource does not exist |
| `409` | Conflict — e.g., section already submitted |

### Validation Error (400)

```json
{
  "code": "validation_failed",
  "message": "Validation failed",
  "details": [
    { "field": "answerText", "message": "answerText must be a string" }
  ]
}
```

---

## Table of Contents

| # | Section | Endpoints |
|---|---------|-----------|
| 1 | [Internal – Sync](#1-internal--sync) | 1 |
| 2 | [Progress & Submit](#2-progress--submit) | 5 |
| 3 | [Quiz](#3-quiz) | 4 |
| 4 | [Technical](#4-technical) | 4 |
| 5 | [Coding](#5-coding) | 4 |
| 6 | [Evaluations](#6-evaluations) | 1 |
| 7 | [Sources](#7-sources) | 1 |
| 8 | [User Practices](#8-user-practices) | 1 |

---

## 1. Internal – Sync

### `POST /internal/practices/sync`

> **Auth**: `x-api-key` header (service-to-service)

Syncs a complete practice session (with quiz, coding, technical questions, and sources) from Content AI into the Practice AI data store. This is an **idempotent upsert** — calling it again with the same `id` updates existing data.

**Request Body**:

```json
{
  "id": "content-ai-notebook-id-123",
  "title": "Java Collections Deep Dive",
  "practiceName": "java-collections-deep-dive",
  "level": "Intermediate",
  "description": "Practice session covering Java Collections Framework",
  "languages": "Java",
  "quizQuestions": [
    {
      "contentAiQuizQuestionId": "quiz-q-001",
      "type": "MCQ",
      "question": "Which interface does ArrayList implement?",
      "options": ["List", "Set", "Map", "Queue"],
      "validOptions": ["List"],
      "explanation": "ArrayList implements the List interface.",
      "difficulty": "basic"
    }
  ],
  "codingQuestions": [
    {
      "contentAiCodingQuestionId": "coding-q-001",
      "difficulty": "intermediate",
      "title": "Reverse a Linked List",
      "codingProblem": "Write a method to reverse a singly linked list.",
      "codingInstructions": "Implement the reverseList method.",
      "starterCode": "public ListNode reverseList(ListNode head) {\n  // your code\n}",
      "topics": ["LinkedList", "Recursion"]
    }
  ],
  "technicalQuestions": [
    {
      "contentAiTechnicalQuestionId": "tech-q-001",
      "difficulty": "basic",
      "conceptQuestion": "Explain the difference between ArrayList and LinkedList."
    }
  ],
  "sources": [
    {
      "contentAiSourceId": "src-001",
      "name": "Java Collections Docs",
      "type": "website",
      "url": "https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html",
      "content": null,
      "summary": "Official Java Collections API documentation."
    }
  ]
}
```

**Success Response** `200 OK`:

```json
{
  "success": true,
  "practiceId": "content-ai-notebook-id-123"
}
```

---

## 2. Progress & Submit

### 2.1 `POST /practices/:practiceId/progress/init`

> **Auth**: JWT Bearer

Initializes a new practice progress for the authenticated learner, or resumes an existing one. Also generates per-learner question and option shuffle for the quiz section (idempotent).

**Path Parameters**:

| Param | Type | Description |
|-------|------|-------------|
| `practiceId` | `string` | The practice session ID |

**Request Body**: None

**Success Response** `200 OK`:

```json
{
  "id": "6654abc123def456ghi789",
  "userId": 42,
  "practiceId": "content-ai-notebook-id-123",
  "status": "IN_PROGRESS",
  "startedAt": "2026-08-19T12:00:00.000Z",
  "submittedAt": null,
  "quizSubmittedAt": null,
  "technicalSubmittedAt": null,
  "codingSubmittedAt": null,
  "timeSpentMs": 0,
  "lastActiveAt": "2026-08-19T12:00:00.000Z",
  "autoSavedAt": null,
  "currentQuizQuestionId": null,
  "currentCodingQuestionId": null,
  "currentTechnicalQuestionId": null,
  "createdAt": "2026-08-19T12:00:00.000Z",
  "updatedAt": "2026-08-19T12:00:00.000Z"
}
```

---

### 2.2 `GET /practices/:practiceId/progress/state`

> **Auth**: JWT Bearer

Returns the authenticated learner's current overall practice progress.

**Path Parameters**:

| Param | Type | Description |
|-------|------|-------------|
| `practiceId` | `string` | The practice session ID |

**Success Response** `200 OK`: Same shape as `POST /progress/init` response above.

**Error** `404`: `"No progress found for this practice"`

---

### 2.3 `PATCH /practices/:practiceId/progress/time`

> **Auth**: JWT Bearer

Updates the total time spent for the practice session (called periodically by the frontend timer).

**Path Parameters**:

| Param | Type | Description |
|-------|------|-------------|
| `practiceId` | `string` | The practice session ID |

**Request Body**:

```json
{
  "timeSpentMs": 125000
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `timeSpentMs` | `number` | Yes | Total time spent in milliseconds |

**Success Response** `200 OK`: Full `PracticeProgress` object with updated `timeSpentMs` and `lastActiveAt`.

---

### 2.4 `POST /practices/:practiceId/submit`

> **Auth**: JWT Bearer

Final submission of the entire practice. Marks the practice as `COMPLETED`. Cannot be called twice.

**Path Parameters**:

| Param | Type | Description |
|-------|------|-------------|
| `practiceId` | `string` | The practice session ID |

**Request Body**: None

**Success Response** `200 OK`:

```json
{
  "success": true,
  "submittedAt": "2026-08-19T13:45:00.000Z"
}
```

**Error** `409`: `"Practice already submitted"`

---

### 2.5 `DELETE /practices/:practiceId/delete-progress`

> **Auth**: JWT Bearer

Clears all progress for the authenticated learner on this practice, allowing them to start fresh.

**Path Parameters**:

| Param | Type | Description |
|-------|------|-------------|
| `practiceId` | `string` | The practice session ID |

**Success Response** `200 OK`:

```json
{
  "success": true,
  "message": "Practice progress cleared successfully"
}
```

**Error** `404`: `"Progress not found"`

---

### 2.6 `GET /practices/:practiceId/submission-status`

> **Auth**: Public (no JWT required)

Returns the submission status for a specific learner. Used by external services to check if a learner has submitted.

**Path Parameters**:

| Param | Type | Description |
|-------|------|-------------|
| `practiceId` | `string` | The practice session ID |

**Query Parameters**:

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `learnerId` | `number` | Yes | The learner's user ID |

**Success Response** `200 OK`:

```json
{
  "alreadySubmitted": false,
  "practiceId": "content-ai-notebook-id-123",
  "submittedAt": null,
  "quizSubmittedAt": null,
  "technicalSubmittedAt": null,
  "codingSubmittedAt": null
}
```

---

## 3. Quiz

### 3.1 `GET /practices/:practiceId/quiz`

> **Auth**: JWT Bearer

Fetches the quiz section with per-learner shuffled questions and options.

**Path Parameters**:

| Param | Type | Description |
|-------|------|-------------|
| `practiceId` | `string` | The practice session ID |

**Success Response** `200 OK`:

```json
{
  "title": "Quiz Section",
  "totalQuestions": 10,
  "timeLimitMs": 600000,
  "questions": [
    {
      "id": "quiz-q-001",
      "position": 0,
      "type": "MCQ",
      "question": "Which interface does ArrayList implement?",
      "options": [
        { "id": "A", "text": "Set" },
        { "id": "B", "text": "List" },
        { "id": "C", "text": "Queue" },
        { "id": "D", "text": "Map" }
      ],
      "difficulty": "basic"
    }
  ]
}
```

> **IMPORTANT**: Options are **shuffled per-learner**. The option IDs (`A`, `B`, `C`, `D`) are based on the shuffled display order — **not** the original data order.

---

### 3.2 `GET /practices/:practiceId/quiz/state`

> **Auth**: JWT Bearer

Returns the learner's current quiz state — saved answers, review flags, and time info.

**Success Response** `200 OK`:

```json
{
  "progressId": "6654abc123def456ghi789",
  "timeSpentMs": 35000,
  "submittedAt": null,
  "autosavedAt": "2026-08-19T12:05:00.000Z",
  "answers": {
    "quiz-q-001": ["B"],
    "quiz-q-003": ["A", "C"]
  },
  "reviewList": ["quiz-q-002", "quiz-q-005"]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `answers` | `Record<questionId, string[] or null>` | Map of question ID to selected option letter(s) |
| `reviewList` | `string[]` | Question IDs flagged for review |

---

### 3.3 `PATCH /practices/:practiceId/quiz/answers`

> **Auth**: JWT Bearer

Autosaves a single quiz answer. Called each time the learner selects/deselects an option.

**Request Body**:

```json
{
  "questionId": "quiz-q-001",
  "answer": ["B"]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `questionId` | `string` | Yes | The quiz question ID |
| `answer` | `string[] or null` | No | Selected option letter(s). `null` to clear. |

**Success Response** `200 OK`:

```json
{
  "questionId": "quiz-q-001",
  "answer": ["B"],
  "autosavedAt": "2026-08-19T12:05:30.000Z"
}
```

**Error** `409`: `"Quiz already submitted"`

---

### 3.4 `PATCH /practices/:practiceId/quiz/review`

> **Auth**: JWT Bearer

Toggle the "review later" flag for a quiz question.

**Request Body**:

```json
{
  "questionId": "quiz-q-002",
  "review": true
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `questionId` | `string` | Yes | The quiz question ID |
| `review` | `boolean` | Yes | `true` to flag, `false` to unflag |

**Success Response** `200 OK`:

```json
{
  "questionId": "quiz-q-002",
  "review": true
}
```

---

### 3.5 `POST /practices/:practiceId/quiz/submit`

> **Auth**: JWT Bearer

Submits the quiz section and triggers local auto-grading. Returns the score immediately.

**Request Body**: None

**Success Response** `200 OK`:

```json
{
  "submittedAt": "2026-08-19T12:10:00.000Z",
  "score": {
    "total": 10,
    "correct": 7,
    "percentage": 70
  }
}
```

**Error** `409`: `"Quiz already submitted"`

---

## 4. Technical

### 4.1 `GET /practices/:practiceId/technical`

> **Auth**: JWT Bearer

Fetches the technical section with its conceptual questions.

**Success Response** `200 OK`:

```json
{
  "title": "Technical Concepts",
  "questions": [
    {
      "id": "tech-q-001",
      "position": 1,
      "difficulty": "basic",
      "conceptQuestion": "Explain the difference between ArrayList and LinkedList."
    },
    {
      "id": "tech-q-002",
      "position": 2,
      "difficulty": "intermediate",
      "conceptQuestion": "What is the time complexity of HashMap.get()?"
    }
  ]
}
```

---

### 4.2 `GET /practices/:practiceId/technical/state`

> **Auth**: JWT Bearer

Returns the learner's saved text answers for the technical section.

**Success Response** `200 OK`:

```json
{
  "progressId": "6654abc123def456ghi789",
  "submittedAt": null,
  "answers": {
    "tech-q-001": {
      "answerText": "ArrayList uses a dynamic array, LinkedList uses a doubly-linked list...",
      "savedAt": "2026-08-19T12:15:00.000Z"
    }
  }
}
```

---

### 4.3 `PUT /practices/:practiceId/technical/questions/:questionId/answer`

> **Auth**: JWT Bearer

Saves or updates a text answer for a specific technical question.

**Path Parameters**:

| Param | Type | Description |
|-------|------|-------------|
| `practiceId` | `string` | The practice session ID |
| `questionId` | `string` | The technical question ID |

**Request Body**:

```json
{
  "answerText": "ArrayList uses a dynamic array internally, which provides O(1) random access..."
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `answerText` | `string` | Yes | The learner's text answer |

**Success Response** `200 OK`:

```json
{
  "questionId": "tech-q-001",
  "answerText": "ArrayList uses a dynamic array internally, which provides O(1) random access...",
  "savedAt": "2026-08-19T12:16:00.000Z"
}
```

**Error** `400`: `"Answer text is required"`
**Error** `409`: `"Technical section already submitted"`

---

### 4.4 `POST /practices/:practiceId/technical/submit`

> **Auth**: JWT Bearer

Submits the technical section.

**Request Body**: None

**Success Response** `200 OK`:

```json
{
  "submittedAt": "2026-08-19T12:20:00.000Z"
}
```

**Error** `409`: `"Technical already submitted"`

---

## 5. Coding

### 5.1 `GET /practices/:practiceId/coding`

> **Auth**: JWT Bearer

Fetches the coding section with all coding problems.

**Success Response** `200 OK`:

```json
{
  "title": "Coding Challenges",
  "questions": [
    {
      "id": "coding-q-001",
      "position": 1,
      "difficulty": "intermediate",
      "title": "Reverse a Linked List",
      "codingProblem": "Write a method to reverse a singly linked list.",
      "codingInstructions": "Implement the reverseList method.",
      "starterCode": "public ListNode reverseList(ListNode head) {\n  // your code\n}",
      "topics": ["LinkedList", "Recursion"]
    }
  ]
}
```

---

### 5.2 `GET /practices/:practiceId/coding/state`

> **Auth**: JWT Bearer

Returns the learner's saved code solutions and the last run result per question.

**Success Response** `200 OK`:

```json
{
  "progressId": "6654abc123def456ghi789",
  "submittedAt": null,
  "answers": {
    "coding-q-001": {
      "code": "public ListNode reverseList(ListNode head) {\n  ListNode prev = null;\n  ...\n}",
      "savedAt": "2026-08-19T12:25:00.000Z",
      "lastRunResult": {
        "stdout": "1 -> 2 -> 3 -> null",
        "stderr": null,
        "compileError": null,
        "exitCode": 0,
        "durationMs": 142,
        "ranAt": "2026-08-19T12:24:30.000Z"
      }
    }
  }
}
```

> **NOTE**: `lastRunResult` is `null` if the learner hasn't run their code yet for that question.

---

### 5.3 `PUT /practices/:practiceId/coding/questions/:questionId/answer`

> **Auth**: JWT Bearer

Saves/updates code for a specific coding question (autosave).

**Path Parameters**:

| Param | Type | Description |
|-------|------|-------------|
| `practiceId` | `string` | The practice session ID |
| `questionId` | `string` | The coding question ID |

**Request Body**:

```json
{
  "code": "public ListNode reverseList(ListNode head) {\n  ListNode prev = null;\n  ListNode current = head;\n  while (current != null) {\n    ListNode next = current.next;\n    current.next = prev;\n    prev = current;\n    current = next;\n  }\n  return prev;\n}"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `code` | `string` | Yes | The code solution |

**Success Response** `200 OK`:

```json
{
  "questionId": "coding-q-001",
  "savedAt": "2026-08-19T12:26:00.000Z"
}
```

**Error** `409`: `"Coding section already submitted"`

---

### 5.4 `POST /practices/:practiceId/coding/questions/:questionId/run`

> **Auth**: JWT Bearer

Compiles and runs the code against Judge0 (code execution engine). Also autosaves the code before running.

**Path Parameters**:

| Param | Type | Description |
|-------|------|-------------|
| `practiceId` | `string` | The practice session ID |
| `questionId` | `string` | The coding question ID |

**Request Body**:

```json
{
  "code": "public class Main {\n  public static void main(String[] args) {\n    System.out.println(\"Hello World\");\n  }\n}",
  "language": "java",
  "stdin": ""
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `code` | `string` | Yes | — | Source code to execute |
| `language` | `string` | No | `"java"` | Language: `python`, `javascript`, `java`, `cpp` |
| `stdin` | `string` | No | `""` | Standard input for the program |

**Success Response** `200 OK`:

```json
{
  "stdout": "Hello World\n",
  "stderr": null,
  "compileError": null,
  "exitCode": 0,
  "durationMs": 156,
  "ranAt": "2026-08-19T12:27:00.000Z"
}
```

**Compilation Error Example**:

```json
{
  "stdout": null,
  "stderr": null,
  "compileError": "Main.java:3: error: ';' expected\n    System.out.println(\"Hello\")\n                               ^\n1 error",
  "exitCode": 1,
  "durationMs": 85,
  "ranAt": "2026-08-19T12:27:00.000Z"
}
```

**Runtime Error Example**:

```json
{
  "stdout": "",
  "stderr": "Exception in thread \"main\" java.lang.NullPointerException\n\tat Main.main(Main.java:5)",
  "compileError": null,
  "exitCode": 1,
  "durationMs": 110,
  "ranAt": "2026-08-19T12:27:00.000Z"
}
```

---

### 5.5 `POST /practices/:practiceId/coding/submit`

> **Auth**: JWT Bearer

Submits the coding section.

**Request Body**: None

**Success Response** `200 OK`:

```json
{
  "submittedAt": "2026-08-19T12:30:00.000Z"
}
```

**Error** `409`: `"Coding already submitted"`

---

## 6. Evaluations

### 6.1 `GET /practices/:practiceId/evaluations/:section`

> **Auth**: JWT Bearer

Returns AI/local evaluation results for a specific section.

**Path Parameters**:

| Param | Type | Description |
|-------|------|-------------|
| `practiceId` | `string` | The practice session ID |
| `section` | `string` | One of: `quiz`, `coding`, `technical` |

**Success Response** `200 OK` (Quiz example):

```json
[
  {
    "id": "eval-001",
    "evaluationId": "eval-001",
    "practiceProgressId": "6654abc123def456ghi789",
    "evaluationType": "quiz",
    "submittedAt": "2026-08-19T12:10:00.000Z",
    "completedAt": "2026-08-19T12:10:00.000Z",
    "quizEvaluation": {
      "totalQuestions": 10,
      "correctAnswers": 7,
      "successPercentage": 70,
      "details": [
        {
          "question_text": "Which interface does ArrayList implement?",
          "quiz_type": "MCQ",
          "possible_answers": ["Set", "List", "Queue", "Map"],
          "selected_answers": ["List"],
          "correct_answers": ["List"],
          "is_correct": true
        }
      ]
    },
    "codeEvaluation": null,
    "textEvaluation": null,
    "createdAt": "2026-08-19T12:10:00.000Z",
    "updatedAt": "2026-08-19T12:10:00.000Z"
  }
]
```

> **NOTE**: Returns an **array** of evaluation records. For quiz, only `quizEvaluation` is populated. For coding, `codeEvaluation` is populated. For technical, `textEvaluation` is populated. The others will be `null`.

**Error** `404`: `"Progress not found"`

---

## 7. Sources

### 7.1 `GET /practices/:practiceId/sources`

> **Auth**: JWT Bearer

Returns all source materials linked to the practice session.

**Success Response** `200 OK`:

```json
[
  {
    "id": "src-001",
    "name": "Java Collections Docs",
    "type": "website",
    "url": "https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html",
    "content": null,
    "summary": "Official Java Collections API documentation.",
    "createdAt": "2026-08-19T10:00:00.000Z"
  },
  {
    "id": "src-002",
    "name": "Data Structures PDF",
    "type": "pdf",
    "url": "https://storage.googleapis.com/..../ds-guide.pdf",
    "content": null,
    "summary": "Comprehensive guide to data structures in Java.",
    "createdAt": "2026-08-19T10:00:00.000Z"
  }
]
```

| Field | Type | Description |
|-------|------|-------------|
| `type` | `string` | One of: `pdf`, `website`, `text`, `video`, `audio` |

---

## 8. User Practices

### 8.1 `GET /user-practices`

> **Auth**: JWT Bearer

Returns all practice progress records for the authenticated user. Useful for a "My Practices" dashboard showing all practices a learner has started or completed.

**Success Response** `200 OK`:

```json
[
  {
    "id": "6654abc123def456ghi789",
    "userId": 42,
    "practiceId": "content-ai-notebook-id-123",
    "status": "IN_PROGRESS",
    "startedAt": "2026-08-19T12:00:00.000Z",
    "submittedAt": null,
    "quizSubmittedAt": "2026-08-19T12:10:00.000Z",
    "technicalSubmittedAt": null,
    "codingSubmittedAt": null,
    "timeSpentMs": 600000,
    "lastActiveAt": "2026-08-19T12:10:00.000Z",
    "createdAt": "2026-08-19T12:00:00.000Z",
    "updatedAt": "2026-08-19T12:10:00.000Z"
  },
  {
    "id": "7765bcd234efg567hij890",
    "userId": 42,
    "practiceId": "content-ai-notebook-id-456",
    "status": "COMPLETED",
    "startedAt": "2026-08-18T09:00:00.000Z",
    "submittedAt": "2026-08-18T10:30:00.000Z",
    "quizSubmittedAt": "2026-08-18T09:30:00.000Z",
    "technicalSubmittedAt": "2026-08-18T10:00:00.000Z",
    "codingSubmittedAt": "2026-08-18T10:25:00.000Z",
    "timeSpentMs": 5400000,
    "lastActiveAt": "2026-08-18T10:30:00.000Z",
    "createdAt": "2026-08-18T09:00:00.000Z",
    "updatedAt": "2026-08-18T10:30:00.000Z"
  }
]
```

---

## Endpoint Summary

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/internal/practices/sync` | API Key | Sync practice from Content AI |
| `POST` | `/practices/:practiceId/progress/init` | JWT | Init or resume progress |
| `GET` | `/practices/:practiceId/progress/state` | JWT | Get progress state |
| `PATCH` | `/practices/:practiceId/progress/time` | JWT | Update time spent |
| `POST` | `/practices/:practiceId/submit` | JWT | Final submit |
| `DELETE` | `/practices/:practiceId/delete-progress` | JWT | Clear progress |
| `GET` | `/practices/:practiceId/submission-status?learnerId=` | Public | Check submission status |
| `GET` | `/practices/:practiceId/quiz` | JWT | Fetch quiz (shuffled) |
| `GET` | `/practices/:practiceId/quiz/state` | JWT | Get quiz answers/flags |
| `PATCH` | `/practices/:practiceId/quiz/answers` | JWT | Autosave quiz answer |
| `PATCH` | `/practices/:practiceId/quiz/review` | JWT | Toggle review flag |
| `POST` | `/practices/:practiceId/quiz/submit` | JWT | Submit quiz + get score |
| `GET` | `/practices/:practiceId/technical` | JWT | Fetch technical questions |
| `GET` | `/practices/:practiceId/technical/state` | JWT | Get saved text answers |
| `PUT` | `/practices/:practiceId/technical/questions/:questionId/answer` | JWT | Save text answer |
| `POST` | `/practices/:practiceId/technical/submit` | JWT | Submit technical |
| `GET` | `/practices/:practiceId/coding` | JWT | Fetch coding problems |
| `GET` | `/practices/:practiceId/coding/state` | JWT | Get saved code + runs |
| `PUT` | `/practices/:practiceId/coding/questions/:questionId/answer` | JWT | Save code |
| `POST` | `/practices/:practiceId/coding/questions/:questionId/run` | JWT | Run code (Judge0) |
| `POST` | `/practices/:practiceId/coding/submit` | JWT | Submit coding |
| `GET` | `/practices/:practiceId/evaluations/:section` | JWT | Get evaluation results |
| `GET` | `/practices/:practiceId/sources` | JWT | Get source materials |
| `GET` | `/user-practices` | JWT | Get all user practices |
