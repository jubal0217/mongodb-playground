# HANDOFF — Simulated MongoDB Ops Playground (open source)

- **Date:** 2026-10-02
- **From:** claude.ai chat (learning session: Ops Manager docs → interactive labs → playground prototype)
- **To:** a fresh Claude Code session
- **Next session goal:** scaffold the open-source repo, port the simulator prototype into a tested engine, and stand up the web app shell.

---

## 1. What we're building

An **open-source web app** for people learning to run MongoDB:

1. **Register / log in**, track progress.
2. **Modules**: interactive lessons (diagrams, sliders, quizzes) that always link to the **official MongoDB docs**.
3. **Playground**: a **simulated** lab. No real VMs.
   - Grey VM cards (e.g. 3 Rocky Linux 9 nodes) + a **fake terminal** below.
   - `ssh root@<ip>` → that VM card **lights up and pulses** (your session). `exit` → back to the laptop; SSH into the next one.
   - Commands mutate a fake VM state (files, packages, firewall, services). Realistic success and error output.
   - Goal checklist + "next step" hints + official doc links.
   - Scenarios: standalone, replica set, sharding, Ops Manager install/config.

Target learner: someone like the author — real sysadmin tasks on Rocky Linux, wants concrete, hands-on practice without paying for VMs.

---

## 2. Before you start — save these from the chat

These exist only as artifacts in the originating chat. The user must save them into the repo first.

| Chat artifact | Save to | What it is |
|---|---|---|
| Replica Set Playground (prototype) | `prototype/replica-set-playground.html` | **Reference implementation** of the simulator (vanilla JS, single file). Port its logic; don't redesign blind. |
| Ops Manager Install Lab (Part 1) | `prototype/labs/part1-install.html` | Install, sizing, ports, pre-flight, download modes, roles |
| Ops Manager Lab · Part 2 | `prototype/labs/part2-backing-dbs.html` | Backing DBs, failure sim, `mongo.mongoUri` builder, TLS, credentialstool, AppDB monitoring, config linter |
| Ops Manager Lab · Part 3 (Plan it) | `prototype/labs/part3-plan.html` | Install checklist, topology sim, decision→plan, host planner, versions/OS |
| Ops Manager Lab · Part 4 (Backup) | `prototype/labs/part4-backup.html` | Backup flow, job states, snapshot stores, oplog store, load factor |
| Ops Manager Lab · Part 5 (MongoDB Agent) | `prototype/labs/part5-agent.html` | Agent loop, prereqs, functions sim, agent DB user, config file, automation config, compatibility |

The labs are **content prototypes** for future modules. Each one already shows the content tone and interaction style the user liked.

---

## 3. Decisions already made

| Decision | Why |
|---|---|
| **Simulate, don't run real VMs** | No per-user servers to pay for or secure. No abuse risk. Scales. Avoids hosting Ops Manager (see §4). |
| **Stack (recommended): static frontend + Supabase** (auth + Postgres for progress) | No server to run. Alternative considered: Next.js full-stack + own backend (more control, more work). Final call still open — see §10. |
| **Terminal UI:** start simple, move to **xterm.js** | Prototype uses a plain input line; xterm.js gives a real terminal feel later. |
| **Engine = pure functions over a state object** | Prototype proved it. Makes TDD easy, UI-independent. |
| **Lab target:** Rocky Linux 9 + MongoDB 8.0 Community | Matches the author's real environment. (9.0 exists; version picker is an open question.) |
| **Labs are explicitly insecure** (no auth, `bindIp 0.0.0.0`) | Simplicity for the first module. Every such screen shows a "lab only" warning. Securing (keyFile + users) is a planned module. |

---

## 4. Constraints (non-negotiable)

1. **Never ship MongoDB Enterprise / Ops Manager binaries.** The free license is evaluation/development in the licensee's own internal environment only. The simulator never runs them — keep it that way.
2. **MongoDB docs are CC BY-NC-SA 3.0 US.** Lessons must be written in our own words + link to official pages. If any docs text is adapted, credit MongoDB and license that content CC BY-NC-SA (non-commercial). Code can be MIT.
3. **Unofficial project.** README and footer: "Unofficial. Not affiliated with MongoDB, Inc." Don't brand it to look like a MongoDB product.
4. **No secrets in the repo.** `.gitignore`: `.env*`, `*.pem`, `gen.key`, `*.keytab`, Supabase service keys.
5. **Commands shown to learners must be real and exact.** If the sim accepts a simplified form, mark it clearly.
6. **Supabase RLS on every table**: users read/write only their own rows.

