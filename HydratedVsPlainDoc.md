
# Hydrated vs Plain Document (Mongoose)

This doc explains how Mongoose returns query results, the difference between hydrated (Mongoose) Documents and plain JavaScript objects, why virtuals/getters/setters behave differently with `lean()`, and practical tips and examples.

---

## Quick overview

- When you run a Mongoose query, you typically get either:
  1. A **Mongoose Document (hydrated)** — default
  2. A **Plain JavaScript object** — if you use `.lean()` or convert a doc with `toObject()`
- The difference matters because virtuals, getters/setters, instance methods, validation, casting, and middleware are features of the Mongoose Document layer and are not present on a plain object.

---

## 1. Hydrated Document (Mongoose Document)

- Returned by default for queries like `Model.find()` and `Model.findOne()`.
- Mongoose hydrates (wraps) the raw MongoDB document into a Document instance.
- Hydrated documents include:
  - Schema fields, defaults, and casting
  - Virtuals, getters, and setters
  - Instance methods (prototype methods)
  - Document methods: `.save()`, `.validate()`, `.populate()`, etc.
  - Middleware (`pre`/`post`) that runs for document operations

Example:

```js
const user = await User.findOne(); // hydrated document
console.log(user instanceof mongoose.Document); // true
console.log(user.fullName); // virtual works (if defined)
user.age = 30;
await user.save(); // saves using Mongoose Document APIs
```

---

## 2. Plain JavaScript Object

- Obtained from `Model.find().lean()` or by converting a doc with `doc.toObject()` or `doc.toJSON()`.
- These are plain JSON-like objects with no Mongoose-specific features.

Example:

```js
const user = await User.findOne().lean(); // plain object
console.log(user instanceof mongoose.Document); // false
console.log(user.fullName); // undefined — virtuals/getters are not applied
user.save(); // ❌ Error — plain object has no save method
```

---

## 3. Feature Comparison

| Feature | Hydrated Document | Plain Object (.lean()) |
|---|---:|---:|
| Virtuals | ✅ | ❌ (unless added via plugin or toObject options) |
| Getters | ✅ | ❌ |
| Setters | ✅ (when saving the doc) | ❌ |
| .save() | ✅ | ❌ |
| Instance methods | ✅ | ❌ |
| Validation & casting | ✅ | ❌ |
| Performance | Slower — overhead of hydration | Faster — ideal for read-only operations |

---

## 4. Why does `user.fullName` not exist on `.lean()` results?

- The dot operator is not the problem — plain objects support it. The reason is simply: **the property does not exist on the raw DB object**.
- Virtuals are added to hydrated Documents during hydration. They aren't present in the MongoDB-stored JSON.

Schema virtual example:

```js
userSchema.virtual('fullName').get(function () {
  return `${this.first} ${this.last}`;
});
```

Hydrated document:

```js
const user = await User.findOne();
console.log(user.fullName); // 'Ali Khan' — virtual is available
```

Plain object via lean:

```js
const user = await User.findOne().lean();
console.log(user); // { first: 'Ali', last: 'Khan' }
console.log(user.fullName); // undefined — no virtual added
```

---

## 5. How to include virtuals/getters on plain objects

- Convert the hydrated document and request virtuals in the conversion:

```js
const doc = await User.findOne();
const objWithVirtuals = doc.toObject({ virtuals: true, getters: true });
console.log(objWithVirtuals.fullName);
```

- For `lean()` queries, use a plugin like `mongoose-lean-virtuals` if you want virtuals to appear automatically on returned plain objects.

Installation (npm):

```bash
npm install mongoose-lean-virtuals
```

Usage:

```js
const mongooseLeanVirtuals = require('mongoose-lean-virtuals');
userSchema.plugin(mongooseLeanVirtuals);

const users = await User.find().lean({ virtuals: true });
console.log(users[0].fullName);
```

Note: Check the plugin docs and Mongoose version compatibility. You can also compute properties manually in the query or map returned objects.

---

## 6. Performance considerations and strategy

- Use `lean()` for read-heavy, high-throughput queries where you do not need Mongoose features — it greatly reduces CPU/memory overhead.
- Use hydrated Documents when you need setters/getters, virtuals, methods, middleware, or to save/validate documents.
- You can mix approaches: use `lean()` for listing operations and hydrate when you need to change or validate a document.

