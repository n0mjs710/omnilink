# CONFIG.md — Configuration, Validation, and Live Reload

Two TOML files, split along the line hblink3 already draws with
`hblink.cfg` and `rules.py` (D-09):

| file | contains | changed by |
|---|---|---|
| `omnilink.toml` | `[core]`, `[log]`, `[[system]]` — binds, ports, credentials, protocol parameters, per-system timers | **restart** |
| `rules.toml` | `[[bridge]]` definitions and their members, `[acl]` | **live reload** |

The boundary is drawn there for a practical reason: rebinding sockets and
re-authenticating peers mid-flight drops calls anyway, so runtime system
changes would buy an operator nothing a restart does not. Runtime *rules*
changes are the thing operators actually want, sometimes hourly.

## 1. Why TOML, and why a file at all

TOML because it is already the house format across `ipsc2hbpc`, `cc2obp`,
and `dmr-talkback`, its parser is already vendored and proven, and
consistency across the family is worth more than any marginal syntax
preference. YAML has no dependency-free C parser worth vendoring and
would break D-08. JSON has no comments. SQLite is a dependency and would
end git-tracked, diffable, reviewable configs — which is how these
networks are actually operated.

**The file stays the source of truth even once a web editor exists**
(§7). A database-backed editor would end diffable history, break the SSH
workflow operators actually use, and manufacture a merge problem where
none exists today.

## 2. `omnilink.toml`

```toml
[core]
stream_timeout   = 2.0        # s; core frees a silent stream (ROUTING §3)
max_streams      = 200        # fixed pool (D-23)
loop_gap         = 0.5        # s; sub-human re-capture threshold (D-19)
loop_count       = 4          # consecutive re-captures before loop_suspected
control_socket   = "/run/omnilink/control.sock"
rules_file       = "rules.toml"

[log]
file    = "/var/log/omnilink/omnilink.log"
level   = "info"              # error | warn | info | debug
console = false
```

### 2.1 Keys common to every `[[system]]`

```toml
[[system]]
name     = "KS-DMR"           # unique; the identity used in rules.toml and events
protocol = "hbp"              # hbp | ipsc | obp | cc | xlx
enabled  = true
```

Slotted systems (`hbp`, `ipsc`) additionally carry the channel timers,
because they are properties of a radio channel and belong with the system
that owns one (D-16, ROUTING §5.2):

```toml
group_hangtime = 4.0          # s; per-(system,slot) hang. 0 disables.
stream_to      = 0.36         # s; silent-stream slot release. NOT stream_timeout.
```

`stream_to` and `[core].stream_timeout` are two timers doing two jobs and
the sample file says so in a comment, because conflating them holds a
slot for two seconds after every dropped stream.

Per-system ACL keys are documented in §5.

### 2.2 Per-protocol keys

Full semantics live in ADAPTERS.md; this is the config surface.

```toml
# --- HBP master (hblink3 "SERVER") ---
[[system]]
name = "KS-DMR"
protocol = "hbp"
mode = "master"
bind = "0.0.0.0:62031"
passphrase = "s3cr3t"
max_endpoints = 50
repeat = true                 # in-adapter local repeat (D-18)

# --- HBP peer (hblink3 "OUTBOUND") ---
[[system]]
name = "UPSTREAM"
protocol = "hbp"
mode = "peer"
server = "master.example.net:62031"
passphrase = "homebrew"
radio_id = 312000
callsign = "W1ABC"
loose = false

# --- OpenBridge ---
[[system]]
name = "BM-3102"
protocol = "obp"
bind = "0.0.0.0:62035"
target = "1.2.3.4:62035"
network_id = 3129100
passphrase = "password"
both_slots = true
preserve_source_peer = false

# --- IPSC ---
[[system]]
name = "MOTO-EAST"
protocol = "ipsc"
mode = "peer"                 # peer | master
bind = "0.0.0.0:50000"
master = "10.0.0.1:50000"
radio_id = 3120001
egress_clock = true           # position-preserving playout (D-15)

# --- CC-CC conduit ---
[[system]]
name = "CC-KSDMR"
protocol = "cc"
server = "cbridge.example.net:8080"
link_id = 17

# --- XLX module ---
[[system]]
name = "XLX307-D"
protocol = "xlx"
server = "xlx307.example.net:62030"
module = "D"                  # letter A-Z only; numbers are an error
radio_id = 312000
callsign = "W1ABC"
```

