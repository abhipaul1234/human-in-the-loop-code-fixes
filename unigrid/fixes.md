# UniGrid: fixes

A Next.js + PostgreSQL university management app with student and faculty directories, courses and a help desk.

## Logic & data

### Help-desk request IDs collide (Moderate)

**Why it's a problem.** Request IDs were `REQ-<date>-<random number from 1 to 999>`, and the column is `UNIQUE`. With only 999 values, a duplicate on the same day becomes likely after a few dozen tickets (about a 50% chance by roughly 37 tickets), and the insert then failed with a server error and the ticket was lost.

**AI-written code**

```ts
function generateRequestId(): string {
  const now = new Date()
  const dateStr = now.getFullYear().toString() +
    (now.getMonth() + 1).toString().padStart(2, '0') +
    now.getDate().toString().padStart(2, '0')
  const randomNum = Math.floor(Math.random() * 999) + 1
  return `REQ-${dateStr}-${randomNum.toString().padStart(3, '0')}`
}
```

**Fixed code**

```diff
+import { randomBytes } from 'crypto'
+
-// Generate Request ID
+// REQ-YYYYMMDD-XXXXXX: 6 random hex characters (about 16.7 million per day); still fits VARCHAR(20)
 function generateRequestId(): string {
   const now = new Date()
   const dateStr = ...
-  const randomNum = Math.floor(Math.random() * 999) + 1
-  return `REQ-${dateStr}-${randomNum.toString().padStart(3, '0')}`
+  return `REQ-${dateStr}-${randomBytes(3).toString('hex').toUpperCase()}`
 }
```

```ts
// a duplicate request_id is retried with a new one instead of losing the ticket
for (let attempt = 0; ; attempt++) {
  try {
    return await insertRequest(generateRequestId())
  } catch (error: any) {
    if (error?.code !== '23505' || attempt >= 2) throw error   // 23505 = unique violation
  }
}
```

(Simplified: `insertRequest` stands for the existing INSERT statement.)

**Status.** Fixed (untested)

**Lesson.** A small random range plus a `UNIQUE` column is a bug waiting for the busy day; use a wide ID space and retry on a clash.