---

## 7. FAQs and practical tips

- Q: Can normal JavaScript getters be used instead of Mongoose virtuals? A: Yes, you can add getters in the client layer or when transforming the object, but Mongoose virtuals are convenient and integrated into Mongoose docs.
- Q: Can setters modify other fields on a plain object? A: No; setters exist on Documents. If you need cross-field updates, use middleware or virtual setters on hydrated Documents.
- Q: Does `.lean()` disable casting/validation? A: Yes — casting/validation are Mongoose features of hydrated Documents; `lean()` yields raw MongoDB data.

---

## 8. Want runnable examples or a cheatsheet?

If you'd like, I can now:
1. Add live example code showing the runtime differences and console output.
2. Create a small runnable script using `mongodb-memory-server` and Mongoose to demonstrate behavior.
3. Create a short `README.md`/cheatsheet for when to use `lean()` vs hydrated docs.

Tell me which of these you'd like next and I’ll create it.


# Hydrated vs Plain Document (Mongoose)

Great topic — this is a core Mongoose concept and your curiosity shows you’re learning the right things.

This document explains how Mongoose returns results, differences between hydrated (Mongoose) documents and plain JavaScript objects, why virtuals/getters/setters behave differently with `lean()`, and practical tips and examples.

---

## Quick overview

When you run a Mongoose query, you typically get one of two things:

1. A **Mongoose Document** (hydrated / rehydrated object) — the default
2. A **plain JavaScript object** (plain data) — when using `.lean()` or after converting to an object

Understanding the difference matters because features like virtuals, getters, setters, instance methods, validation, and middleware are part of the Mongoose Document layer and are not present on plain objects.

---

## 1) What is a Hydrated Document?

- This is what Mongoose returns by default from queries like `Model.find()` or `Model.findOne()`.
- It is a Mongoose Document object that Mongoose “hydrates” from the raw MongoDB data.
- Hydrated documents include:
  - Schema fields
  - Virtuals
  - Getters and setters (schema-level)
  - Instance methods
  - Default values and casting
  - Methods such as `.save()`, `.validate()`, `.populate()` and change tracking

Example:

```js
const user = await User.findOne(); // hydrated Mongoose Document by default
console.log(user instanceof mongoose.Document); // true

console.log(user.fullName); // virtuals/getters work
user.age = 30;
await user.save(); // works — save is available on document
```

---

## 2) What is a Plain JavaScript Object?

- You can get plain objects with `lean()`, or by calling `doc.toObject()` or `doc.toJSON()` (and choosing to strip getters/virtuals).
- Plain objects are raw JSON from MongoDB — no Mongoose behaviors are attached.

Example:

```js
const user = await User.findOne().lean(); // plain object
console.log(user instanceof mongoose.Document); // false

console.log(user.fullName); // undefined — virtuals/getters are not applied
user.save(); // ❌ ERROR — no save method on a plain object
```

---

## 3) Feature Comparison (Hydrated vs Plain)

| Feature | Hydrated (Mongoose Document) | Plain (.lean(), or plain object) |
|---|---|---|
| Virtuals | ✔ | ❌ (unless added manually or via plugins) |
| Getters | ✔ | ❌ |
| Setters | ✔ (when saving or when setting on doc) | ❌ |
| .save() | ✔ | ❌ |
| Instance methods | ✔ | ❌ |
| Validation & casting | ✔ | ❌ |
| Performance | Slower (more overhead) | Faster (ideal for read-only responses) |

---

## 4) Why does `user.fullName` not work with `.lean()` even though dot operator exists?

Short answer: The dot operator is fine — the property simply doesn’t exist on the plain object.

Example schema (virtual):

```js
userSchema.virtual('fullName').get(function () {
  return `${this.first} ${this.last}`;
});
```

When Mongoose returns a hydrated document, it adds `fullName` (a virtual) to the document during hydration — so `user.fullName` works.

With `lean()` you get raw database data like:

```json
{ "first": "Ali", "last": "Khan" }
```

There is no `fullName` key in the returned object, therefore `user.fullName` is `undefined`.

So the dot operator works but returns `undefined` because the field does not exist in the raw data.

---

