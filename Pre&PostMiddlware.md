
# Pre & Post Middleware (Mongoose)

This file explains how Mongoose middleware (pre and post hooks) works, shows examples and expected output, highlights differences between document and query middleware, and provides best practices and tips.

---

## Summary — what middleware is

- Middleware runs before/after certain Mongoose model or document operations, such as `save`, `validate`, `remove` (document middleware) and `find`, `updateOne`, `findOneAndUpdate` (query middleware).
- Use middleware for auditing (timestamps), enforcing business rules, logging, populating derived fields, validation, and automations.

---

## Document middleware (this is a document instance)

- Document middleware runs when you call document methods like `doc.save()` or `doc.remove()`.
- In these hooks, `this` is the document instance.

Example — `pre('save')` and `post('save')`:

```js
const userSchema = new mongoose.Schema({
  name: String,
  createdAt: Date,
  updatedAt: Date,
});

// Pre-save: runs before saving; can modify the document directly
userSchema.pre('save', function (next) {
  console.log('PRE: About to save user');
  if (!this.createdAt) this.createdAt = new Date();
  this.updatedAt = new Date();
  next();
});

// Post-save: receives the saved document
userSchema.post('save', function (doc) {
  console.log('POST: User saved:', doc.name);
});

const User = mongoose.model('User', userSchema);

// Usage
const user = new User({ name: 'Ali' });
await user.save();
```

Expected console output:
```
PRE: About to save user
POST: User saved: Ali
```

Notes:
- `pre('save')` can modify the doc and influence what gets persisted. If you modify fields inside `post('save')`, those changes are not applied to the database unless you call `doc.save()` again.
- Prefer `function (next)` or `async function()` for hooks; don't use arrow functions because `this` must refer to the document.

---

## Query middleware (this is the Query)

- Query middleware runs for model queries such as `Model.find()`, `Model.findOne()`, `Model.updateOne()`, `Model.findOneAndUpdate()`.
- In query middleware, `this` is the query object (not a document). Use Query helper methods to inspect/update:
  - `this.getFilter()` — return the query filter
  - `this.getUpdate()` — return the update payload
  - `this.getOptions()` — return the query options
  - `this.setUpdate(update)` — safely set/replace the update payload
  - `this.model` — the model executing the query

Example — `pre('find')` (add a default filter):

```js
userSchema.pre('find', function (next) {
  console.log('PRE-FIND: Filtering out deleted users');
  this.where({ deleted: { $ne: true } });
  next();
});

userSchema.post('find', function (docs) {
  console.log('POST-FIND: Found', docs.length, 'users');
});

await User.find();
```

Output:
```
PRE-FIND: Filtering out deleted users
POST-FIND: Found 3 users
```

Notes:
- Query middleware runs in the query's lifecycle. `post('find')` receives the result documents after the query executes.
- If you use `lean()` the docs are plain objects (not hydrated), so virtuals, getters, and schema methods are not available.

---

## Modifying updates in query middleware (safe patterns)

Example — automatically add `updatedAt` to updates using `pre('updateOne')`:

```js
userSchema.pre('updateOne', function (next) {
  // inspect current update payload
  const update = this.getUpdate() || {};
  const $set = update.$set ? { ...update.$set } : {};
  $set.updatedAt = new Date();
  // set the update to merge the new set
  this.setUpdate({ ...update, $set });
  next();
});
```

Why this pattern?
- `this.getUpdate()` returns the existing update, which might include operators like `$set`, `$inc`, etc.
- `this.setUpdate()` safely replaces the update payload — it's clearer than calling `this.update()` in middleware, which can be confusing and sometimes replace the filter.

Bad pattern to avoid:

```js
this.update({}, { $set: { updatedAt: new Date() } });
```

This may behave unexpectedly because `this.update` can change both filter and update; prefer `setUpdate` for clarity.

---

## Post hooks for update queries

- `post('findOneAndUpdate', function(result) { ... })` receives the document returned by the operation (if any). Make sure to pass `new: true` or `returnDocument: 'after'` so you receive the updated document rather than the original.

Example:

```js
userSchema.pre('findOneAndUpdate', function (next) {
  console.log('PRE: findOneAndUpdate -', this.getFilter());
  next();
});

userSchema.post('findOneAndUpdate', function (result) {
  console.log('POST: Updated document:', result); // may be null if not found
});

await User.findOneAndUpdate({ name: 'Ali' }, { $set: { name: 'Ali Khan' } }, { new: true });
```

---

## .save() and hydration

- `.save()` always operates on and returns a hydrated Mongoose Document. So `post('save')` receives a hydrated document instance that includes virtuals, getters, and instance methods.

Example:

```js
userSchema.post('save', function (doc) {
  console.log('POST: Is hydrated?', doc instanceof mongoose.Document); // true
});

await new User({ name: 'Ali' }).save();
```

---

## Lifecycle tips & common pitfalls

- `pre('save')` will not run for `Model.updateOne()` or `Model.findOneAndUpdate()`; those are query hooks. If you need to change fields for any update path, add `pre('updateOne')`/`pre('findOneAndUpdate')` middleware.
- `pre('remove')` is a document hook for `doc.remove()`.
- Use `function()` (not arrow functions) to preserve `this` binding to the document or query.
- If you use `lean()`, middleware still runs (e.g., pre/post find), but the returned docs in post hooks are plain objects.
- To abort an operation from `pre` middleware, call `next(err)` or throw an error in an async hook.

---

## Examples: put it together

1) Add `updatedAt` to any updateOne:

```js
userSchema.pre('updateOne', function (next) {
  const update = this.getUpdate() || {};
  const $set = update.$set ? { ...update.$set } : {};
  $set.updatedAt = new Date();
  this.setUpdate({ ...update, $set });
  next();
});
```

2) Soft-delete filter for all finds:

```js
userSchema.pre('find', function (next) {
  this.where({ deleted: { $ne: true } });
  next();
});
```

3) Inspect filter and update in `findOneAndUpdate`:

```js
userSchema.pre('findOneAndUpdate', function (next) {
  console.log('Filter:', this.getFilter());
  console.log('Update:', this.getUpdate());
  next();
});
```

---

## Further steps I can do for you
1. Add a runnable example using `mongodb-memory-server` showing console output.
2. Add unit tests demonstrating middleware flows and outcomes.
3. Create a short cheatsheet or quick reference of hooks (`save`, `validate`, `remove`, `find`, `updateOne`, `findOneAndUpdate`) and what environment they run in (document vs query) and their `this` binding.

Which of these would you like next? ✅

---

## Quick mental model

- Pre hooks = modify/validate/intercept before the operation runs
- Post hooks = observe/react after the operation completes (logging, notifications, side effects)

---

## Example: What gets stored in DB (illustration)

If you use this pre-save hook:

```js
userSchema.pre('save', function (next) {
  this.createdAt = new Date();
  next();
});
```

And then run:

```js
const user = new User({ name: 'Ali' });
await user.save();
```

The object stored in MongoDB will look like:

```json
{
  "name": "Ali",
  "createdAt": "2025-11-28T10:15:30.000Z"
}
```

This demonstrates that **pre** middleware affects the stored document, while **post** middleware runs after and receives the saved result.
Just tell me!