---

## 5. Proposed architecture

```
<repo>/                         # name TBD (avoid "MongoDB" as the brand)
  apps/web/                     # Vite + React + TS (or Next.js if chosen)
  packages/sim-core/            # pure TS engine — NO DOM
    src/world.ts                # types: World, NodeState, Session, Cluster
    src/exec.ts                 # exec(world, line) -> { world, output }
    src/commands/               # ssh, hostnamectl, dnf, sed, systemctl, firewall-cmd, mongosh...
    src/checks.ts               # goals(world), nodeReady(node)
    src/hints.ts                # nextHint(world)
    test/                       # golden path + failure cases (§7)
  packages/scenarios/           # scenario definitions (nodes, goals, hint rules)
  content/modules/              # lessons (MDX), own words + official links
  supabase/migrations/          # profiles, progress, playground_saves
  prototype/                    # saved chat artifacts (reference only)
  docs/adr/  docs/content-style.md  docs/licensing.md
  LICENSE (MIT, code)  LICENSE-content (CC BY-NC-SA 3.0)  README.md  CONTRIBUTING.md
```

Engine interface sketch:

```ts
type Line = { text: string; kind: 'out' | 'err' | 'dim' | 'ok' | 'warn' | 'prompt' };
interface Scenario { id: string; title: string; nodes: NodeSpec[]; goals: GoalDef[] }
interface World { nodes: NodeState[]; session: { node: number | null; mode: 'sh' | 'mongo'; heredoc?: Heredoc }; cluster: ClusterState | null }
exec(world: World, line: string): { world: World; output: Line[] }   // pure
goals(world: World): { id: string; label: string; done: boolean }[]
nextHint(world: World): { cmd: string; why: string; docUrl: string } | null
prompt(world: World): string
```

Supabase tables (minimal): `profiles(id, display_name)`, `progress(user_id, module_id, step, completed_at)`, `playground_saves(user_id, scenario_id, world_json, updated_at)`.

---

## 6. How the prototype simulator works (port this)

**State per node:** `hostname`, `files` map (`/etc/hosts`, `/etc/mongod.conf`, `/etc/yum.repos.d/*.repo`, `/etc/os-release`), `installed`, firewall `fwPerm` / `fw` (runtime) sets, `running`, `enabled`, `failed`, `lastErr`, `live` (config parsed **at start**), `startedConf`, `role`, `touched`, `known` (ssh known_hosts).
**Global:** `session` (laptop or node index), `mode` (`sh` / `mongo`), `heredoc`, `rs` (`{ set, members[{host,node}] }`), `verified`, history.

**Supported commands**
- Laptop: `ssh root@<ip>` (by IP only — laptop has no lab DNS), `nodes`, `hint`, `help`, `clear`.
- Node shell: `exit/logout`, `whoami`, `pwd`, `hostname`, `hostnamectl [set-hostname X]`, `cat FILE`, heredocs (`cat >|>> FILE <<EOF`, `cat <<EOF >|>> FILE`, `tee [-a] FILE <<EOF`), `echo [-e]|printf "..." [>|>>|| tee [-a]] FILE`, `ls DIR`, `grep [-E|-i] PAT FILE`, `sed -i 's/A/B/[gI]' FILE` (JS regex, multiline, `\n` and `&` in replacement), `dnf|yum install [-y] PKG`, `systemctl start|stop|restart|enable [--now]|disable|status|is-active|is-enabled mongod`, `journalctl -xeu mongod`, `firewall-cmd [--permanent] --add-port|--remove-port`, `--reload`, `--list-ports`, `--state`, `getent hosts`, `ping`, `mongosh`. `vi/vim/nano` → message suggesting sed/printf/heredoc.
- mongosh: `rs.initiate([cfg])` (one line), `rs.add("host:port")`, `rs.status()`, `rs.conf()`, `db.hello()`, `exit`.