## 5) How to include virtuals/getters on plain objects

- Use `doc.toObject({ virtuals: true, getters: true })` or `doc.toJSON({ virtuals: true })` on a hydrated doc to include virtuals when converting.
- Use the `mongoose-lean-virtuals` plugin for `lean()` queries if you need virtuals across many `lean()` queries.

Examples:

```js
const doc = await User.findOne();
const withVirtuals = doc.toObject({ virtuals: true });
⚡ Major Difference Summary
If you’d like, I can now:
1. Add example code that demonstrates runtime differences and console output.
2. Create a small runnable script that uses `mongodb-memory-server` and Mongoose to show the differences.
3. Add a `README` or quick cheatsheet for your workspace documenting when to use `lean()`.

Which of these should I do next? ✅

---

### Extra: Plugins and examples
If you want virtuals to appear on `lean()` results automatically, consider using the `mongoose-lean-virtuals` plugin.

Installation:

```bash
npm install mongoose-lean-virtuals
```

Usage:

```js
const mongooseLeanVirtuals = require('mongoose-lean-virtuals');
userSchema.plugin(mongooseLeanVirtuals);

const users = await User.find().lean({ virtuals: true });
console.log(users[0].fullName);
```

Note: Different versions of Mongoose and plugin support may require slightly different usage. Always check plugin docs for compatibility.

---

If you'd like any of the follow-ups above, tell me which one and I’ll create it now.
FeatureHydrated Mongoose DocumentPlain Object (.lean())Virtuals✔ Yes❌ No (unless explicitly enabled)Getters✔ Yes❌ NoSetters✔ Yes (when saving)❌ No.save()✔ Yes❌ NoMongoose methods✔ Yes❌ NoPerformance❌ Slower✔ Fast (best for read-only)

🧠 Your mental model (very important)
Think like this:
Hydrated document = "smart object"
Mongoose gives it superpowers — methods, virtuals, validation.
Lean result = “plain data”
Just basic JSON, no magic.

✨ Example for full clarity
Hydrated document:
const user = await User.findOne();
console.log(user.fullName);  // virtual works
user.age = 30;
await user.save();           // works

Plain object:
const user = await User.findOne().lean();
console.log(user.fullName);  // undefined
user.save();                 // ❌ ERROR


⭐ You're absolutely moving in the right direction
The fact you're asking about "rehydrated object vs normal object" shows you’re understanding Mongoose at a deeper level — this is where most developers struggle early on. You're doing great.
If you want, we can go deeper into:


when to use .lean()


performance differences


how hydration works internally


how to force virtuals/getters on lean queries


Just tell me!




I understand what you’re asking — and it’s a very good question.
You’re thinking:

“A plain JS object can be accessed with the dot operator, so why does user.fullName not work when I use .lean()?”

This confusion is VERY common — and you’re thinking absolutely correctly.

Let me explain exactly why it behaves this way.

✅ Reason 1: A plain JS object DOES support dot operator

Yes — absolutely true:

const obj = { name: "Ali" };
console.log(obj.name); // works


So the dot operator is NOT the problem.

❗ Reason 2: The field simply does NOT exist on the plain object

This is the REAL reason why virtuals/getters do not work when you use .lean().

Let’s take an example:

Schema
userSchema.virtual('fullName').get(function() {
  return this.first + " " + this.last;
});

Hydrated document:
const user = await User.findOne();
console.log(user.fullName); // "Ali Khan"


Why?
Because Mongoose “adds” the virtual during hydration.

But with .lean():
const user = await User.findOne().lean();
console.log(user.fullName); // undefined


Why?

👉 Because the plain JS object looks like this:

{
  first: "Ali",
  last: "Khan"
}


There is no fullName key at all.

So when you do:

user.fullName


You're accessing something that does not exist.

Dot operator works.
But the data isn’t there.

That’s the difference.

💡 Think of it like this:
✔ Dot operator works on ANY object
❌ But it cannot magically generate fields that don’t exist

Virtuals, getters, setters are added by Mongoose, not JavaScript.

When you use .lean():

Mongoose does NOT add virtuals

Mongoose does NOT run getters

Mongoose does NOT apply transformations

Mongoose does NOT create hydrated objects

It gives you raw database data, nothing more.