# PurrNet ↔ Dissonance Adapter Refactor Notes

Fixes for the PurrNet Dissonance integration (`BobsiUnity/PurrNet-VoiceChat`,
`Dissonance.Integrations.PurrNet`) that make voice work reliably with the PurrNet
**host** model and across **reconnects**, plus init-order hardening so the adapter
starts deterministically regardless of Unity's Awake/Start ordering.

The lifecycle fixes touch `PurrNetCommsNetwork.cs`, `PurrNetClient.cs`, and
`PurrNetServer.cs`; a small Unity 6 maintenance update also touches
`PurrNetDissonancePlayer.cs`. This document justifies every change.

The changes fall into two groups:

- **Bug fixes (§1–§5)** — things that were broken with a PurrNet host or on
  reconnect. These are the substance of the PR.
- **Init-order hardening (§6–§8)** — making *when/which role* the session starts
  explicit and race-free, and making the manual (`startFlags = None`) path a
  first-class citizen.

## Testing scope / caveats (please double-check in review)

- Tested **in the Unity Editor only** (Multiplayer Play Mode), Unity 6000.4,
  PurrNet `v1.20.0-beta.249`, Dissonance 9.0.9.
- Exercised mostly the **host + client(s)** model (1 host, 1–3 clients), all
  connect/disconnect/reconnect orders, and the **server → host promotion** case.
- **Not** tested in a standalone build, and **not** with a real headless
  dedicated server (planned server-only, `startServerAsHost` off). The code paths are meant to
  be role-agnostic, but the build and dedicated-server flows deserve explicit
  verification. `PassesStartFlags` now delegates to `NetworkManager.ShouldStart`
  (§7), so the build flags follow PurrNet's own auto-start logic rather than a
  hand-rolled branch — but that delegation is still only exercised in the editor.
- Host **migration** (server itself restarting while clients persist) is out of
  scope — that is a separate feature.

---

# Bug fixes

## 1. Host-client identity captured as the transient `Server` id

**Symptom / scenario.** With a PurrNet host, voice worked only intermittently and
was start-order dependent. The Dissonance server evicted its own host-client:
`BroadcastingClientCollection: Client '<remote>' handshake received but client is
already connected with a different connection! Disconnecting client
'<host-client>/0/Server'`. Result: room membership churned and no voice flowed
(`VoiceSender: Dropping voice packet (no listening destinations)`).

**Root cause.** The host-client identity was cached lazily on its first send:

```csharp
if (NetworkManager.main.isHost) {
    if (!_network.hasHostClientId) {
        _network.hostClientPlayerId = NetworkManager.main.localPlayer; // ← captured now
        _network.hasHostClientId = true;
    }
    ...
}
```

On a PurrNet host `NetworkManager.localPlayer` is transiently `Server` and only
later becomes a numeric id. If the first send happens in that window, the
host-client is registered under `Server`. The remote client's `ServerRpc`
`info.sender` also resolves to `Server` in that window, so the two collide on one
identity and Dissonance evicts the host-client.

**Fix.** The host-client does not send until its local player has its **final** id:
a guard `isHost && !isLocalPlayerReady` in `PurrNetClient.SendReliable/SendUnreliable`
returns early, so the id can never be cached as `Server`. (The eager capture from
`onLocalPlayerReceivedID` also exists — see §6 — but this guard is the actual
correctness barrier and stands on its own.)

## 2. Host vs dedicated-server misdetection (transient runtime role)

**Symptom / scenario.** A host sometimes started as a `DedicatedServer` (no local
client → no host voice), start-order dependent.

**Root cause.** The mode was picked from the **runtime** role at connect time
(`isHost` / `isServerOnly` / `isClientOnly`). On a host the server connects a few
frames before the client, so `isServerOnly` is transiently `true`, which made the
adapter start `RunAsDedicatedServer`. Dissonance's network mode is atomic (`Host`
vs `DedicatedServer`); once running as `DedicatedServer` you cannot add a client
without a mode transition that **restarts the server** (kicking all clients).

**Fix.** Resolve the mode from the NetworkManager's **planned** role, not the
transient runtime role — see §6/§7 (`isPlannedHost`, `startServerAsHost`). This
removes the "host briefly looks server-only" race at its source.

## 3. Reconnect → `Kicked from session - wrong session ID` (stale receive queues)

**Symptom / scenario.** Reconnecting a client produced
`PurrNetClient: Kicked from session - wrong session ID. Mine:X Theirs:Y`.

**Root cause.** The receive buffers are `static`
(`PurrNetClient._receivedData`, `PurrNetServer._receivedData`) and were **never
cleared**. On reconnect the freshly created client/server dequeued packets left
over from the previous session (carrying the old `SessionId`) and got kicked.

**Fix.** Added `ClearReceiveQueues()` to `PurrNetClient` and `PurrNetServer`,
called on every session start and stop. Each drained queue is also returned to the
pool via `QueuePool<byte[]>.Destroy(queue)` (its buffers already went back to
`ByteArrayPool`), so repeated reconnects don't leak the pooled `Queue` objects.

