# API Explorer — Build-Plan Wayfinding

A wayfinder map charting the route from the finished
[API Explorer design](../README.md) to a **sequenced, sized implementation plan** — a
spec a team can pick up and build from.

This is a *planning* tracker, not design intent: the map indexes decisions as they are
made and points at the tickets that hold their detail.

* [map.md](./map.md) — the map: destination, notes, decisions-so-far, and the fog ahead.
* [issues/](./issues/) — one file per decision ticket, child of the map.

Each ticket carries a `Type:` (`grilling`/`research`/`prototype`/`task`), a `Status:`
(`open`/`claimed`/`resolved`), and a `Blocked by:` line. The **frontier** — open,
unblocked, unclaimed tickets — is where work is takeable now.