**Hostnames are resolved once, at startup** (D-22). Every `server`,
`master`, and `target` becomes a stored `sockaddr` and no reconnect timer
or datapath ever calls `getaddrinfo` again.

## 3. `rules.toml`

```toml
[acl]                          # global layer (§5)
sub      = "DENY:1"
tgid_ts1 = "PERMIT:ALL"
tgid_ts2 = "PERMIT:ALL"
reg      = "PERMIT:ALL"

[[bridge]]
name = "STATEWIDE"
members = [
  { system = "KS-DMR",    slot = 1, tgid = 3120 },
  { system = "MOTO-EAST", slot = 2, tgid = 2 },
  { system = "BM-3102",   tgid = 3120 },
  { system = "XLX307-D" },
]

[[bridge]]
name = "TAC-1"
members = [
  { system = "KS-DMR", slot = 2, tgid = 8951,
    to_type = "ON", timeout = 10, on = [8951], off = [4000], reset = [] },
  { system = "BM-3102", tgid = 8951, active = false,
    to_type = "ON", timeout = 10, on = [8951], off = [4000] },
]
```

## 4. One member syntax (D-07)

There is **one** member form. The named system's protocol determines
which fields are required, optional, or an error — the special-casing
lives in the validator, where it can produce a specific message with a
remedy, rather than in the grammar, where it becomes boilerplate the
operator has to get right by rote.

| protocol | `slot` | `tgid` |
|---|---|---|
| `hbp`, `ipsc` | **required** | **required** |
| `obp` | optional, defaults to 1 | **required** — it *is* the route |
| `cc` | **error** — injected | **required** — the conduit's local TGID |
| `xlx` | **error** — injected as 2 | **error** — injected as 9 |

Injected versus exposed is about observability, not taste (D-07). XLX's
TS2/TG9 is a protocol constant whose wrong value is *silently* fatal — no
acknowledgement exists and no frame carries module identity, so
mis-addressed traffic vanishes without symptom. CC's local TGID is the
opposite: a wrong value shows up immediately as traffic under a
consistent but unexpected talkgroup, and the CC-CC specification expects
each end to configure it independently.

Optional dynamic-rule fields on any member (ROUTING §4; omit them and the
member is simply always active):

```
active   = true         # initial state
to_type  = "NONE"       # NONE | ON | OFF
timeout  = 2            # MINUTES in config; ×60 at load, as hblink3 does
on       = []           # trigger TGIDs that activate
off      = []           # trigger TGIDs that deactivate
reset    = []           # trigger TGIDs that reset the timer only
```

`timeout` is in **minutes** because that is what existing `rules.py`
files mean, and silently reinterpreting an operator's `2` as two seconds
would be the worst kind of migration bug.

## 5. ACLs (D-12, ROUTING §2.1)

Global keys live in `[acl]`; per-system keys live in the `[[system]]`
stanza and use the same names with an `_acl` suffix:

```toml
[[system]]
name = "KS-DMR"
use_acl      = true
reg_acl      = "DENY:1"
sub_acl      = "DENY:1"
tgid_ts1_acl = "PERMIT:ALL"
tgid_ts2_acl = "PERMIT:ALL"
```

Grammar, layering, enforcement order, and the OpenBridge slot-1 rider are
normative in ROUTING §2.1. Two config-level rules:

- **`use_acl = false` does not disable `reg_acl`.** Registration
  admission is always enforced on master-mode systems.
- **A malformed ACL is fatal, never ignored.** At startup the daemon
  refuses to start; on reload it refuses the swap and keeps the previous
  rules (§6).

## 6. Validation and reload

### 6.1 The validator is one implementation, used three ways

The same code path runs at startup, under `omnilink --check-config`, and
on reload. There is no second, laxer parser anywhere.

**Every rejection names the offending line and states the remedy.** A
validator that says "invalid member" has moved the operator's problem
around rather than solved it. Specific messages the docs require:

- A member naming a system that no `[[system]]` stanza defines, listing
  the names that do exist.
- `slot` or `tgid` supplied for a system whose protocol injects it,
  saying which value will be used and why supplying it is an error.
