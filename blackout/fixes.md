# Blackout: fixes

Multiplayer web board game. A server on an edge runtime holds one object per room plus one shared registry object.

## Reliability & quality

### A malformed path or a failed registry call stops work half-way (Moderate)

**Why it's a problem.** A request path with a broken escape sequence threw inside the API handler and came back as a bare server error. Separately, when a room had already saved its state but the follow-up call to the shared registry failed, the code stopped there and never sent the update to the players.

**AI-written code**

```ts
// api handler: decodeURIComponent throws on a broken escape such as "%"
const parts = path.split("/").map(decodeURIComponent);
const bad = () => json({ ok: false, message: "Bad request" }, 400);
```

```ts
// room: state is saved first, then a call to another object, then the players are told
await storage.put(KEY, room);
await registry.syncRoom(name, accounts, host); // if this throws, nothing below runs
// ... send replies and broadcasts to the players
```

**Fixed code**

```diff
-  const parts = path.split("/").map(decodeURIComponent);
   const bad = () => json({ ok: false, message: "Bad request" }, 400);
+  let parts: string[];
+  try {
+    parts = path.split("/").map(decodeURIComponent);
+  } catch {
+    return bad();
+  }
```

```diff
-      await this.registry().syncRoom(this.name, activeAccounts(out.room), hostAccount(out.room));
+      try {
+        await this.registry().syncRoom(this.name, activeAccounts(out.room), hostAccount(out.room));
+      } catch (e) {
+        // the state is already saved: still send it to the players
+        console.log(`[room ${this.name}] registry sync failed: ${String(e)}`);
+      }
```

**Status.** Fixed (untested)

**Lesson.** Wrap calls to other services so one failure cannot silently cancel the work that comes after it.
