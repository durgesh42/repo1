# GITHUB COPILOT INSTRUCTIONS – COACHDASHBOARD-API
                
## FRAMEWORK OVERVIEW

This is a **Coach dashboard API** with dual architecture:

-   Legacy **Sails v0.12.13** (being phased out)
-   New **Fastify v4.28.1** (target architecture)
-   **Mongoose 8.4.4** as ORM for MongoDB (3 connections: main, mayden, tools)
    -   **Main DB**: Core user data, subscriptions, appointments, coach assignments
    -   **Mayden DB**: Partner integration data (referrals, care records) - separate for compliance/security
    -   **Tools DB**: Agenda Jobs, background processes
-   **NODE.JS 20.19.1** runtime

## DUAL ARCHITECTURE MIGRATION

-   **Active migration** from Sails.js to Fastify.js
-   Both systems run in parallel via `app.js` (Sails) and `fastify-app.js` (Fastify)
-   **New Fastify code**: `api/fastify-models/` and `api/fastify-services/`
-   **Legacy Sails code**: `api/models/` and `api/services/` (still operational)
-   **Shared components**: Controllers in `api/controllers/` serve both systems

## Repository Structure

```
coachdashboard-api/
├── app.js                           # Main Sails application entry point
├── fastify-app.js                   # Fastify application entry point
├── package.json                     # Node.js dependencies and scripts
├── getEnvironmentVariables.js       # Environment configuration helper
├── api/                            # Core API layer
│   ├── controllers/                # Request controllers (shared)
│   │   └── MainController.js       # Main application controller
│   ├── fastify-models/             # NEW: Mongoose-based models
│   ├── fastify-services/           # NEW: Fastify services  
│   ├── models/                     # LEGACY: Sails Waterline models
│   ├── services/                   # LEGACY: Sails services
│   │   └── [Business logic services]
│   ├── policies/                   # Authentication & authorization
│   └── responses/                  # Custom response handlers
├── assets/                         # Static assets
│   ├── images/                     # Image assets
│   ├── styles/                     # CSS stylesheets
│   └── templates/                  # Email/document templates
├── config/                         # Application configuration
│   ├── bootstrap.js                # Application bootstrap
│   ├── connections.js              # Database connections
│   ├── globals.js                  # Global configuration
│   ├── http.js                     # HTTP server configuration
│   ├── policies.js                 # Policy configuration
│   ├── routes.js                   # Route definitions
│   ├── RiskScoreGlobals.js         # Risk scoring configuration
│   ├── WellBeingScoreGlobals.js    # Well-being scoring configuration
│   └── env/                        # Environment-specific configs
├── setup/                          # Fastify initialization modules
│   ├── setupGlobals.js             # Global objects setup
│   ├── setupMongo.js               # MongoDB connections
│   └── setupModels.js              # Model registration
├── test/                           # Mocha test suite
│   ├── bootstrap.test.js           # Test bootstrap
│   ├── integration/                # Integration tests
│   ├── unit/                       # Unit tests
│   ├── generators/                 # Test data generators
│   └── generated_datasets/         # Test datasets
├── monstache/                      # MongoDB-Elasticsearch sync
│   ├── config.toml                 # Monstache configuration
│   └── monstache.conf              # Additional settings
├── coverage/                       # Test coverage reports
└── views/                          # EJS view templates
    ├── homepage.ejs                # Application views
    └── [error pages]               # 403, 404, 500 templates
```

## CRITICAL PATTERNS

### GLOBAL SETUP

-   **Fastify bootstrap**: `setup/setupGlobals.js` creates global objects (`mongoose`, `app`, `_`, `async`)
-   **MongoDB connections**: `setup/setupMongo.js` creates 3 connections (main, mayden, tools)
    -   **Main**: Core healthcare data (`mongoConnection`) - default for most models
    -   **Mayden**: Partner integration (`mongoMaydenConnection`) - for `Referral`, `CareRecord` models
    -   **Tools**: Background processes (`mongoToolsConnection`) - for `AgendaJobs` and scheduling
-   **Model registration**: `setup/setupModels.js` auto-registers all fastify-models with callback mixins
-   **Global access**: Services/models available globally (e.g., `User.find()`, `UtilService.encrypt()`)

### MONGODB CALLBACK MIGRATION

-   **Critical mixin**: All Mongoose queries get `.callbackExec()` method for Sails compatibility
-   **Usage**: `User.find({}).callbackExec((err, users) => {})`
-   **Create pattern**: `User.callbackCreate(data, (err, doc) => {})`
-   **Performance**: Use `.lean()` for queries that don't need Mongoose document methods

## ASYNC PATTERNS BY FOLDER

### FOR FASTIFY FOLDERS (api/fastify-models, api/fastify-services)

-   Use async/await for DB calls
-   Use callbacks for other async calls

### FOR SAILS FOLDERS (api/models, api/services)

-   Use callbacks for all async operations

## CORE PATTERNS

