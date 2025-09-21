# CODE REVIEW INSTRUCTIONS

## CORE REVIEW CRITERIA

Review code to ensure compliance with:
- **Code Generation Patterns**: Follow all patterns in code-generation.instructions.md
- **Test Generation Standards**: Follow all patterns in test-generation.instructions.md
- **Main Copilot Instructions**: Follow all patterns in copilot-instructions.md

## MODEL REVIEW CHECKLIST

**✅ Model Structure (Fastify)**
- Uses `module.exports = { attributes: {} }` pattern
- Complex queries implemented as static methods in model files
- Uses `.callbackExec()` for all Mongoose queries
- Uses `callbackCreate()` for creation operations

**❌ Common Model Issues**
- Raw database queries in services/controllers (should be in model static methods)
- Missing `.callbackExec()` on Mongoose queries
- Direct Mongoose schema creation instead of attributes pattern

## SERVICE REVIEW CHECKLIST

**✅ Service Structure**
- Methods accept `(opts, callback)` parameters
- Proper parameter destructuring: `const { userId, coachId } = opts;`
- Validation using `_.isEmpty()` and `_.isUndefined()`
- Structured error logging with context
- All database operations use `.callbackExec()`

**❌ Common Service Issues**
- Missing parameter validation
- Direct database queries instead of using model static methods
- Callback nesting deeper than 3 levels (use `async.waterfall`)
- Missing error context in logs

## CONTROLLER REVIEW CHECKLIST

**✅ Controller Structure**
- Controllers only handle HTTP logic, call services for business logic
- Proper request validation before calling services
- Uses `res.badRequest()`, `res.negotiate()`, `res.json()` responses
- No direct database operations

**❌ Common Controller Issues**
- Business logic in controllers (should be in services)
- Direct database queries (should call services)
- Missing request validation

## ASYNC CONTROL REVIEW CHECKLIST

**✅ Async Patterns**
- Uses `async.waterfall` for sequential dependent operations
- Uses `async.parallelLimit` for concurrent operations with concurrency control
- Proper callback error handling throughout chains

**❌ Common Async Issues**
- Callback nesting deeper than 3 levels
- Missing error handling in callback chains
- Using `async/await` in Sails folders (use callbacks only)

## TEST REVIEW CHECKLIST

**✅ Test Structure**
- Tests use proper `describe`/`it` structure
- Uses `before`/`after` for suite-level setup/cleanup
- Uses `beforeEach`/`afterEach` for test-level setup/cleanup
- Descriptive test names with context
- Tests both success and error paths
- Proper database cleanup

**❌ Common Test Issues**
- Missing error case testing
- Inadequate test cleanup
- Missing test database configuration
- Vague test descriptions

## LOGGING STANDARDS

-   Use `app.logger` (never `sails.log`)
-   Include service name and function name in error messages
-   Include relevant IDs (userId, referralId) for debugging (no PII)
-   Log levels: `error` (breaks functionality), `warn` (concerning), `info` (notable events)

Multi-line format for complex logs:

```js
app.logger.error(
	`Error in PatientService.discharge for:`,
	`userId: ${userId},`,
	`referralId: ${referralId},`,
	`error: ${JSON.stringify(err)}`
);
```

## JSDOC STANDARDS

-   Keep brief - single sentence unless complexity demands more
-   Skip obvious descriptions
-   Be specific about function purpose, not implementation
-   For complex functions, list key steps as bullets
-   No descriptions for obvious fields (userId, coachId)

Example:

```js
/**
 * Handles patient discharge process with multiple validation stages.
 *
 * Process steps:
 * 1. Validates required parameters
 * 2. Expires unused invite codes
 * 3. Marks user inactive in UserCoachMapping
 * 4. Expires current subscription
 *
 * @param {Object} opts
 * @param {string} opts.referralId
 * @param {string} opts.userId
 * @param {boolean} opts.isPostDischargeBenefitApplicable
 * @param {function} callback
 */
```

## PROHIBITED PATTERNS REVIEW

**❌ NEVER ALLOW:**
- `setTimeout` or `setInterval` in production code (use Agenda jobs)
- Direct database queries in controllers (use services and model static methods)
- Callback nesting deeper than 3 levels (use `async.waterfall`)
- Synchronous file operations (`fs.readFileSync`)
- Modifying `req` or `res` objects in services
- `sails.log` (use `app.logger`)
- `console.log` (use `app.logger`)
- `async/await` for non-DB operations in Sails folders
- ES6 `import` syntax (use `require`)
- Missing audit trails for healthcare actions
- Logging sensitive/decrypted data

## DATABASE PATTERN REVIEW

**✅ Required Database Patterns**
- All Mongoose queries end with `.callbackExec(callback)`
- Creation uses `ModelName.callbackCreate(data, callback)`
- Complex queries are static methods in model files
- Pagination always limits and validates page sizes
- Aggregation pipelines use `.callbackExec()`

**❌ Database Anti-patterns**
- Missing `.callbackExec()` on queries
- Raw queries in services instead of model static methods
- Unbounded result sets (missing pagination)
- Direct Mongoose `.exec()` usage

## SECURITY REVIEW CHECKLIST

**✅ Security Requirements**
- All PII encrypted using `UtilService.v2Encrypt()`
- KMS decryption using `EncryptionService.kmsDecryptData()`
- Input sanitization using `_.pick()`
- Audit trails for healthcare actions
- No sensitive data in logs

**❌ Security Issues**
- Hardcoded credentials or secrets
- Unencrypted PII storage
- Missing input validation
- Unsafe regex patterns
- Sensitive data in error logs

## SONAR QUALITY CHECKS

**Avoid:**

-   Code duplication (extract reusable functions)
-   Unused variables or imports
-   Empty catch blocks
-   Magic numbers (use named constants)
-   Variable shadowing
-   Commented-out code blocks

**Use:**

-   `const` whenever possible
-   Single responsibility functions
-   Break complex code into smaller functions
