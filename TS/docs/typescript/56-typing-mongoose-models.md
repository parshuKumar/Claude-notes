# 56 — Typing database models with Mongoose

## What is this?

**Mongoose** is the ODM (Object Document Mapper) that most Node backends use to talk to MongoDB. In plain JavaScript, a Mongoose model is a bag of untyped documents — you define a `Schema`, call `mongoose.model("User", schema)`, and from that point on every document you touch is `any`.

Typing Mongoose means giving the compiler three separate pieces of information that Mongoose keeps in three separate places:

```ts
import { Schema, model, Model, HydratedDocument, Types } from "mongoose";

// 1. The shape of the raw document as it lives in MongoDB:
interface IUser {
  _id:       Types.ObjectId;
  email:     string;
  passwordHash: string;
  createdAt: Date;
}

// 2. The schema — validated against IUser at compile time:
const userSchema = new Schema<IUser>({
  email:        { type: String, required: true, unique: true },
  passwordHash: { type: String, required: true },
  createdAt:    { type: Date,   default: Date.now },
});

// 3. The model — the factory that produces typed documents:
const UserModel = model<IUser>("User", userSchema);
```

From then on, `await UserModel.findById(userId)` returns `HydratedDocument<IUser> | null` — a real typed object with `email`, `passwordHash`, `save()`, `toObject()`, and `_id: Types.ObjectId`, all known to the compiler.

Mongoose 6 introduced first-class generics; Mongoose 7 and 8 refined them. Everything in this doc targets **Mongoose 7/8 style** — the interface-first approach, where *you* write the TypeScript interface and the schema is checked against it.

## Why does it matter?

A Mongoose model sits at the boundary between your typed application code and an untyped, schemaless database. That boundary is where most production bugs live:

- You rename `passwordHash` to `password_hash` in the schema but 14 call sites still read `user.passwordHash` — in JS, they silently read `undefined` and your auth check compares `undefined === undefined`.
- You add `required: true` to a field but your `create()` calls elsewhere don't pass it — you find out in production.
- You call `.lean()` for performance and then call `user.save()` on the result — `save is not a function`, at 3am.
- You compare `user._id === req.params.userId` — an `ObjectId` is never `===` a `string`, so the check is always false and your authorization silently fails open.
- You `populate("organizationId")` and then read `user.organizationId.name` — but only on some code paths is it populated, so half the time it's an `ObjectId` and `.name` is `undefined`.

Every one of those is a compile error once the model is properly typed. Mongoose typing is not decoration — it's the only thing standing between your app and a schemaless database.

## The JavaScript way vs the TypeScript way

```js
// ── JavaScript — everything past the schema is a black box ──────────────────
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema({
  email:        { type: String, required: true },
  passwordHash: { type: String, required: true },
  role:         { type: String, enum: ["admin", "member"], default: "member" },
  createdAt:    { type: Date, default: Date.now },
});

const User = mongoose.model("User", userSchema);

async function authenticate(email, password) {
  const user = await User.findOne({ emial: email });   // typo — silently matches nothing
  if (!user) return null;

  // Is passwordHash the field name? passwordHash? password_hash? hashedPassword?
  // Nothing tells you. Nothing checks. undefined === undefined is true.
  const valid = await bcrypt.compare(password, user.passwordHash);
  if (!valid) return null;

  // Is _id a string? An ObjectId? Depends. You will find out at runtime.
  return { userId: user._id, role: user.roll };        // typo again — undefined role
}

async function listAdmins() {
  const users = await User.find({ role: "admin" }).lean();
  // .lean() returns POJOs — no .save(), no virtuals, no methods.
  // Nothing in JS reminds you of that:
  users[0].save();                                     // TypeError at runtime
}
```

```ts
// ── TypeScript — the schema, the documents, and every query are checked ─────
import { Schema, model, Types, HydratedDocument } from "mongoose";
import bcrypt from "bcryptjs";

interface IUser {
  _id:          Types.ObjectId;
  email:        string;
  passwordHash: string;
  role:         "admin" | "member";
  createdAt:    Date;
}

const userSchema = new Schema<IUser>({
  email:        { type: String, required: true, unique: true },
  passwordHash: { type: String, required: true },
  role:         { type: String, enum: ["admin", "member"], default: "member" },
  createdAt:    { type: Date,   default: Date.now },
});

const UserModel = model<IUser>("User", userSchema);

interface AuthResult { userId: string; role: "admin" | "member" }

async function authenticate(email: string, password: string): Promise<AuthResult | null> {
  const user = await UserModel.findOne({ emial: email });
  //                                     ~~~~~ ❌ Object literal may only specify known
  //                                           properties; 'emial' does not exist in
  //                                           FilterQuery<IUser>. Did you mean 'email'?

  if (!user) return null;

  const valid = await bcrypt.compare(password, user.passwordHash); // ✅ known string field
  if (!valid) return null;

  return {
    userId: user._id.toHexString(),  // ✅ compiler knows _id is Types.ObjectId
    role:   user.role,               // ✅ narrowed to "admin" | "member"
    // role: user.roll               // ❌ Property 'roll' does not exist on IUser
  };
}

async function listAdmins(): Promise<void> {
  const users = await UserModel.find({ role: "admin" }).lean();
  users[0].save();
  //       ~~~~ ❌ Property 'save' does not exist on type
  //            'FlattenMaps<IUser> & { _id: Types.ObjectId }'
}
```

The revelation: once the interface exists, **the schema, the filters, the projections, the update objects, and the returned documents are all checked against the same single source of truth.** A field rename becomes a wave of compile errors instead of a wave of `undefined`s.

---

## Syntax

```ts
import {
  Schema, model, Model, Types,
  HydratedDocument, FilterQuery, UpdateQuery,
} from "mongoose";

// ── 1. Raw document interface — what lives in MongoDB ──────────────────────
interface IUser {
  _id:       Types.ObjectId;          // Mongo's primary key type — NOT a string
  email:     string;                  // required field → non-optional
  nickname?: string;                  // optional field → optional property
  orgId:     Types.ObjectId;          // a reference (ref) is stored as an ObjectId
  createdAt: Date;
}

// ── 2. Instance methods interface — methods available on each document ─────
interface IUserMethods {
  isAdmin(): boolean;                 // `this` will be typed as the hydrated doc
}

// ── 3. Model interface — statics + the doc/methods generics ────────────────
//    Model<TRawDocType, TQueryHelpers, TInstanceMethods, TVirtuals>
interface UserModelType extends Model<IUser, {}, IUserMethods> {
  findByEmail(email: string): Promise<HydratedDocument<IUser, IUserMethods> | null>;
}

// ── 4. Schema — generics mirror the model generics ────────────────────────
//    Schema<TRawDocType, TModelType, TInstanceMethods>
const userSchema = new Schema<IUser, UserModelType, IUserMethods>({
  email:     { type: String, required: true, unique: true },
  nickname:  { type: String },                    // no `required` → optional in IUser
  orgId:     { type: Schema.Types.ObjectId, ref: "Organization", required: true },
  createdAt: { type: Date, default: Date.now },
});

// ── 5. Instance method — `this` is inferred as the hydrated document ───────
userSchema.method("isAdmin", function isAdmin(): boolean {
  return this.email.endsWith("@admin.internal");   // ✅ this.email: string
});

// ── 6. Static method — `this` is the model ─────────────────────────────────
userSchema.static("findByEmail", function findByEmail(email: string) {
  return this.findOne({ email });                  // ✅ this: UserModelType
});

// ── 7. Build the model with the model-type generic ─────────────────────────
const UserModel = model<IUser, UserModelType>("User", userSchema);

// ── 8. Usage — everything typed ────────────────────────────────────────────
const user: HydratedDocument<IUser, IUserMethods> | null =
  await UserModel.findByEmail("ada@example.com");  // ✅ static is on the model type

if (user) {
  user.isAdmin();                                  // ✅ instance method
  user.email = "new@example.com";                  // ✅ typed assignment
  await user.save();                               // ✅ documents have save()
}

// ── 9. Typed filters and updates as standalone values ─────────────────────
const activeFilter: FilterQuery<IUser> = { email: /@example\.com$/ };
const patch:        UpdateQuery<IUser> = { $set: { nickname: "ada" } };
await UserModel.updateMany(activeFilter, patch);
```

