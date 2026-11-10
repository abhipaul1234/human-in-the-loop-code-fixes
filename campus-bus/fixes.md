# Campus Bus Tracking: fixes

Flask + Socket.IO app that simulates one bus on a campus route, shows it on a map, and tells riders when it is coming.

## Logic & data

### Wait and journey times assume a loop, but the bus drives back and forth (High)

**Why it's a problem.** The simulation turns the bus around at both ends of the route, but the time calculation counted stops "forward, wrapping" as if the route were a circle. With the bus at the last stop and a rider waiting at the first stop, the code added one short straight-line jump and reported a few minutes, while the bus really had the whole route to drive. Journey times for a pickup that comes after the dropoff were equally wrong.

**AI-written code**

```python
# the simulation: the bus bounces between the two ends
if next_idx >= TOTAL_STOPS:
    bus_direction = -1
    next_idx = TOTAL_STOPS - 2
elif next_idx < 0:
    bus_direction = 1
    next_idx = 1

# the ETA code: assumes a circle
def stops_forward(from_idx, to_idx):
    if to_idx >= from_idx:
        return to_idx - from_idx
    return TOTAL_STOPS - from_idx + to_idx

def route_distance_metres(from_idx, to_idx):
    total = 0.0
    for i in range(stops_forward(from_idx, to_idx)):
        a = ROUTE_PATH[(from_idx + i) % TOTAL_STOPS]
        b = ROUTE_PATH[(from_idx + i + 1) % TOTAL_STOPS]
        total += haversine(a['lat'], a['lng'], b['lat'], b['lng'])
    return total
```

**Fixed code**

```python
def next_stop(idx, direction):
    """Same bounce rule as the simulation."""
    nxt = idx + direction
    if nxt >= TOTAL_STOPS:
        return TOTAL_STOPS - 2, -1
    if nxt < 0:
        return 1, 1
    return nxt, direction

def travel(from_idx, direction, to_idx):
    """Distance, number of stops and final direction along the bus's real path."""
    metres, stops, idx = 0.0, 0, from_idx
    while idx != to_idx:
        nxt, direction = next_stop(idx, direction)
        a, b = ROUTE_PATH[idx], ROUTE_PATH[nxt]
        metres += haversine(a['lat'], a['lng'], b['lat'], b['lng'])
        stops += 1
        idx = nxt
    return metres, stops, direction
```

```diff
-    wait_metres   = route_distance_metres(current_idx, pickup_idx)
-    wait_minutes  = metres_to_minutes(wait_metres) if stops_forward(current_idx, pickup_idx) > 0 else 0
-    journey_metres  = route_distance_metres(pickup_idx, dropoff_idx)
+    wait_metres, wait_stops, dir_at_pickup = travel(current_idx, current_dir, pickup_idx)
+    wait_minutes  = metres_to_minutes(wait_metres) if wait_stops > 0 else 0
+    journey_metres, _, _ = travel(pickup_idx, dir_at_pickup, dropoff_idx)
```

**Status.** Fixed (untested)

**Lesson.** Compute estimates from the same movement rule the simulation uses, not from a simpler model of it.

### The "arriving" alert fires at the wrong moment and never fires again (High)

**Why it's a problem.** The alert went off whenever the bus was within 150 m of the pickup. Stops are only 90 to 250 m apart and the bus jumps from stop to stop, so it fired when the bus was at a neighbouring stop, including after it had already passed and was driving away. The text always said "in 2 mins", and the alert was sent once per rider and never re-armed, so a rider who missed the bus got no alert on its next pass.

**AI-written code**

```python
if sub is None or sub.get('alert_sent'):
    continue

dist = haversine(bus_lat, bus_lng, pickup['lat'], pickup['lng'])
if dist < 150:
    emit('proximity_alert',
         {'message': f"Bus is arriving at {pickup['name']} in 2 mins!"}, to=sid)
    sub['alert_sent'] = True        # never reset
```