- `slot` or `tgid` missing where required.
- A `cc` or `xlx` system appearing in more than one bridge — naming both
  bridges, and saying that the connection *is* the bridge identity so
  the operator understands why (D-07).
- A `cc` or `xlx` system named as a unit-call target.
- An XLX `module` given as a number rather than a letter A–Z, with the
  letter that number would correspond to.
- A malformed ACL string, quoting the fragment that failed and restating
  that one ACL carries exactly one action.
- Duplicate system names, and duplicate bridge names.

**Duplicate `(system, slot, tgid)` across bridges is legal**, because
hblink3 permits it and existing configs contain it (D-04). It is
reported at load as an informational line — "this member feeds N
bridges" — never as an error.

### 6.2 `omnilink --check-config`

A first-class subcommand. Parses and fully validates the files on disk,
prints every finding, exits non-zero on any error. This is the piece that
makes rules editing survivable: it belongs in a pre-commit hook and in a
deploy script, so a bad rules file is caught before it reaches a live
network rather than after.

It is also what the web editor shells out to (§7), which is why there is
exactly one validator.

### 6.3 Reload

Triggered by `SIGHUP` or the `reload` command on the control socket
(D-10). The sequence is **validate, then swap**:

1. Parse `rules.toml` into a **new** table arena. Allocation here is
   legal: reload is a control-plane event, not the datapath (D-23), and
   the code says so at the allocation site.
2. Validate it completely, including cross-references into the immutable
   system table.
3. **Any error → abort.** Nothing is swapped. The daemon keeps routing on
   the existing rules and reports every finding on the control socket, in
   the log, and as a `reload_failed` event. *This is the property that
   makes reload safe to use, and it is the actual cure for what hurts
   about `rules.py` today.*
4. Success → bump the generation counter and swap one pointer.
5. Streams in flight keep their `rules_generation` and finish on the
   arena they opened under (ROUTING §3). The old arena is released when
   its last referencing stream ends.

**What reload does not touch:** system stanzas, sockets, peer sessions,
login state, or in-flight calls. A `rules.toml` that references a system
which does not exist is a validation error, not an invitation to create
one.

**Dynamic-rule state on reload.** A member that survives a reload
unchanged — same `(system, slot, tgid)` in the same bridge — keeps its
runtime `active` state and its timer. A member that is new, moved, or
whose trigger configuration changed starts fresh from its configured
`active`. Anything else either resets live bridges on every unrelated
edit or silently carries stale state into a rule the operator just
rewrote.

### 6.4 Startup

Failure at startup is fatal and loud. A daemon that starts with half a
config is worse than one that does not start, because the operator
discovers the difference from an on-air complaint rather than from
`systemctl status`.

## 7. The web rules editor (deferred, PLAN phase 7)

Decoupled on purpose so it blocks nothing and can be built whenever it is
wanted:

- It reads and writes the same `rules.toml`. No database, no second
  source of truth.
- It validates by shelling `omnilink --check-config`, so the editor and
  the daemon can never disagree about what is valid.
- It applies changes by sending `reload` on the control socket, and
  surfaces the daemon's findings verbatim if the reload is refused.
- Comments and formatting in a hand-edited file are preserved where
  practical; an editor that reflows an operator's annotated rules file on
  every save will not be used twice.

Until it exists, an editor plus `--check-config` covers the great
majority of the pain, because the pain was never really the syntax.

## 8. Migration from `rules.py` (PLAN phase 3)

`rules.py` is importable Python, so the converter imports an operator's
existing file and emits `rules.toml`:

- `BRIDGES` members map field-for-field: `SYSTEM`/`TS`/`TGID` →
  `system`/`slot`/`tgid`, and `ACTIVE`/`TO_TYPE`/`TIMEOUT`/`ON`/`OFF`/
  `RESET` keep both their names and their units.
- `OBP_BRIDGES` rows become ordinary members with the mapped TGID, and
  the `(TGID, TS)` override form becomes an explicit `slot`. hblink3
  already expands these into synthetic bridge members internally, so this
  is a change of notation, not of meaning.
- `XLX_BRIDGES` rows become bare `{ system = "..." }` members.
- ACLs and system stanzas come across from `hblink.cfg` in the same pass.

**It reports what it cannot represent rather than guessing**, and its
output is run through `--check-config` before it is offered to the
operator. A converter that silently drops a rule is worse than no
converter.