---

## How it works — concept by concept

### Concept 1 — The interface-first approach, and why the schema is *checked*, not *inferred*

There are two directions you can go with Mongoose + TypeScript:

1. **Infer the type from the schema** (`InferSchemaType<typeof userSchema>`) — less typing, but the inferred type is often subtly wrong (optionality, defaults, `Date` vs `NativeDate`) and impossible to read in error messages.
2. **Write the interface first**, then pass it as the schema generic — the schema is validated against it.

Direction 2 is the recommended, mainstream approach and the one this doc uses. The key insight is what the generic actually does:

```ts
interface IUser {
  email:     string;
  loginCount: number;
  createdAt: Date;
}

// The generic makes the schema definition object CHECKED against IUser:
const userSchema = new Schema<IUser>({
  email:      { type: String, required: true },
  loginCount: { type: String, default: 0 },
  //            ~~~~~~~~~~~~ ❌ Type 'StringConstructor' is not assignable —
  //                            IUser.loginCount is number, not string
  createdAt:  { type: Date, default: Date.now },
});

// A field that doesn't exist on IUser is also an error:
const badSchema = new Schema<IUser>({
  email:     { type: String, required: true },
  loginCont: { type: Number },   // ❌ 'loginCont' does not exist in type ...
  createdAt: { type: Date },
});
```

`InferSchemaType` still has a place — as a *sanity check* that your hand-written interface and your schema agree:

```ts
import { InferSchemaType } from "mongoose";

type InferredUser = InferSchemaType<typeof userSchema>;
// Useful in a test file: assert the inferred shape is assignable to IUser.
// It is NOT a good primary source of truth for application code.
```

### Concept 2 — `HydratedDocument<T>` — the difference between a document and its data

`IUser` describes the *data*. A Mongoose document is that data **plus** the document machinery: `save()`, `toObject()`, `isModified()`, `populate()`, `$isDeleted`, virtuals, and instance methods.

`HydratedDocument<TRaw, TMethods, TVirtuals>` is the type that combines them:

```ts
import { HydratedDocument } from "mongoose";

// The type of what queries return:
type UserDoc = HydratedDocument<IUser, IUserMethods>;
// ≈ IUser & Document<Types.ObjectId, {}, IUser> & IUserMethods

async function deactivate(userId: string): Promise<void> {
  const user = await UserModel.findById(userId);   // UserDoc | null
  if (!user) throw new Error("User not found");

  user.nickname = undefined;    // ✅ data field
  user.markModified("nickname");// ✅ document method
  await user.save();            // ✅ document method
}

// Always type your helper signatures with HydratedDocument, never with IUser,
// when the function needs document behaviour:
function toPublicJson(user: HydratedDocument<IUser>): { id: string; email: string } {
  return { id: user._id.toHexString(), email: user.email };
}

// ❌ This signature is a lie — you cannot call .save() on it:
function wrong(user: IUser) {
  // user.save();  ❌ Property 'save' does not exist on type 'IUser'
}
```

Rule of thumb:

| You have | Use this type |
|---|---|
| A document from a query (no `.lean()`) | `HydratedDocument<IUser, IUserMethods>` |
| Plain data going *into* `create()` | `Omit<IUser, "_id" \| "createdAt">` or a DTO interface |
| A `.lean()` result | `FlattenMaps<IUser> & { _id: Types.ObjectId }` |
| An API response body | A hand-written `UserResponse` DTO — never `IUser` |

### Concept 3 — `Model<T>` with statics and instance methods

Mongoose's `Model` generic has four slots:

```ts
Model<TRawDocType, TQueryHelpers, TInstanceMethods, TVirtuals>
```

To attach **statics** (methods on the model itself, like `UserModel.findByEmail`), you extend `Model`:

```ts
import { Schema, model, Model, HydratedDocument, Types } from "mongoose";

interface IUser {
  _id:          Types.ObjectId;
  email:        string;
  passwordHash: string;
  role:         "admin" | "member";
  lastLoginAt:  Date | null;
}

// Instance methods — operate on ONE document, `this` is the document:
interface IUserMethods {
  comparePassword(candidate: string): Promise<boolean>;
  touchLogin(): Promise<void>;
}

// Statics — operate on the COLLECTION, `this` is the model:
interface UserModelType extends Model<IUser, {}, IUserMethods> {
  findByEmail(email: string): Promise<HydratedDocument<IUser, IUserMethods> | null>;
  countAdmins(): Promise<number>;
}

const userSchema = new Schema<IUser, UserModelType, IUserMethods>({
  email:        { type: String, required: true, unique: true, lowercase: true },
  passwordHash: { type: String, required: true },
  role:         { type: String, enum: ["admin", "member"], default: "member" },
  lastLoginAt:  { type: Date, default: null },
});

// Instance methods — use `function`, NEVER an arrow (arrows have no `this`):
userSchema.method("comparePassword", async function (candidate: string): Promise<boolean> {
  return bcrypt.compare(candidate, this.passwordHash);   // ✅ this: HydratedDocument<IUser, IUserMethods>
});

userSchema.method("touchLogin", async function (): Promise<void> {
  this.lastLoginAt = new Date();
  await this.save();                                     // ✅ document methods available
});

// Statics:
userSchema.static("findByEmail", function (email: string) {
  return this.findOne({ email: email.toLowerCase() });   // ✅ this: UserModelType
});

userSchema.static("countAdmins", function () {
  return this.countDocuments({ role: "admin" });
});

export const UserModel = model<IUser, UserModelType>("User", userSchema);

// Both are now fully typed at call sites:
const admin = await UserModel.findByEmail("ada@example.com");
if (admin && (await admin.comparePassword("s3cret"))) {
  await admin.touchLogin();
}
const adminCount: number = await UserModel.countAdmins();
```

The alternative `schema.methods.x = ...` / `schema.statics.x = ...` object syntax also works, but the `.method(...)` / `.static(...)` *function* form gives you `this` inference. Prefer it.

### Concept 4 — Typing virtuals

A **virtual** is a computed property that is not stored in MongoDB. It needs its own generic slot because it exists on the document but *not* on `IUser`:

