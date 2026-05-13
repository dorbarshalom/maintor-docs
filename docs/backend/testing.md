# Backend Testing Guide

This document outlines the testing strategy, framework, and test cases implemented for the `maintor-api` service.

## Testing Framework

The `maintor-api` uses **Vitest** as its primary testing framework. 
Vitest was chosen over alternatives like Jest because the project uses native ES Modules (`"type": "module"` in `package.json`). Vitest offers out-of-the-box, configuration-free support for ESM, making it faster and significantly easier to set up.

### Running Tests

You can run the test suite via the following npm commands from the `/maintor-api` directory:

- Run all tests once:
  ```bash
  npm test
  ```
- Run tests in watch mode (re-runs automatically on file changes):
  ```bash
  npm run test:watch
  ```

---

## Testing Strategy & Mocking

The backend interacts heavily with external Google Cloud services (like Cloud Tasks) and MongoDB. Instead of performing actual database writes or hitting live external services during tests, we heavily rely on **mocking**.

Vitest's `vi.mock()` utility is used to mock database utility functions and external API calls. This ensures:
1. Tests run extremely quickly.
2. The local development database state is not mutated by running tests.

---

## Planned Maintenance Lifecycle Tests

We have implemented test coverage to ensure the stability of the planned maintenance automation logic. The tests are located in `src/__tests__/`.

### 1. `openPlannedTickets.test.js`
This suite verifies the `openPlannedTickets` handler, which is executed periodically by Cloud Scheduler to activate due tasks.

**Tested Behaviours:**
- **Filtering Logic:** Verifies that the handler strictly queries the database for tickets where `type: 'PLANNED'`, `status: 'PLANNED'`, and `scheduled_date` $\le$ current time.
- **Batch Processing:** Ensures the handler properly queries the DB in batches and transitions all applicable tickets to `OPEN`.
- **Fault Tolerance:** Ensures that if an error occurs while updating a single ticket, the execution doesn't crash and the handler gracefully logs the error while continuing to process the remaining tickets.

### 2. `updateSkippedTickets.test.js`
This suite tests the `updateSkippedTickets` handler, which skips expired tickets to prevent a massive task backlog.

**Tested Behaviours:**
- **In-Progress Checking:** Ensures tickets that have already been started (`timeline.started` exists) are ignored by the script.
- **Window Calculation:** Validates the logic that checks if the current time exceeds the execution window (`executionWindowDays`) given by the associated template.
- **Transitioning:** Verifies that expired, unstarted `OPEN` tasks are successfully transitioned to `SKIPPED`.
- **Legacy Date Parsing:** Asserts that older `YYYY-MM-DD` date formats are safely converted to ISO strings before execution window computations.
