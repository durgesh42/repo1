# CODE GENERATION INSTRUCTIONS - COACH DASHBOARD API

## MONGOOSE MODEL PATTERNS (FASTIFY)

### Model Structure Template

```js
// api/fastify-models/ModelName.js
module.exports = {
	attributes: {
		userId: {
			type: mongoose.Schema.Types.ObjectId,
			required: true,
		},
		status: {
			type: String,
			enum: ["active", "inactive"],
			required: true,
		},
		createdAt: {
			type: Date,
			default: Date.now,
		},
	},

	// Static method example for complex queries
	findActiveUsersWithSubscription: (opts, callback) => {
		const { coachId, startDate, endDate } = opts;

		if (_.isEmpty(coachId)) {
			return callback({
				status: 400,
				message: "Missing required parameters",
			});
		}

		UserModel.find({
			coach: coachId,
			status: "active",
			subscriptionEnd: { $gte: startDate, $lte: endDate },
		})
			.populate("subscription")
			.callbackExec(callback);
	},
};
```

### Query Organization Pattern

```js
// ALL database queries should be organized properly:

// GOOD: Simple queries can be in services
StepperNote.findOne({ _id: stepperNoteId }).callbackExec((err, note) => {
	if (err) return callback(err);
	callback(null, note);
});

// GOOD: Complex queries as model static methods
// Inside api/fastify-models/User.js
findActiveUsersWithSubscription: (opts, callback) => {
	const { coachId, startDate, endDate } = opts;

	UserModel.find({
		coach: coachId,
		status: "active",
		subscriptionEnd: { $gte: startDate, $lte: endDate },
	})
		.populate("subscription")
		.callbackExec(callback);
},
	// GOOD: Usage in services
	User.findActiveUsersWithSubscription(
		{ coachId, startDate, endDate },
		(err, users) => {
			if (err) return callback(err);
			callback(null, users);
		}
	);

// BAD: Complex queries directly in controllers
// Controllers should call services, not database directly
```

### Service Method Pattern

```js
// api/fastify-services/ServiceName.js
module.exports = {
	/**
	 * Method description
	 * @param {Object} opts - Options object
	 * @param {string} opts.userId - User identifier
	 * @param {string} opts.coachId - Coach identifier
	 * @param {Function} callback - Callback function (err, result)
	 */
	methodName: (opts, callback) => {
		const { userId, coachId } = opts;

		if (_.isEmpty(userId) || _.isEmpty(coachId)) {
			app.logger.error(
				`Missing required parameters in ServiceName.methodName,`,
				`userId: ${userId},`,
				`coachId: ${coachId}`
			);
			return callback({
				status: 400,
				message: "Missing required parameters",
			});
		}

		// Database operations using callbackExec
		ModelName.findOne({ user: userId }).callbackExec((err, data) => {
			if (err) {
				app.logger.error(
					`Error in ServiceName.methodName: ${JSON.stringify(err)}`
				);
				return callback(err);
			}

			callback(null, data);
		});
	},
};
```

### Advanced Database Patterns

```javascript
// Count documents with callbackExec
StepperNoteDetail.countDocuments(query).callbackExec((err, count) => {
	if (err) return callback(err);
	callback(null, count);
});

// Find with populate and callbackExec
StepperNote.find({ user })
	.populate("appointment")
	.callbackExec((err, data) => {
		if (err) return callback(err);
		callback(null, data);
	});

// Update operations
ModelName.updateOne({ _id: id }, updateData).callbackExec((err, result) => {
	if (err) return callback(err);
	callback(null, result);
});

// Create operations (use callbackCreate when available)
ModelName.callbackCreate(dataToCreate, (err, doc) => {
	if (err) return callback(err);
	callback(null, doc);
});

// Complex queries with multiple conditions
ModelName.find({
	$and: [
		{ status: "active" },
		{ createdAt: { $gte: startDate, $lte: endDate } },
	],
})
	.sort({ createdAt: -1 })
	.limit(10)
	.callbackExec(callback);

// Aggregation pipeline (use callbackExec)
ModelName.aggregate([
	{ $match: { status: "active" } },
	{ $group: { _id: "$coachId", count: { $sum: 1 } } },
	{ $sort: { count: -1 } },
]).callbackExec(callback);
```

## CONTROLLER PATTERNS

### Controller Structure

```js
// api/controllers/ControllerName.js
module.exports = {
	/**
	 * Controller method description
	 * @param {Object} req - Request object
	 * @param {Object} res - Response object
	 */
	methodName: (req, res) => {
		// Validate required fields
		if (!req.body?.userId || !req.body?.requiredField) {
			return res.badRequest("Missing required parameters");
		}

		// Call service method
		ServiceName.methodName(req.body, (err, result) => {
			if (err) {
				app.logger.error(
					`Controller error in ControllerName.methodName: ${JSON.stringify(
						err
					)}`
				);
				return res.negotiate(err);
			}

			res.json(result);
		});
	},
};
```

## ADDITIONAL PROHIBITED PATTERNS

**NEVER USE (in addition to copilot-instructions.md):**

-   `setTimeout` or `setInterval` in production code (use Agenda jobs)
-   Direct database queries in controllers (use services and model static methods)
-   Callback nesting deeper than 3 levels (use async.waterfall)
-   Synchronous file operations (`fs.readFileSync`)
-   Modifying `req` or `res` objects in services

## SPECIALIZED PATTERNS

### JSDoc Documentation

```js
/**
 * Finds active users with valid subscriptions
 * @param {Object} opts - Options object
 * @param {string} opts.coachId - Coach identifier
 * @param {Date} opts.startDate - Start date filter
 * @param {Date} opts.endDate - End date filter
 * @param {Function} callback - Callback function (err, users)
 */
const findActiveUsersWithSubscription = function (opts, callback) {
	// implementation
};
```

### Error Handling Patterns

```js
// Validation error handling
if (_.isEmpty(requiredParam)) {
	app.logger.error(
		`Missing required parameter in ServiceName.methodName`,
		`user: ${JSON.stringify(requiredParam.user)}`
	);
	return callback({ message: "Missing required parameters" });
}
```

### Pagination Patterns

```js
// Implement pagination for large datasets
const limit = Math.min(opts.limit || 20, 100); // Cap at 100
const skip = (opts.page - 1) * limit;

ModelName.find(query)
	.skip(skip)
	.limit(limit)
	.sort({ createdAt: -1 })
	.callbackExec((err, results) => {
		if (err) return callback(err);
		callback(null, results);
	});
```

## QUICK REFERENCE ADDITIONS

**Model Organization**: All complex queries in model files as static methods  
**Service Organization**: Business logic in services, not controllers  
**Error Handling**: Always use structured logging with context  
**Validation**: Use `_.isEmpty()` and `_.isUndefined()` for parameter checks  
**Database**: Always use `.callbackExec()` for Mongoose queries  
**Pagination**: Always limit and validate page sizes  
**Documentation**: JSDoc comments for all methods  
**Async Control**: Use `async.waterfall` for sequential, `async.parallelLimit` for concurrent operations