## 4. Reconnect broken — server never removed departed clients

**Symptom / scenario.** After a client left, the others kept a phantom peer;
reconnects only worked in a specific order.

**Root cause.** The adapter never told the Dissonance server when a PurrNet player
left. `BaseServer.ClientDisconnected(connection)` is documented as *"must be called
by the extending network integration implementation when a client disconnects from
the session"* — the adapter simply never called it, so departed clients lingered
and collided with their own reconnects.

**Fix.** Subscribe to `NetworkManager.onPlayerLeft` and, on the server side, call
`ClientDisconnected` via `PurrNetServer.NotifyPlayerDisconnected(player)`.

## 5. Host client and server shared one lifecycle

**Symptom / scenario.** Stopping **only** the host's client connection (PurrNet
allows this while the server keeps running and other clients keep playing) killed
voice for everyone; and the host-client did not rejoin voice when it came back.

**Root cause.** On any client-connection `Disconnected` the adapter called
`Stop()`, tearing down the **whole** Dissonance session including the server — so
the relay for the remaining clients died.

**Fix.**
- On a client `Disconnected` while the server is still up (`isServer`), drop
  **only** the Dissonance client (`client.Disconnect()`), not the session.
  Dissonance's `Host` session keeps the server relaying and re-establishes the
  client automatically when it returns; the departed host-client is removed for the
  others by §4.
- Full `Stop()` only happens when the **server** connection drops.

---

# Init-order hardening

## 6. Race-free server mode: intent-based, with a `startServerAsHost` override

**What.** Replaced the runtime-role branch (`isHost`/`isServerOnly`/`isClientOnly`)
with a single resolver keyed on the NetworkManager's **intent**, plus one opt-in bool:

```csharp
[SerializeField] private bool startServerAsHost;

private NetworkMode ResolveServerMode(NetworkManager nm) =>
    startServerAsHost || nm.isPlannedHost || nm.isHost
        ? NetworkMode.Host
        : NetworkMode.DedicatedServer;
```

**Why.** `isPlannedHost` (`NetworkManager`) derives the role from the configured start
flags (`ShouldStart(_startServerFlags) && ShouldStart(_startClientFlags)`) — the
**intent** — not the transient runtime state that caused §2. The single expression
covers every case:

- **auto-start host** — `isPlannedHost` is true, so it's `Host` even while the runtime
  role is transiently server-only during init (the §2 race is sidestepped, not raced);
- **auto-start dedicated** — planned server-only: both host predicates false → headless;
- **manual settled host** — planned flags may be `None`, so `isHost` (runtime, now
  settled) catches it → `Host`;
