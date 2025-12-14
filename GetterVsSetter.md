# Getter vs Setter in Mongoose — Summary & Examples

Short answer: getters and setters in Mongoose are different depending on whether they are defined on a normal schema field or on a virtual. They use similar syntax but have different effects on storage, output, and queries.

---

## Quick summary
- **Normal schema field setters** run before the data is saved, and they modify the value that gets stored in MongoDB.
- **Normal schema field getters** run when a value is read and can transform the value you get back but do not change the stored value.
- **Virtuals** are not stored in MongoDB. Virtual getters compute values on the fly, and virtual setters can update other fields (for example, setting first/last from a fullName). Virtuals do not affect queries directly.
- Query and middleware behavior depends on whether you run a Mongoose query that returns a hydrated Mongoose Document or a plain JS object (e.g., using `lean()`).

---

## 1) Getters/Setters on normal schema fields

These affect the value that is stored or returned when working with Mongoose Documents.

Example:

```js
const userSchema = new mongoose.Schema({
  name: {
    type: String,
    set: v => v.trim(),         // setter: runs before saving
    get: v => v.toUpperCase()   // getter: runs when reading
  }
});
```

Behavior:
- The setter applies when you assign the property (and before saving) and influences what's persisted.
- The getter applies when you read the property (via property access or conversion to object/JSON) and influences how the value is presented.

Example runtime:

```js
const user = new User({ name: '  alice  ' });
await user.save();
// Stored in DB: 'alice' (set trimmed)
console.log(user.name); // 'ALICE' (getter transformed to uppercase)
```

Important notes:
- The schema setter typically returns the transformed value. It is not designed to modify other fields on the document; do that from virtuals or middleware.
- The getter does not change the stored value in MongoDB (unless you persist changes explicitly).

## 2) Virtual getters/setters

Virtuals are computed and do not create a column/field in MongoDB.

Example:

```js
userSchema.virtual('fullName')
  .get(function () {
    return `${this.first} ${this.last}`;
  })
  .set(function (v) {
    const [first, last] = v.split(' ');
    this.first = first;
    this.last = last;
  });
```

Behavior:
- **Getter:** builds the computed property from existing fields at runtime.
- **Setter:** updates other underlying fields (e.g., `first`, `last`).
- Virtuals are not stored in DB and do not affect queries by themselves.

Example runtime:

```js
const u = new User({ first: 'John', last: 'Doe' });
console.log(u.fullName); // 'John Doe'
u.fullName = 'Jane Smith';
// Now u.first = 'Jane', u.last = 'Smith'
```

## Quick comparison table

| Feature | Normal schema field setter/getter | Virtual getter/setter |
|---|---|---|
| Exists in DB | ✔️ stored | ❌ not stored |
| Setter modifies stored value | ✔️ (returns new stored value) | ❌ (modifies other fields) |
| Getter transforms returned value | ✔️ | ✔️ (computed) |
| Affects queries | ✔️ (indexed/used in queries) | ❌ (virtual itself is not part of query) |

---

## 3) What a query returns: hydrated Document vs plain JS object

- **Hydrated Mongoose Document**: This is the default when you do `Model.find()` (no `lean()`). A Mongoose Document has helper methods, getters, setters, and change tracking.
- **Plain JavaScript object**: You can get this using `lean()` (e.g., `Model.find().lean()`), or by using `doc.toObject()` or `doc.toJSON()`. Plain objects are simple POJOs and **do not** apply Mongoose getters/setters or document methods.

Example:

```js
const doc = await User.findOne({ _id: id });
console.log(doc instanceof mongoose.Document); // true

const plain = await User.findOne({ _id: id }).lean();
console.log(plain instanceof mongoose.Document); // false
```

So if you rely on getters/setters on the document, use hydrated documents; `lean()` removes those wrappers for performance.

---

## 4) Middleware (pre and post hooks)

- **pre hooks** run before an operation; they can modify the query or transform data.
- **post hooks** run after an operation; they receive outcome parameters (e.g., saved doc) and can perform side effects.

Example `post('save')`:

```js
userSchema.post('save', function(doc) {
  console.log('POST: User saved:', doc.name);
  // doc is the *hydrated* document that was just saved
});
```

Notes:
- The `doc` here is the saved Mongoose Document (hydrated). You can read fields or run other operations, but changes to `doc` done here won't affect the already-saved document unless you save again.