```ts
interface IUser {
  _id:       Types.ObjectId;
  firstName: string;
  lastName:  string;
  orgId:     Types.ObjectId;
}

// Virtuals live in their own interface — they are NOT part of the raw doc:
interface IUserVirtuals {
  fullName: string;
  initials: string;
}

interface UserModelType extends Model<IUser, {}, {}, IUserVirtuals> {}

// Schema generic slots: <TRaw, TModel, TInstanceMethods, TQueryHelpers, TVirtuals>
const userSchema = new Schema<IUser, UserModelType, {}, {}, IUserVirtuals>({
  firstName: { type: String, required: true },
  lastName:  { type: String, required: true },
  orgId:     { type: Schema.Types.ObjectId, ref: "Organization", required: true },
});

// Getter-only virtual:
userSchema.virtual("fullName").get(function (this: IUser): string {
  return `${this.firstName} ${this.lastName}`;
});

userSchema.virtual("initials").get(function (this: IUser): string {
  return `${this.firstName[0]}${this.lastName[0]}`.toUpperCase();
});

const UserModel = model<IUser, UserModelType>("User", userSchema);

const user = await UserModel.findById(userId);
if (user) {
  user.fullName;   // ✅ string — the virtual is on the hydrated type
  user.firstName;  // ✅ string — the raw field
}

// ⚠️ Virtuals are NOT serialized by default. Two things to remember:
//  1. toJSON/toObject need { virtuals: true } to include them
//  2. .lean() drops virtuals entirely — the type reflects this if you use FlattenMaps
userSchema.set("toJSON",   { virtuals: true });
userSchema.set("toObject", { virtuals: true });
```

**Virtual populate** — a reverse reference that is also typed through the virtuals interface:

```ts
interface IOrganizationVirtuals {
  members: HydratedDocument<IUser>[];   // populated on demand
}

interface OrgModelType extends Model<IOrganization, {}, {}, IOrganizationVirtuals> {}

const orgSchema = new Schema<IOrganization, OrgModelType, {}, {}, IOrganizationVirtuals>({
  name: { type: String, required: true },
});

orgSchema.virtual("members", {
  ref:          "User",     // the model to pull from
  localField:   "_id",      // this doc's field
  foreignField: "orgId",    // the User field pointing back here
  justOne:      false,      // → an array
});

const org = await OrgModel.findById(orgId).populate("members");
org?.members[0].firstName;  // ✅ typed as HydratedDocument<IUser>[]
```

### Concept 5 — `.lean()`, `FlattenMaps<T>`, and why `_id` is still an `ObjectId`

`.lean()` skips document hydration and returns raw POJOs straight from the BSON driver. It is 3–5x faster and uses far less memory — the right default for read-only endpoints. Its return type is:

```ts
// For a findOne:
FlattenMaps<IUser> & { _id: Types.ObjectId } | null

// For a find:
(FlattenMaps<IUser> & { _id: Types.ObjectId })[]
```

Two things confuse people here.

**(a) `FlattenMaps<T>` exists because of `Map` fields.** Mongoose supports `Map<string, V>` schema types. A hydrated document exposes them as a real JS `Map`; a lean result exposes them as a plain object. `FlattenMaps<T>` recursively rewrites `Map<string, V>` → `Record<string, V>` so the type matches reality:

```ts
interface IUser {
  _id:      Types.ObjectId;
  email:    string;
  settings: Map<string, string>;   // stored as a Mongo sub-document
}

const doc  = await UserModel.findById(userId);          // HydratedDocument<IUser>
doc?.settings.get("theme");                             // ✅ real Map API

const lean = await UserModel.findById(userId).lean();   // FlattenMaps<IUser> & { _id: ObjectId }
lean?.settings["theme"];                                // ✅ plain object index access
// lean?.settings.get("theme");                         // ❌ Property 'get' does not exist
```

If your schema has no `Map` fields, `FlattenMaps<IUser>` is structurally identical to `IUser` — it just looks scary in error messages.

**(b) `_id` is *still* a `Types.ObjectId`, not a string.** `.lean()` removes the *Mongoose* wrapper, not the *BSON* types. `_id`, `Date`, `Buffer` and nested `ObjectId` refs all survive lean untouched:

```ts
const lean = await UserModel.findById(userId).lean();
if (lean) {
  lean._id;                 // Types.ObjectId — NOT string
  lean._id.toHexString();   // ✅ this is how you get the string
  // JSON.stringify(lean)._id → "507f1f77bcf86cd799439011" (ObjectId has a toJSON)
}

// Lean results have no document methods — the compiler enforces it:
// lean.save();          ❌ Property 'save' does not exist
// lean.toObject();      ❌ Property 'toObject' does not exist
// lean.fullName;        ❌ virtuals are dropped by lean
```

You can override the lean generic when you want the *post-serialization* shape:

```ts
// A DTO describing what leaves your API — ids as strings:
interface UserDto { _id: string; email: string; createdAt: string }

// .lean<T>() lets you assert the shape. ⚠️ This is an ASSERTION — Mongoose does not
// convert anything. Only do this if you actually transform the data afterwards.
const rows = await UserModel.find().lean<UserDto[]>();
```

### Concept 6 — `Types.ObjectId` vs `string` — the single biggest source of bugs

MongoDB ids are 12-byte BSON `ObjectId`s. Your HTTP layer only ever has strings. The two are **never `===`** and TypeScript will tell you so:

```ts
import { Types, isValidObjectId } from "mongoose";

const paramId: string = requestBody.userId;
const user = await UserModel.findById(paramId);   // ✅ findById accepts a string and casts

if (user) {
  // ❌ Always false — comparing an object to a string:
  if (user._id === paramId) { /* unreachable */ }
  //     ~~~~~~~~~~~~~~~~~ Error: This comparison appears unintentional because the
  //                       types 'ObjectId' and 'string' have no overlap.

  // ✅ Correct comparisons — pick one:
  if (user._id.equals(paramId))             { /* ObjectId.equals accepts string */ }
  if (user._id.toHexString() === paramId)   { /* explicit string comparison */ }
  if (String(user._id) === paramId)         { /* works, less explicit */ }
}

// Validate untrusted input BEFORE hitting the DB — an invalid id throws a CastError:
function toObjectId(raw: string): Types.ObjectId {
  if (!isValidObjectId(raw)) throw new Error(`Invalid ObjectId: ${raw}`);
  return new Types.ObjectId(raw);
}

// In filters, string vs ObjectId both work (Mongoose casts) — but the TYPE must match
// the interface, so declare refs as Types.ObjectId and cast at the edge:
const filter: FilterQuery<IUser> = { orgId: toObjectId(requestBody.orgId) };
```

A useful discipline: **`Types.ObjectId` never escapes the repository layer.** Convert to `string` in the mapper that produces your API DTOs, and convert back in the repository. Then `Types.ObjectId` appears in exactly two files instead of forty.

### Concept 7 — Typed queries: `FilterQuery`, `UpdateQuery`, `ProjectionType`

Mongoose exports helper types so you can pass filters and updates around as typed values:

```ts
import { FilterQuery, UpdateQuery, ProjectionType, QueryOptions, SortOrder } from "mongoose";

// FilterQuery<IUser> — keys must exist on IUser; values may be the field type
// OR a Mongo query operator object:
const filter: FilterQuery<IUser> = {
  role:      "admin",                       // exact match
  createdAt: { $gte: new Date("2026-01-01") },
  email:     { $in: ["ada@example.com", "grace@example.com"] },
  $or: [{ role: "admin" }, { lastLoginAt: null }],
  // rol: "admin"   ❌ 'rol' does not exist in type FilterQuery<IUser>
};

// UpdateQuery<IUser> — operators are checked against the field types:
const update: UpdateQuery<IUser> = {
  $set:   { role: "member" },
  $unset: { lastLoginAt: 1 },
  $inc:   { loginCount: 1 },
  // $set: { role: "superadmin" }  ❌ not assignable to "admin" | "member"
};

// ProjectionType<IUser> — which fields to return:
const projection: ProjectionType<IUser> = { email: 1, role: 1 };

const options: QueryOptions<IUser> = { sort: { createdAt: -1 }, limit: 20 };

const admins = await UserModel.find(filter, projection, options).lean();
```