- **manual server-only** — both false → headless;
- **server → host promotion** — the *only* case that needs the explicit
  `startServerAsHost`: bring voice up as `Host` even when the instance is (or plans to
  be) server-only, so a local client can join **later** without restarting voice
  (Dissonance's mode is atomic and can't be switched at runtime like PurrNet's).

Checking a planned/actual host **first** is the key: keying off runtime `isServerOnly`
(as the stock adapter did) is exactly the §2 host-as-dedicated bug.

**Why a bool, not the earlier `Auto/Host/DedicatedServer` enum.** The enum's three
values collapse: `Auto` and `DedicatedServer` are both "follow intent" (a server-only
instance is already headless under `Auto`), and forcing dedicated on a planned/actual
host is meaningless (a host wants local voice). Only the host override carries real
information, so it becomes a single default-off checkbox — smaller surface, same
capability. Default `false` is backward-compatible: a normal host runs as `Host`, a
dedicated server runs headless. **Caveat (in the tooltip):** with PurrNet start flags
`None`, a manual voice start *before* the instance is actually a host counts as
headless — for delayed promotion, enable `startServerAsHost` beforehand.

## 7. `PassesStartFlags` defers to `NetworkManager.ShouldStart`

**What.** The whole hand-rolled gate (an `#if UNITY_EDITOR` branch checking only
`StartFlags.Editor`, plus a build branch checking `ServerBuild`/`ClientBuild` by
planned role) is replaced with a single call:

```csharp
private bool PassesStartFlags() => NetworkManager.ShouldStart(startFlags);
```

**Why.** `ShouldStart` (`NetworkManager.cs`) is PurrNet's own auto-start predicate:
it evaluates `Editor && isMainEditor`, `Clone && isClone`, `ClientBuild`/`ServerBuild`
against the actual `ApplicationContext`, and returns `false` while the global
`DisableFlags()` counter is active. The original editor branch only tested the
`Editor` flag — so it ignored `Clone` (a clone with `Clone` but not `Editor` would
never start; with `Editor` but not `Clone` it would wrongly start) and ignored
`DisableFlags()` entirely. Reusing `ShouldStart` makes the adapter's auto-start
match PurrNet's exactly, and removes the need to branch on runtime vs planned role
here at all (role is resolved in §6's `ResolveServerMode`). On the default flags
(`Editor|Clone|ServerBuild|ClientBuild`) this is behaviour-neutral; it only changes
non-standard flag combinations and honours `DisableFlags()`.

## 8. Deterministic subscription (Start + catch-up) and a real manual path

**Subscribe in `Start`, not `Awake`.** `NetworkManager.main` is assigned in NM's own
`Awake`, and Awake order between objects is undefined — subscribing in Awake could
hit a null `main` (NRE). NM runs at `[DefaultExecutionOrder(-999)]`, so by any other
component's `Start` `main` is guaranteed to exist.

**Fail fast if `main` is still null.** Because the execution order guarantees `main`
by `Start`, a null there is a genuine misconfiguration (no NetworkManager in the
scene, or one created dynamically after this component — unsupported). Rather than
silently return and leave voice deaf to every lifecycle event, `EnsureSubscribed`
throws a descriptive `InvalidOperationException`. (Calling the manual path *before
the role is settled* while the NM exists is still a supported no-op — that check is
separate, after subscription.)

**Catch up on already-reached state.** Subscribing in `Start` does **not** guarantee
we subscribe before the network comes up (NM auto-starts the network in its own
`Start`, and Start order is undefined). So `EnsureSubscribed()` — an idempotent
method holding the subscription — immediately replays the current state after
subscribing:

```csharp
if (nm.isServer)                OnServerConnectionState(ConnectionState.Connected);
else if (nm.isLocalPlayerReady) OnLocalPlayerReceivedID(nm.localPlayer);
```

The `_autoStarted` guard collapses a caught-up event and a later real event into a
single start, so double-subscription/double-start is impossible. (Subscriptions
cannot live inside `StartSession`: the auto-start path is *triggered by* a
connection event, which requires the subscription to already exist — a chicken/egg.)

**Manual path (`startFlags = None`) is now complete.** `TryRunManually()` calls
`EnsureSubscribed()` first, so after a manual start the full lifecycle (reconnect,
stop, departed-client removal) is handled automatically — previously the manual path
started a session but, with subscriptions gated behind auto-start, silently lost all
of §3–§5. Contract (documented on the method): call it once the PurrNet role is
settled; calling it before the role is known does nothing and, with `None`, there is
no auto-start to pick it up later.

**Reconnect starts bypass the start-flag gate.** `PassesStartFlags` is meant to gate only the
*first* start (build/editor flags). But a client's reconnect start also flows through `TryStart`
(`onLocalPlayerReceivedID → TryStart(Client)`), and with `startFlags = None` that gate is always
false — so after a manual first start a client would silently fail to rejoin voice on reconnect. (The
host survives via a different path: `client.Disconnect()` + Dissonance's Host retry, which never hits
the gate — which is exactly why the bug was client-only.) A `_startAllowed` flag, set on the first
`StartSession` (auto after passing the flags, or manual via `TryRunManually`) and never cleared, lets
reconnect starts skip the gate: the gate authorizes the session once, reconnects don't re-ask.

**Guards.** `EnsureSubscribed` fails fast with a descriptive exception if
`NetworkManager.main` is unexpectedly absent at subscription time;
`OnServerConnectionState` null-checks the cached manager, and `OnDestroy`
unsubscribes only when actually subscribed.

---

# Maintenance

## 9. Deprecated `FindObjectOfType` → `FindAnyObjectByType`

**What.** In `PurrNetDissonancePlayer`, the lazy `dissonanceComms` lookup used
`FindObjectOfType<DissonanceComms>()`, which Unity 6 deprecates. Replaced with
`FindAnyObjectByType<DissonanceComms>()` — same active-only semantics, and since
there is a single `DissonanceComms` per scene "any" is correct and the faster new
API (no InstanceID sort). Silences the editor deprecation warning.

---

## Changed files

| File | Change |
|---|---|
| `PurrNetCommsNetwork.cs` | `startServerAsHost` bool + single `ResolveServerMode` (`startServerAsHost \|\| isPlannedHost \|\| isHost`), intent-based and race-free, one resolver for both paths (§2, §6); `PassesStartFlags` defers to `NetworkManager.ShouldStart` (§7); subscribe in `Start` via idempotent `EnsureSubscribed` with catch-up, fail-fast on null `main` (§8); keep server alive when only the client drops (§5); clear queues + reset host-client id on start/stop (§3); remove departed clients on `onPlayerLeft` (§4); complete manual `TryRunManually` path + reconnect starts bypass the flag gate once started (§8). |
| `PurrNetClient.cs` | Gate host-client sends on `isLocalPlayerReady` (§1); add `ClearReceiveQueues()` returning queues to `QueuePool` (§3). |
| `PurrNetServer.cs` | Add `NotifyPlayerDisconnected()` → `ClientDisconnected` (§4); add `ClearReceiveQueues()` returning queues to `QueuePool` (§3). |
| `PurrNetDissonancePlayer.cs` | `FindObjectOfType` → `FindAnyObjectByType` (§9, Unity 6 deprecation). |