**Key semantics**
- `mongod.conf` parsed as mini-YAML: top-level key allowlist; `replSetName` must be indented under uncommented `replication:` → else start fails `Unrecognized option: …`. Trailing `# comments` stripped. Reads `net.bindIp`, `net.bindIpAll`.
- Config is read **only at start**. Editing after start = "stale" until restart.
- `dnf install mongodb-org` needs a repo file whose `baseurl` points at `repo.mongodb.org/yum/redhat/`; else `Unable to find a match`.
- Reachability A→B: B running; B's bindIp allows remote (`0.0.0.0`, `::`, own IP, `bindIpAll`) else **Connection refused**; B's runtime firewall has `27017/tcp` else **Connection timed out**. Local mongosh needs bindIp to include `127.0.0.1`/`0.0.0.0` else `ECONNREFUSED`.
- Name resolution: per-node `/etc/hosts` + own hostname resolves (like nss-myhostname).
- `rs.initiate` checks, in order: started with replSet → set name matches → a member maps to self → quorum (reachable + same set name) → lab-only all-to-all name resolution.
- Elections: majority of members alive → keep or pick a primary; else all alive are SECONDARY (no primary).
- firewalld: permanent vs runtime; `--reload` copies permanent → runtime.

**Known simplifications / gaps:** one SSH session at a time (no nested ssh), mongosh input must be one line, no auth/keyFile, not a real YAML parser, no Tab completion, no persistence, sed uses JS regex, heredocs don't expand `$vars`, election picks the first alive member.

---

## 7. Acceptance tests — port these first (TDD)

**Golden path** (must reach 10/10 goals). For node1 (repeat for node2/node3 with their IP + name):

```
ssh root@10.0.0.11
hostnamectl set-hostname node1.lab.local
printf "10.0.0.11 node1.lab.local node1\n10.0.0.12 node2.lab.local node2\n10.0.0.13 node3.lab.local node3\n" >> /etc/hosts
printf "[mongodb-org-8.0]\nname=MongoDB Repository\nbaseurl=https://repo.mongodb.org/yum/redhat/9/mongodb-org/8.0/x86_64/\ngpgcheck=1\nenabled=1\ngpgkey=https://pgp.mongodb.com/server-8.0.asc\n" > /etc/yum.repos.d/mongodb-org-8.0.repo
dnf install -y mongodb-org
sed -i 's/bindIp: 127.0.0.1/bindIp: 0.0.0.0/' /etc/mongod.conf
sed -i 's/^#replication:/replication:\n  replSetName: rs0/' /etc/mongod.conf
firewall-cmd --permanent --add-port=27017/tcp
firewall-cmd --reload
systemctl enable --now mongod
exit
```
Then on node1: `mongosh` → `rs.initiate({ _id: 'rs0', members: [ { _id: 0, host: 'node1.lab.local:27017' }, { _id: 1, host: 'node2.lab.local:27017' }, { _id: 2, host: 'node3.lab.local:27017' } ] })` → `{ ok: 1 }` → `rs.status()` shows 1 PRIMARY + 2 SECONDARY.

**Failure cases (each must produce the realistic error):**

| Setup | Expected |
|---|---|
| Skip firewall on node2 | initiate fails: `… node2.lab.local:27017 failed with … Connection timed out` |
| Skip bindIp on node2 | `… Connection refused` |
| `replSetName` at column 0 | `systemctl start` fails; `journalctl` shows `Unrecognized option: replSetName` |
| Edit conf after start, no restart | old config stays live; hint says restart |
| bindIp = node IP only | local `mongosh` → `ECONNREFUSED 127.0.0.1:27017` |
| `dnf install` before repo | `Unable to find a match: mongodb-org` |
| `ssh root@node1.lab.local` from laptop | `Could not resolve hostname` |
| Different set name in `rs.initiate` | `Rejecting initiate with a set name that differs…` |
| `rs.add` on a secondary | `MongoServerError: not primary` |
| `systemctl stop mongod` on PRIMARY | election → another member PRIMARY |
| Stop 2 of 3 | no PRIMARY |

---

## 8. Roadmap

- **M0 — scaffold:** repo layout (§5), licenses, README (unofficial notice + licensing), CONTRIBUTING, `.gitignore`, CI (lint + tests), pre-commit.
- **M1 — engine:** port prototype into `packages/sim-core` behind the §5 interface; §7 tests green.
- **M2 — web shell:** module list → lesson view → playground view (node cards w/ pulse, terminal, goals, hints, doc links). Mobile-friendly.
- **M3 — accounts:** Supabase auth, progress, save/restore playground state. RLS.
- **M4 — scenarios:** standalone; replica set + keyFile/auth; sharding (config server RS, shards, mongos, `sh.addShard`); Ops Manager install sim (from Parts 1–5).
- **M5 — content:** port the 5 lab HTML files into modules; quizzes.