⚠️ **Projections do not narrow the return type.** `find(filter, { email: 1 })` still returns the full `IUser` type even though `role`, `passwordHash` etc. are `undefined` at runtime. This is a genuine hole in Mongoose's typings — see *Going deeper*.

### Concept 8 — Typing `populate()`

A reference field is stored as an `ObjectId`. After `populate()`, it is a document. These are different types, and the honest way to model that is a second interface:

```ts
interface IOrganization {
  _id:  Types.ObjectId;
  name: string;
  plan: "free" | "pro" | "enterprise";
}

interface IUser {
  _id:   Types.ObjectId;
  email: string;
  orgId: Types.ObjectId;          // ← unpopulated shape
}

// The populated shape — same doc, one field swapped:
type PopulatedUser = Omit<IUser, "orgId"> & { orgId: IOrganization };

// Mongoose's populate generic lets you declare which paths changed:
const user = await UserModel
  .findById(userId)
  .populate<{ orgId: IOrganization }>("orgId");
// user: (HydratedDocument<IUser> & { orgId: IOrganization }) | null

user?.orgId.name;   // ✅ string
user?.orgId.plan;   // ✅ "free" | "pro" | "enterprise"

// Multiple paths:
const order = await OrderModel
  .findById(orderId)
  .populate<{ userId: IUser; orgId: IOrganization }>(["userId", "orgId"]);

order?.userId.email;  // ✅
order?.orgId.name;    // ✅

// Combined with lean — populate generic first, then lean:
const leanUser = await UserModel
  .findById(userId)
  .populate<{ orgId: IOrganization }>("orgId")
  .lean();
leanUser?.orgId.name;  // ✅ and no document methods

// ⚠️ The populate generic is an ASSERTION, not a proof. If you typo the path name,
// Mongoose populates nothing and the type still claims it is populated:
const wrong = await UserModel.findById(userId).populate<{ orgId: IOrganization }>("orgID");
//                                                                                ~~~~~~ runtime: not populated
// wrong.orgId.name → TypeError at runtime, no compile error.
```

For codebases where the same model is sometimes populated and sometimes not, model it as a **discriminated pair** and use a type guard:

```ts
type MaybePopulatedUser = HydratedDocument<IUser> & { orgId: Types.ObjectId | IOrganization };

function isOrgPopulated(
  user: MaybePopulatedUser,
): user is HydratedDocument<IUser> & { orgId: IOrganization } {
  // An unpopulated value is an ObjectId; a populated one is a document with `name`
  return user.orgId instanceof Types.ObjectId === false;
}

if (isOrgPopulated(user)) {
  user.orgId.name;   // ✅ narrowed
} else {
  user.orgId.toHexString();  // ✅ still an ObjectId
}
```

---

## Example 1 — basic

```ts
// A typed User model, end to end, with statics, instance methods, and a virtual.

import { Schema, model, Model, Types, HydratedDocument } from "mongoose";
import bcrypt from "bcryptjs";

// ── Raw document shape ──────────────────────────────────────────────────────
export interface IUser {
  _id:          Types.ObjectId;
  email:        string;
  firstName:    string;
  lastName:     string;
  passwordHash: string;
  role:         "admin" | "member" | "viewer";
  isActive:     boolean;
  lastLoginAt:  Date | null;
  createdAt:    Date;
  updatedAt:    Date;
}

// ── Instance methods ────────────────────────────────────────────────────────
export interface IUserMethods {
  verifyPassword(candidate: string): Promise<boolean>;
  recordLogin(): Promise<void>;
}

// ── Virtuals ────────────────────────────────────────────────────────────────
export interface IUserVirtuals {
  fullName: string;
}

// ── Model type (statics live here) ──────────────────────────────────────────
export interface UserModelType extends Model<IUser, {}, IUserMethods, IUserVirtuals> {
  findByEmail(email: string): Promise<UserDoc | null>;
  findActiveByRole(role: IUser["role"]): Promise<UserDoc[]>;
}

// Convenience alias used everywhere else in the codebase:
export type UserDoc = HydratedDocument<IUser, IUserMethods, IUserVirtuals>;

// ── Schema ──────────────────────────────────────────────────────────────────
const userSchema = new Schema<IUser, UserModelType, IUserMethods, {}, IUserVirtuals>(
  {
    email:        { type: String, required: true, unique: true, lowercase: true, trim: true },
    firstName:    { type: String, required: true, trim: true },
    lastName:     { type: String, required: true, trim: true },
    passwordHash: { type: String, required: true, select: false },  // excluded by default
    role:         { type: String, enum: ["admin", "member", "viewer"], default: "member" },
    isActive:     { type: Boolean, default: true },
    lastLoginAt:  { type: Date, default: null },
  },
  {
    timestamps: true,             // provides createdAt / updatedAt (declared on IUser)
    toJSON:   { virtuals: true }, // include fullName when serializing
    toObject: { virtuals: true },
  },
);

// ── Indexes ─────────────────────────────────────────────────────────────────
userSchema.index({ role: 1, isActive: 1 });
userSchema.index({ createdAt: -1 });

// ── Virtual ─────────────────────────────────────────────────────────────────
userSchema.virtual("fullName").get(function (this: IUser): string {
  return `${this.firstName} ${this.lastName}`;
});

// ── Instance methods — `function`, never arrow ──────────────────────────────
userSchema.method("verifyPassword", async function (candidate: string): Promise<boolean> {
  return bcrypt.compare(candidate, this.passwordHash);
});

userSchema.method("recordLogin", async function (): Promise<void> {
  this.lastLoginAt = new Date();
  await this.save();
});

// ── Statics ─────────────────────────────────────────────────────────────────
userSchema.static("findByEmail", function (email: string) {
  return this.findOne({ email: email.toLowerCase() }).select("+passwordHash");
});

userSchema.static("findActiveByRole", function (role: IUser["role"]) {
  return this.find({ role, isActive: true }).sort({ createdAt: -1 });
});

// ── Model ───────────────────────────────────────────────────────────────────
export const UserModel = model<IUser, UserModelType>("User", userSchema);

// ── Usage ───────────────────────────────────────────────────────────────────
async function login(email: string, password: string): Promise<{ authToken: string } | null> {
  const user = await UserModel.findByEmail(email);   // UserDoc | null
  if (!user || !user.isActive) return null;

  const ok = await user.verifyPassword(password);    // ✅ instance method, typed
  if (!ok) return null;

  await user.recordLogin();

  return { authToken: signJwt({ userId: user._id.toHexString(), role: user.role }) };
}
```

---

## Example 2 — real world backend use case