### Async Control Patterns
```javascript
// Sequential operations - use async.waterfall
async.waterfall([
    function(callback) {
        User.findOne({ id: userId }).exec(callback);
    },
    function(user, callback) {
        Subscription.findOne({ userId: user.id }).exec(callback);
    }
], function(err, subscription) {
    // Final result
});

// Concurrent operations with concurrency control
async.parallelLimit({
    users: function(cb) { User.find({}).exec(cb); },
    sessions: function(cb) { Session.find({}).exec(cb); },
    messages: function(cb) { Message.find({}).exec(cb); }
}, 5, function(err, results) {
    // Process combined results
});

// Batch processing with concurrency control
async.eachLimit(userBatch, 10, function(user, callback) {
    processUser(user, callback);
}, function(err) {
    // All users processed
});

// Batch processing loops
async.until(
    function() { return reachedEnd; },
    function(callback) { processBatch(callback); },
    function(err) { /* All batches processed */ }
);
```

### Data Manipulation Patterns
```javascript
// Array operations
const userIds = _.map(users, function(user) { return user.id; });
const activeEvents = _.filter(events, function(event) { return event.active; });
const sessionsByUser = _.groupBy(sessions, 'userId');
const sortedAppointments = _.sortBy(appointments, 'scheduledAt');

// Object operations
const publicUser = _.pick(userObj, ['id', 'name', 'email']);
const safeData = _.omit(sensitiveData, ['password', 'ssn']);
const mergedConfig = _.extend(baseConfig, partnerConfig);

// Utility checks
if (_.isEmpty(requiredParam)) return callback({ status: 400, message: "Missing parameters" });
if (_.isUndefined(value)) return callback({ status: 400, message: "Value required" });
```

### Healthcare Security Patterns
```javascript
// KMS encryption/decryption
EncryptionService.kmsDecryptData(encryptedValue, function(err, decryptedValue) {
    if (err) return callback(err);
    // Use decryptedValue safely - never log sensitive data
    callback(null, decryptedValue);
});

// PII encryption before storage
const encrypted = UtilService.v2Encrypt(sensitiveData);

// Audit trail for healthcare actions  
AuditTrailService.logAuditEvent({
    userId, coachId, type: AUDIT_TYPES.ACTION, 
    action: AUDIT_ACTIONS.CREATE, timestamp: new Date()
}, () => {});

// Validate coach permissions
CoachGroupConfigs.hasAccess({ group, role, feature, action }, (err, hasAccess) => {
    if (!hasAccess) return res.forbidden();
});
```

### Service Structure
```javascript
// api/fastify-services/ServiceName.js
const ServiceName = {
    methodName: async (opts, callback) => {
        const { param1, param2 } = opts;
        if (_.isEmpty(param1)) return callback({ status: 400, message: "Missing parameters" });
        
        try {
            const result = await Model.findOne({ field: param1 }).lean().exec();
            callback(null, result);
        } catch (err) {
            app.logger.error(`Error in ServiceName.methodName: ${JSON.stringify(err)}`);
            callback(err);
        }
    }
};
module.exports = ServiceName;
```

### Controller Structure
```javascript
// api/controllers/ControllerName.js  
module.exports = {
    methodName: (req, res) => {
        if (!req.body?.requiredField) return res.badRequest('Missing required field');
        
        ServiceName.methodName(req.body, (err, result) => {
            if (err) return res.negotiate(err);
            res.json(result);
        });
    }
};
```

## HEALTHCARE DOMAIN CONTEXT

### SUBSCRIPTION & BILLING SYSTEM
- **Credit-based subscriptions**: Users consume session credits with expiry tracking
- **Multiple subscription models**: `credits`, `sessions`, `journalling` with priority logic
- **Coach assignments**: `UserCoachMapping` tracks active coach relationships

### DATA COMPLIANCE
- **Emergency protocols**: `EmergencyDetailsService` for storing emergency contacts for AV patients
- **Multi-tenant architecture**: Coach groups with role-based permissions

### BUSINESS WORKFLOWS
- Patient discharge processes with multiple validation stages
- Credit expiry and renewal cycles
- Coach-patient relationship management
- Emergency escalation protocols
- Billing and insurance claim processing

### KEY SERVICES
- `PatientDischargeService`: Handles patient discharge workflows
- `SubscriptionService`: Manages credit-based subscriptions
- `EmergencyDetailsService`: Crisis intervention protocols
- `AuditTrailService`: Compliance and audit logging
- `USReService`: Healthcare billing and CPT codes

## PROHIBITED PATTERNS

**NEVER USE:**
- `sails.log` (use `app.logger`)
- `console.log` (use `app.logger`)
- `async/await` for non-DB operations in Sails folders
- ES6 `import` syntax (use `require`)
- Missing audit trails for healthcare actions
- Modifying original objects in async loops (causes race conditions)
- Hardcoded partner-specific logic (use configuration maps)
- Logging sensitive/decrypted data

## QUICK REFERENCE

**Function Parameters**: Use `opts` object with destructuring
**Validation**: `_.isEmpty()`, `_.isUndefined()` for checks
**Code Reuse**: Check `UtilService.js` and existing services first
**MongoDB**: Use `.lean()` for queries that don't need Mongoose document methods
**Async Control**: `async.waterfall` (sequential), `async.parallelLimit` (concurrent), `async.eachLimit` (batch processing)
**Data Manipulation**: Use underscore.js utilities (`_.map`, `_.filter`, `_.pick`, `_.omit`)
**Security**: Always encrypt PII, use KMS decryption, include audit trails