---

## 9. Content notes — doc facts & inconsistencies found (as read 2026-10-02)

Useful for accurate lessons. Each was seen on the official Ops Manager docs (current = 9.0):

- Ops Manager **app** OS table names RHEL, not Rocky; the **Agent** table names RHEL/Rocky/Alma/Oracle 8/9/10.
- DEB install: download name `mongodb-mms-<version>.x86_64.deb` vs install command `mongodb-mms_<version>_x86_64.deb`.
- Requirements page lists LDAP/LDAPS ports as UDP.
- Sizing example (OM app + Backup Daemon) totals 15 GB RAM / 100 GB disk, omitting the app's 10 GB `/opt` + logs.
- Backing-DB version table differs between the requirements page (has 9.0) and the backing-DB install page (older).
- `mongo.mongoUri` SRV examples include a port and sometimes miss `//`.
- Auth page lists LDAP under MongoDB Community; LDAP deprecated starting MongoDB 8.0.
- PEM key password: one page says `credentialstool`, another says `encryptiontool`.
- OIDC settings named `mms.oidc…`, examples show `mongo.oidc…`.
- Checklist says Ops Manager uses `w:2`; backing-DB page says 8.0.26+ uses `majority` for durable writes.
- Backup overview says blockstore/S3 metadata DB lives "on the Ops Manager host"; S3 page says separate hosts.
- Agent prereqs: `dig +short myip.opendns.com @resolver1.opendns.com` shows the public IP, not FQDN resolution.
- Agent settings: `logLevel=ROUTINE` example isn't a listed level.
- Required Access page says "non-Automation only" but lists Automation roles.
- Automation config: FCV accepted values listed as 5.0/6.0/7.0 (looks outdated); agent compat table has no 9.0 row.
- Dangerous automation-config fields: `processes[n].lastResync` (wipes dbPath), `replicaSets[n].force.currentVersion: -1` (can roll back majority-committed writes).

---

## 10. Content style guide (from the author's preferences)

- Plain words, short sentences, short paragraphs. Explain a jargon word in the same breath.
- Paths, commands, names, errors, versions: **character-for-character**.
- Every lesson step links its **official doc page**.
- Mark anything not from MongoDB's docs as **EXTRA**.
- Say risky things plainly ("Terminate deletes all retained backups").
- Interactive > static: sims, calculators, click-to-reveal, quizzes.
- For decisions: two options, context to pick fast, and a recommendation.
- The author may want Traditional Chinese (zh-TW) content later.

---

## 11. Open questions (ask the user)

1. Project name / branding.
2. Ever charge money? (Decides whether docs-adapted content is allowed — CC BY-NC.)
3. Supabase + static vs Next.js full-stack — final call.
4. Scenario authoring format: declarative JSON vs TS code.
5. MongoDB version picker (8.0 / 9.0) per scenario?
6. Terminal accessibility (screen readers) and mobile input.
7. i18n (zh-TW) from day one or later?
8. Should the sim ever accept "approximately right" commands, or always demand exact real syntax?

---

## 12. Suggested skills (invoke if available in Claude Code)

- **project-governance** — at repo start: constitution, falsifiable spec, contracts, contract tests.
- **mattpocock-skills:grill-me** / **grill-with-docs** — stress-test this plan and resolve §11 before coding.
- **mattpocock-skills:to-spec** → **mattpocock-skills:to-tickets** — turn this handoff into a spec and tracer-bullet tickets.
- **mattpocock-skills:wayfinder** — the full app is bigger than one session; map decisions as tickets.
- **mattpocock-skills:tdd** — build `sim-core` test-first from §7.
- **mattpocock-skills:design-an-interface** / **codebase-design** — shape the engine API (§5) as a deep module.
- **mattpocock-skills:ubiquitous-language** — lock terms: node, session, world, scenario, goal, hint, live config.
- **system-architect** — review the architecture and rollout.
- **frontend-design** — playground UI (node cards, pulse, terminal).
- **mattpocock-skills:setup-pre-commit** — lint/format/typecheck/test hooks.

---

## Working agreements (from the author's CLAUDE.md)

Think before coding and surface assumptions · simplest thing that works · surgical changes only · define success as tests, loop until green.