```ts
// A multi-tenant orders service: three models, refs, populate, lean reads,
// transactions, and a repository layer where ObjectId never escapes.

import {
  Schema, model, Model, Types, HydratedDocument,
  FilterQuery, ClientSession, connection,
} from "mongoose";

// ═══ Models ═══════════════════════════════════════════════════════════════

// ── Organization ────────────────────────────────────────────────────────────
export interface IOrganization {
  _id:  Types.ObjectId;
  name: string;
  plan: "free" | "pro" | "enterprise";
}

const organizationSchema = new Schema<IOrganization>({
  name: { type: String, required: true, trim: true },
  plan: { type: String, enum: ["free", "pro", "enterprise"], default: "free" },
});

export const OrganizationModel = model<IOrganization>("Organization", organizationSchema);

// ── User ────────────────────────────────────────────────────────────────────
export interface IUser {
  _id:   Types.ObjectId;
  orgId: Types.ObjectId;
  email: string;
  name:  string;
}

const userSchema = new Schema<IUser>({
  orgId: { type: Schema.Types.ObjectId, ref: "Organization", required: true, index: true },
  email: { type: String, required: true, lowercase: true },
  name:  { type: String, required: true },
});

userSchema.index({ orgId: 1, email: 1 }, { unique: true });

export const UserModel = model<IUser>("User", userSchema);

// ── Order (with a typed sub-document array) ─────────────────────────────────
export interface IOrderItem {
  sku:        string;
  quantity:   number;
  unitCents:  number;
}

export interface IOrder {
  _id:         Types.ObjectId;
  orgId:       Types.ObjectId;
  userId:      Types.ObjectId;
  status:      "pending" | "paid" | "shipped" | "cancelled";
  items:       Types.DocumentArray<IOrderItem>;  // sub-doc array on hydrated docs
  totalCents:  number;
  paymentId:   string | null;
  createdAt:   Date;
  updatedAt:   Date;
}

export interface IOrderMethods {
  recalculateTotal(): number;
  markPaid(paymentId: string, session?: ClientSession): Promise<void>;
}

export interface IOrderVirtuals {
  itemCount: number;
}

export interface OrderModelType extends Model<IOrder, {}, IOrderMethods, IOrderVirtuals> {
  findForOrg(orgId: Types.ObjectId, status?: IOrder["status"]): Promise<OrderDoc[]>;
  revenueForOrg(orgId: Types.ObjectId): Promise<number>;
}

export type OrderDoc = HydratedDocument<IOrder, IOrderMethods, IOrderVirtuals>;

const orderItemSchema = new Schema<IOrderItem>(
  {
    sku:       { type: String, required: true },
    quantity:  { type: Number, required: true, min: 1 },
    unitCents: { type: Number, required: true, min: 0 },
  },
  { _id: false },   // sub-documents without their own _id
);

const orderSchema = new Schema<IOrder, OrderModelType, IOrderMethods, {}, IOrderVirtuals>(
  {
    orgId:      { type: Schema.Types.ObjectId, ref: "Organization", required: true, index: true },
    userId:     { type: Schema.Types.ObjectId, ref: "User", required: true, index: true },
    status:     { type: String, enum: ["pending", "paid", "shipped", "cancelled"], default: "pending" },
    items:      { type: [orderItemSchema], required: true },
    totalCents: { type: Number, required: true, default: 0 },
    paymentId:  { type: String, default: null },
  },
  { timestamps: true, toJSON: { virtuals: true } },
);

orderSchema.index({ orgId: 1, status: 1, createdAt: -1 });

orderSchema.virtual("itemCount").get(function (this: IOrder): number {
  return this.items.reduce((sum, item) => sum + item.quantity, 0);
});

orderSchema.method("recalculateTotal", function (): number {
  this.totalCents = this.items.reduce((sum, i) => sum + i.quantity * i.unitCents, 0);
  return this.totalCents;
});

orderSchema.method("markPaid", async function (paymentId: string, session?: ClientSession) {
  this.status    = "paid";
  this.paymentId = paymentId;
  await this.save({ session });
});

orderSchema.static("findForOrg", function (orgId: Types.ObjectId, status?: IOrder["status"]) {
  const filter: FilterQuery<IOrder> = status ? { orgId, status } : { orgId };
  return this.find(filter).sort({ createdAt: -1 });
});

orderSchema.static("revenueForOrg", async function (orgId: Types.ObjectId): Promise<number> {
  // Aggregations are NOT typed by the schema — annotate the result explicitly:
  const rows = await this.aggregate<{ _id: null; total: number }>([
    { $match: { orgId, status: { $in: ["paid", "shipped"] } } },
    { $group: { _id: null, total: { $sum: "$totalCents" } } },
  ]);
  return rows[0]?.total ?? 0;
});

export const OrderModel = model<IOrder, OrderModelType>("Order", orderSchema);

// ═══ DTO layer — ObjectId never crosses this line ═════════════════════════

export interface OrderItemDto { sku: string; quantity: number; unitCents: number }

export interface OrderDto {
  orderId:    string;
  orgId:      string;
  user:       { userId: string; email: string; name: string } | null;
  status:     IOrder["status"];
  items:      OrderItemDto[];
  totalCents: number;
  createdAt:  string;
}

// The lean+populated read shape — what the mapper actually receives:
type LeanPopulatedOrder = Omit<IOrder, "userId" | "items"> & {
  userId: IUser | null;
  items:  IOrderItem[];
};

function toOrderDto(order: LeanPopulatedOrder): OrderDto {
  return {
    orderId: order._id.toHexString(),
    orgId:   order.orgId.toHexString(),
    user: order.userId
      ? {
          userId: order.userId._id.toHexString(),
          email:  order.userId.email,
          name:   order.userId.name,
        }
      : null,
    status:     order.status,
    items:      order.items.map((i) => ({ sku: i.sku, quantity: i.quantity, unitCents: i.unitCents })),
    totalCents: order.totalCents,
    createdAt:  order.createdAt.toISOString(),
  };
}

// ═══ Repository ═══════════════════════════════════════════════════════════

export interface CreateOrderInput {
  orgId:  string;
  userId: string;
  items:  OrderItemDto[];
}

export class OrderRepository {
  /** Read path — lean + populate, mapped straight to DTOs. */
  async listForOrg(orgId: string, status?: IOrder["status"]): Promise<OrderDto[]> {
    const filter: FilterQuery<IOrder> = status
      ? { orgId: new Types.ObjectId(orgId), status }
      : { orgId: new Types.ObjectId(orgId) };

    const rows = await OrderModel
      .find(filter)
      .populate<{ userId: IUser }>("userId", "email name")   // projection on the ref
      .sort({ createdAt: -1 })
      .limit(100)
      .lean();

    return rows.map(toOrderDto);
  }

  /** Single read — returns null instead of throwing. */
  async findById(orderId: string): Promise<OrderDto | null> {
    if (!Types.ObjectId.isValid(orderId)) return null;

    const row = await OrderModel
      .findById(orderId)
      .populate<{ userId: IUser }>("userId", "email name")
      .lean();

    return row ? toOrderDto(row) : null;
  }

  /** Write path — hydrated document so instance methods and validation run. */
  async create(input: CreateOrderInput): Promise<OrderDto> {
    const order = new OrderModel({
      orgId:  new Types.ObjectId(input.orgId),
      userId: new Types.ObjectId(input.userId),
      items:  input.items,
      status: "pending",
    });

    order.recalculateTotal();     // ✅ typed instance method
    await order.save();

    const populated = await order.populate<{ userId: IUser }>("userId", "email name");
    return toOrderDto(populated.toObject());
  }

  /** Transactional write — the session is typed as ClientSession. */
  async payOrder(orderId: string, paymentId: string): Promise<OrderDto> {
    const session: ClientSession = await connection.startSession();
    try {
      let dto: OrderDto | null = null;

      await session.withTransaction(async () => {
        const order = await OrderModel.findById(orderId).session(session);
        if (!order)                    throw new Error("Order not found");
        if (order.status !== "pending") throw new Error(`Cannot pay order in status ${order.status}`);

        await order.markPaid(paymentId, session);

        await OrganizationModel.updateOne(
          { _id: order.orgId },
          { $inc: { lifetimeRevenueCents: order.totalCents } },
          { session },
        );

        const populated = await order.populate<{ userId: IUser }>("userId", "email name");
        dto = toOrderDto(populated.toObject());
      });

      if (!dto) throw new Error("Transaction produced no result");
      return dto;
    } finally {
      await session.endSession();
    }
  }

  async revenue(orgId: string): Promise<number> {
    return OrderModel.revenueForOrg(new Types.ObjectId(orgId));   // ✅ typed static
  }
}
```