**Fixed code**

```diff
-            if sub is None or sub.get('alert_sent'):
-                continue
+            if sub is None:
+                continue
 ...
-            if dist < 150:
-                alert = {'message': f"Bus is arriving at {pickup['name']} in 2 mins!", ...}
+            # "approaching" = the bus's NEXT stop (in its direction) is the pickup
+            upcoming, _ = next_stop(current_snap['index'], current_dir)
+            approaching = upcoming == pickup_idx
+            if not approaching and current_snap['index'] != pickup_idx:
+                sub['alert_sent'] = False      # left the area: re-arm for the next pass
+                continue
+            if not approaching or sub.get('alert_sent'):
+                continue
+            alert = {'message': f"Bus is arriving at {pickup['name']} in about {metres_to_minutes(dist)} min!", ...}
```

(Simplified: the real code takes a lock around the update of the stored flag.)

**Status.** Fixed (untested)

**Lesson.** "Near" must mean "heading toward and about to reach", and an alert state needs a way to reset.

## UI / UX

### The timeline and map draw a one-way loop (Moderate)

**Why it's a problem.** The front end marked every stop before the bus's index as "Passed", which is wrong while the bus drives back toward the first stop. It also drew the route as a closed line back to the first stop and only restarted the travelled trail at index 0, so the map showed a loop the bus never drives.

**AI-written code**

```js
const isDone = i < activeIndex;                              // assumes one direction
routePolyline = L.polyline([...latlngs, latlngs[0]], opts);  // closed loop
if (stop_index === 0) travelledCoords = [];                  // reset at one end only
```

**Fixed code**

```diff
-function buildTimeline(routePath, activeIndex, userPickupName, userDropoffName) {
+function buildTimeline(routePath, activeIndex, userPickupName, userDropoffName, direction = 1) {
 ...
-    const isDone     = i < activeIndex;
+    const isDone     = direction > 0 ? i < activeIndex : i > activeIndex;
```

```diff
-  routePolyline = L.polyline([...latlngs, latlngs[0]], {
+  routePolyline = L.polyline(latlngs, {
```

```diff
-  if (stop_index === 0) travelledCoords = [];
+  if (stop_index === 0 || stop_index === route_path.length - 1) travelledCoords = [];
```

(The server now also sends the bus's `direction` with each location update.)

**Status.** Fixed (untested)

**Lesson.** When the movement is back-and-forth, every "before/after" check in the UI needs the direction.

## Reliability & quality

### Stop list copied into the browser, plus dead code (Moderate)

**Why it's a problem.** The nine stop names were hard-coded in the browser even though the server already sends the route. Renaming or adding a stop on the server would silently break the dropdowns ("Invalid stop names"). A first `IntersectionObserver` was created and never used, and the same logic was repeated just below it.

**AI-written code**

```js
const STOPS = ['Stop A', 'Stop B', 'Stop C' /* ...copied from the server... */];
STOPS.forEach(name => { /* add <option> to both dropdowns */ });

new IntersectionObserver(callback, { threshold: .45 }).observe.bind(null); // never used
sections.forEach(s => new IntersectionObserver(callback, { threshold: .45 }).observe(s));
```

**Fixed code**

```diff
-const STOPS = ['Stop A', 'Stop B', 'Stop C', ...];
-STOPS.forEach(name => { ... });
+function populateStops(routePath) {
+  [pickupSelect, dropoffSelect].forEach(sel => {
+    [...sel.options].forEach(o => { if (o.value) o.remove(); });   // keep the placeholder
+    routePath.forEach(stop => { /* add <option> for stop.name */ });
+  });
+}
+socket.on('route_info', (data) => { populateStops(data.route_path); /* ... */ });
```

```diff
-new IntersectionObserver((entries) => { ... }, { threshold: .45 }).observe.bind(null);
```

**Status.** Fixed (untested)

**Lesson.** Keep one source of truth for shared data, and delete code that is created but never used.