---

## Going deeper

### Projections do not narrow the return type

This is the biggest honest gap in Mongoose's typings:

```ts
// You asked for two fields. The type says you got all of them.
const user = await UserModel.findById(userId).select("email role").lean();
user?.passwordHash;   // ✅ compiles — ❌ undefined at runtime
```

Mongoose cannot narrow this because `select()` accepts strings, objects, exclusion syntax (`"-passwordHash"`), and nested paths. Two defences:

```ts
// (a) Explicit generic on lean, describing what you actually selected:
type UserListRow = Pick<IUser, "_id" | "email" | "role">;
const rows = await UserModel.find().select("email role").lean<UserListRow[]>();
rows[0].passwordHash;  // ❌ Property 'passwordHash' does not exist — good

// (b) A helper that keeps the projection and the type in sync:
function projection<T, K extends keyof T>(...keys: K[]): Record<string, 1> {
  return Object.fromEntries(keys.map((k) => [k, 1]));
}
const rows2 = await UserModel
  .find({}, projection<IUser, "email" | "role">("email", "role"))
  .lean<Pick<IUser, "_id" | "email" | "role">[]>();
```

### `select: false` fields and the `+field` escape hatch

A field marked `select: false` in the schema is omitted from every query by default — but its type stays required on `IUser`. That means the compiler will happily let you read a `passwordHash` that isn't there:

```ts
passwordHash: { type: String, required: true, select: false }

const user = await UserModel.findById(userId);
await bcrypt.compare(pw, user!.passwordHash);   // compiles; passwordHash is undefined at runtime

// ✅ Opt it back in explicitly — and centralise it in a static so it's not forgotten:
const authUser = await UserModel.findById(userId).select("+passwordHash");
```

There is no type-level fix in stock Mongoose. The pragmatic pattern: make the *only* way to fetch credentials a static (`findByEmail` above) that always includes `+passwordHash`, and never `findOne` for auth anywhere else.

### `timestamps: true` and interface drift

`{ timestamps: true }` adds `createdAt` and `updatedAt` at runtime. They are *not* added to your interface — you must declare them yourself, and they must be non-optional or your DTO mappers will fight you:

```ts
interface IUser {
  // ...
  createdAt: Date;   // declared by you, populated by Mongoose
  updatedAt: Date;
}
```

Conversely, on the *input* side those fields must not be required. Use a derived input type rather than a second hand-maintained interface:

```ts
type CreateUserInput = Omit<IUser, "_id" | "createdAt" | "updatedAt">;

async function createUser(input: CreateUserInput): Promise<UserDoc> {
  return UserModel.create(input);   // ✅ _id and timestamps generated by Mongoose
}
```

### Defaults make fields present at read time but optional at write time

A field with `default:` is always present when you read, but you don't have to supply it when you write. `IUser` can only express one of those. Declare it **required** (read-truth) and strip it in the input type:

```ts
interface IUser { role: "admin" | "member"; /* has a schema default */ }

type CreateUserInput = Omit<IUser, "_id" | "createdAt" | "updatedAt" | "role">
                     & Partial<Pick<IUser, "role">>;   // role optional on write
```

### Never use arrow functions for methods, statics, virtuals, or hooks

Mongoose binds `this` when it calls your function. Arrow functions capture the enclosing `this` — which is the module scope — so `this.email` is a compile error *and* a runtime bug:

```ts
// ❌ Arrow — `this` is not the document:
userSchema.method("isAdmin", () => this.role === "admin");
//                                  ~~~~ 'this' implicitly has type 'any'

// ✅ function expression:
userSchema.method("isAdmin", function (): boolean { return this.role === "admin"; });
```

The same applies to `pre`/`post` hooks:

```ts
userSchema.pre("save", async function (next) {
  // `this` is HydratedDocument<IUser>
  if (!this.isModified("passwordHash")) return next();
  this.passwordHash = await bcrypt.hash(this.passwordHash, 12);
  next();
});

// ⚠️ Query middleware has a DIFFERENT `this` — it's the Query, not the document:
userSchema.pre("findOneAndUpdate", function (next) {
  // this: Query<...>
  this.setOptions({ new: true, runValidators: true });
  next();
});
```

### `aggregate()` is completely untyped — annotate it

An aggregation pipeline can produce any shape, so Mongoose can't infer it. `aggregate<T>()` takes an explicit generic and it is an **unchecked assertion**:

```ts
interface RevenueByStatus { _id: IOrder["status"]; total: number; count: number }

const rows = await OrderModel.aggregate<RevenueByStatus>([
  { $match: { orgId } },
  { $group: { _id: "$status", total: { $sum: "$totalCents" }, count: { $sum: 1 } } },
]);
// rows: RevenueByStatus[] — TypeScript believes you. Nothing verifies the pipeline.
// Treat aggregate generics with the same suspicion as `as` casts.
```

### `strict` mode, unknown fields, and the compile/runtime gap

Schema `strict: true` (the default) silently *drops* fields not in the schema on write. TypeScript catches this at compile time for object literals but not for spread objects typed as `any`:

```ts
await UserModel.create({ email, isAdmin: true });
//                              ~~~~~~~ ❌ not in IUser — good, caught

const body = requestBody as any;
await UserModel.create(body);   // compiles; unknown fields silently dropped at runtime
```

Never pass an unvalidated `req.body` to `create()` or `updateOne()`. Parse it into a typed DTO first (see *57 — Typing database queries with pg* for the same discipline on the SQL side, and *41 — Type guards* for the validation shape).

### `findOneAndUpdate` returns `T | null` even with `upsert: true`

```ts
// Even though upsert guarantees a document, the type is nullable:
const user = await UserModel.findOneAndUpdate(
  { email },
  { $setOnInsert: { role: "member" } },
  { upsert: true, new: true },
);
// user: HydratedDocument<IUser> | null

// ✅ Assert with a runtime guard, not with `!`:
if (!user) throw new Error("Upsert unexpectedly returned null");
```

### Performance: `.lean()` is not just a typing choice

Hydration allocates a Mongoose document, getters, setters, change tracking, and virtuals per row. On a 1000-row read that is measurably slower and heavier than lean. The typing follows the runtime:

| | Hydrated | `.lean()` |
|---|---|---|
| Return type | `HydratedDocument<IUser>` | `FlattenMaps<IUser> & { _id: ObjectId }` |
| `save()`, `markModified()` | ✅ | ❌ (compile error) |
| Virtuals | ✅ | ❌ dropped |
| Getters / setters | ✅ | ❌ |
| Relative speed | 1x | ~3–5x faster |
| Use for | writes, validation | read-only endpoints |

### Sub-documents: `Types.DocumentArray<T>` vs `T[]`

On a hydrated document, an array of sub-documents is a `Types.DocumentArray<T>` — it has `.id()`, `.push()` with casting, and each element has `_id` and `.toObject()`. After `.lean()` it is a plain `T[]`. Declare the interface with `Types.DocumentArray<IOrderItem>` if you use those APIs, and map to `IOrderItem[]` in your lean/DTO types (as Example 2 does with `LeanPopulatedOrder`).

---

## Common mistakes

### Mistake 1 — Comparing `_id` to a string with `===`

```ts
// ❌ Always false — an ObjectId object is never === a string:
async function assertOwnership(orderId: string, userId: string): Promise<void> {
  const order = await OrderModel.findById(orderId);
  if (order && order.userId === userId) {
    //          ~~~~~~~~~~~~~~~~~~~~~~ Error: types 'ObjectId' and 'string' have no overlap
    return;
  }
  throw new Error("Forbidden");
}

// ✅ Use ObjectId.equals — it accepts strings, ObjectIds, and documents:
async function assertOwnership(orderId: string, userId: string): Promise<void> {
  const order = await OrderModel.findById(orderId);
  if (order && order.userId.equals(userId)) return;
  throw new Error("Forbidden");
}
```

Without `strictNullChecks`-era comparison checking this can even compile silently in JS-flavoured configs — the fix is the same.

### Mistake 2 — Calling document methods on `.lean()` results

```ts
// ❌ .lean() returns POJOs; save/virtuals/methods do not exist:
async function deactivateUser(userId: string): Promise<void> {
  const user = await UserModel.findById(userId).lean();
  if (!user) return;
  user.isActive = false;
  await user.save();
  //       ~~~~ Error: Property 'save' does not exist on type
  //            'FlattenMaps<IUser> & { _id: ObjectId }'
}

// ✅ Either hydrate for writes...
async function deactivateUser(userId: string): Promise<void> {
  const user = await UserModel.findById(userId);   // hydrated
  if (!user) return;
  user.isActive = false;
  await user.save();
}

// ✅ ...or skip hydration entirely and issue an update:
async function deactivateUserFast(userId: string): Promise<void> {
  await UserModel.updateOne({ _id: userId }, { $set: { isActive: false } });
}
```

### Mistake 3 — Using `IUser` where `HydratedDocument<IUser>` is required

```ts
// ❌ The signature promises data, but the body needs a document:
function auditUser(user: IUser): void {
  logger.info("user touched", {
    userId:   user._id.toHexString(),
    modified: user.isModified("email"),
    //             ~~~~~~~~~~ Error: Property 'isModified' does not exist on type 'IUser'
  });
}

// ✅ Ask for the document type when you need document behaviour:
function auditUser(user: HydratedDocument<IUser>): void {
  logger.info("user touched", {
    userId:   user._id.toHexString(),
    modified: user.isModified("email"),
  });
}

// ✅ And ask for IUser (or a Pick of it) when you only need the data —
//    that way lean results and hydrated documents both satisfy the signature:
function formatUserLabel(user: Pick<IUser, "firstName" | "lastName" | "email">): string {
  return `${user.firstName} ${user.lastName} <${user.email}>`;
}
```

### Mistake 4 — Trusting the `populate<T>()` generic

```ts
// ❌ The generic is an assertion. A wrong path name compiles and crashes at runtime:
const order = await OrderModel.findById(orderId).populate<{ user: IUser }>("user");
//                                                                         ~~~~~~
// The schema field is `userId`, not `user`. Mongoose populates nothing.
order?.user.email;   // TypeScript: string. Runtime: TypeError on undefined.

// ✅ Keep the generic key identical to the schema path, and derive the type
//    from the interface so a field rename breaks the build:
const order2 = await OrderModel
  .findById(orderId)
  .populate<Pick<Record<keyof IOrder, IUser>, "userId">>("userId");
//   → { userId: IUser }

// ✅ Simpler and equally safe in practice: a named helper type per populate.
type OrderWithUser = Omit<IOrder, "userId"> & { userId: IUser };
const order3 = await OrderModel.findById(orderId).populate<{ userId: IUser }>("userId");
const typed: OrderWithUser | null = order3 ? order3.toObject() : null;
```

### Mistake 5 — Arrow functions for methods, statics, and hooks

```ts
// ❌ `this` is not the document — compile error under noImplicitThis:
userSchema.method("verifyPassword", async (candidate: string) =>
  bcrypt.compare(candidate, this.passwordHash),
  //                        ~~~~ 'this' implicitly has type 'any'
);

// ✅ function expression — Mongoose binds `this` to the hydrated document:
userSchema.method("verifyPassword", async function (candidate: string): Promise<boolean> {
  return bcrypt.compare(candidate, this.passwordHash);
});
```

---

## Practice exercises

### Exercise 1 — easy

Model a **Product catalogue** collection with full Mongoose typing.

Requirements:

1. Write an `IProduct` interface with: `_id: Types.ObjectId`, `sku: string`, `name: string`, `description: string | null`, `priceCents: number`, `currency: "USD" | "EUR" | "GBP"`, `stockCount: number`, `isPublished: boolean`, `tags: string[]`, `createdAt: Date`, `updatedAt: Date`.
2. Build `productSchema` with `new Schema<IProduct>(...)`, using `timestamps: true`, a unique index on `sku`, and a compound index on `{ isPublished: 1, createdAt: -1 }`.
3. Add a virtual `priceDisplay: string` (e.g. `"$19.99"`) — type it via an `IProductVirtuals` interface.
4. Add an instance method `isInStock(): boolean` — type it via `IProductMethods`.
5. Export `type ProductDoc = HydratedDocument<IProduct, IProductMethods, IProductVirtuals>`.
6. Write `type CreateProductInput` derived from `IProduct` with `Omit`/`Partial` so that `_id`, `createdAt`, `updatedAt` are excluded and `isPublished`, `stockCount`, `tags` are optional.
7. Write `async function publishProduct(sku: string): Promise<ProductDoc | null>` that finds by sku, sets `isPublished`, and saves.
8. Write `async function listPublished(): Promise<Array<Pick<IProduct, "_id" | "sku" | "name" | "priceCents">>>` using `.lean()` with the correct explicit generic.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a **typed Comment thread** model with statics, references, and populate typing.

Requirements:

1. Interfaces: `IUser` (`_id`, `email`, `displayName`), `IPost` (`_id`, `authorId: Types.ObjectId`, `title`, `body`, `createdAt`), `IComment` (`_id`, `postId: Types.ObjectId`, `authorId: Types.ObjectId`, `parentCommentId: Types.ObjectId | null`, `body`, `isDeleted: boolean`, `createdAt`).
2. A `CommentModelType extends Model<IComment, {}, ICommentMethods>` with statics:
   - `findThreadForPost(postId: Types.ObjectId): Promise<CommentDoc[]>` — non-deleted, sorted oldest first
   - `countForPost(postId: Types.ObjectId): Promise<number>`
3. An instance method `softDelete(): Promise<void>` that sets `isDeleted` and blanks the body.
4. A read function `getThread(postId: string)` that:
   - uses `.populate<{ authorId: IUser }>("authorId", "displayName email")`
   - uses `.lean()`
   - returns `CommentDto[]` where `CommentDto` has `commentId: string`, `parentCommentId: string | null`, `author: { userId: string; displayName: string }`, `body: string`, `createdAt: string` — i.e. **no `ObjectId` and no `Date` escapes the function**.
5. A `buildTree(comments: CommentDto[]): CommentNode[]` function where `CommentNode = CommentDto & { replies: CommentNode[] }` — nest by `parentCommentId`.
6. A type guard `isPopulatedAuthor(c: unknown): c is { authorId: IUser }` used to prove at runtime that populate succeeded before mapping.

Every function signature must be explicitly annotated. No `any`, no non-null `!` assertions.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a **generic typed repository layer** on top of Mongoose that enforces the ObjectId/string boundary at the type level.

Requirements:

1. Define a base document constraint:
   ```ts
   interface BaseDoc { _id: Types.ObjectId; createdAt: Date; updatedAt: Date }
   ```
2. Write a mapped type `Serialized<T>` that recursively converts, for any `T`:
   - `Types.ObjectId` → `string`
   - `Date` → `string`
   - arrays and nested objects → recursively serialized
   - leaves primitives alone
   (Use conditional + mapped types; see *44 — Conditional types* and *43 — Mapped types*.)
3. Write `abstract class BaseRepository<TRaw extends BaseDoc, TModel extends Model<TRaw>>` with:
   - `protected abstract readonly model: TModel`
   - `findById(id: string): Promise<Serialized<TRaw> | null>` — validates the id, uses `.lean()`
   - `findMany(filter: FilterQuery<TRaw>, options?: { limit?: number; sort?: Record<string, 1 | -1> }): Promise<Serialized<TRaw>[]>`
   - `create(input: Omit<TRaw, "_id" | "createdAt" | "updatedAt">): Promise<Serialized<TRaw>>`
   - `updateById(id: string, patch: UpdateQuery<TRaw>): Promise<Serialized<TRaw> | null>`
   - `deleteById(id: string): Promise<boolean>`
   - `withTransaction<R>(fn: (session: ClientSession) => Promise<R>): Promise<R>`
   - a `protected serialize(doc: unknown): Serialized<TRaw>` runtime mapper matching the type
4. Write a concrete `UserRepository extends BaseRepository<IUser, UserModelType>` that adds `findByEmail(email: string): Promise<Serialized<IUser> | null>`.
5. Prove the boundary holds: write a function `handleGetUser(userId: string): Promise<{ userId: string; createdAt: string }>` that consumes the repository and show that **no `Types.ObjectId` or `Date` type appears anywhere in its signature or body**.
6. Show a compile error for each of: passing `_id` to `create()`, calling `.save()` on a repository result, and comparing a serialized `_id` to an `ObjectId`.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
import {
  Schema, model, Model, Types, HydratedDocument,
  FilterQuery, UpdateQuery, ProjectionType, FlattenMaps, InferSchemaType, ClientSession,
} from "mongoose";

// ── The four pieces ─────────────────────────────────────────────────────────
interface IUser        { _id: Types.ObjectId; email: string }        // raw data
interface IUserMethods { isAdmin(): boolean }                        // instance methods
interface IUserVirtuals{ fullName: string }                          // virtuals
interface UserModelType extends Model<IUser, {}, IUserMethods, IUserVirtuals> {
  findByEmail(email: string): Promise<HydratedDocument<IUser, IUserMethods> | null>;   // statics
}

// ── Schema generics: <TRaw, TModel, TInstanceMethods, TQueryHelpers, TVirtuals>
const s = new Schema<IUser, UserModelType, IUserMethods, {}, IUserVirtuals>({ /* ... */ });

s.method("isAdmin",   function () { return this.email.endsWith("@corp"); });  // `this` = doc
s.static("findByEmail", function (email: string) { return this.findOne({ email }); }); // `this` = model
s.virtual("fullName").get(function (this: IUser) { return this.email.split("@")[0]; });

const UserModel = model<IUser, UserModelType>("User", s);

// ── Return types ────────────────────────────────────────────────────────────
await UserModel.findById(id);            // HydratedDocument<IUser, IUserMethods> | null
await UserModel.find();                  // HydratedDocument<IUser, IUserMethods>[]
await UserModel.findById(id).lean();     // (FlattenMaps<IUser> & { _id: Types.ObjectId }) | null
await UserModel.find().lean<UserDto[]>();// UserDto[]  ⚠️ unchecked assertion
await UserModel.aggregate<Row>([...]);   // Row[]      ⚠️ unchecked assertion

// ── ObjectId ↔ string ───────────────────────────────────────────────────────
new Types.ObjectId(str);      // string → ObjectId (throws if malformed)
oid.toHexString();            // ObjectId → string
oid.equals(str);              // ✅ correct comparison
Types.ObjectId.isValid(str);  // validate before casting
// oid === str                // ❌ compile error, always false

// ── Populate ────────────────────────────────────────────────────────────────
await UserModel.findById(id).populate<{ orgId: IOrganization }>("orgId");
// → (HydratedDocument<IUser> & { orgId: IOrganization }) | null

// ── Derived input types ─────────────────────────────────────────────────────
type CreateUserInput = Omit<IUser, "_id" | "createdAt" | "updatedAt">;
```

| Symbol | What it is | Gotcha |
|---|---|---|
| `IUser` | The raw document data | No `save()`, no virtuals |
| `HydratedDocument<T, M, V>` | Data + document API + methods + virtuals | What every non-lean query returns |
| `Model<T, Q, M, V>` | The collection-level type; extend it for statics | Generic order: raw, query helpers, methods, virtuals |
| `Schema<T, TModel, M, Q, V>` | Checks the schema definition against `T` | Different generic order than `Model` |
| `FlattenMaps<T>` | `Map<K,V>` → `Record<K,V>`, recursively | Only meaningful if you use `Map` schema types |
| `Types.ObjectId` | BSON 12-byte id | Survives `.lean()`; never `===` a string |
| `FilterQuery<T>` | Typed Mongo filter | Keys checked, operator values loosely checked |
| `UpdateQuery<T>` | Typed `$set`/`$inc`/`$unset` | Values checked against field types |
| `InferSchemaType<typeof s>` | Type derived *from* the schema | Use as a cross-check, not a source of truth |
| `.lean<T>()` / `.aggregate<T>()` | Explicit result generic | **Unchecked assertion** — nothing validates it |

---

## Connected topics

- **57 — Typing database queries with pg (PostgreSQL)** — the same boundary problem on the SQL side, where row generics are also unchecked assertions.
- **28 — Generic interfaces** — `Model<T>`, `HydratedDocument<T>` and `FilterQuery<T>` are generic interfaces; understanding the slots makes Mongoose's error messages readable.
- **32 — Utility types** — `Omit`, `Pick`, `Partial` and `Record` do all the heavy lifting for input DTOs, lean projections and populate shapes.
- **41 — Type guards** — `isOrgPopulated(user): user is ... & { orgId: IOrganization }` is how you make a populate assertion actually safe.
- **42 — Discriminated unions** — model an order's `status` as a discriminated union in the domain layer, then flatten it to a single `status` string field for storage.
