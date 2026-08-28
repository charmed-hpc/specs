---
index: UHPC017
title: Charm and solution promotion criteria for Charmed HPC
---

# Charm and solution promotion criteria for Charmed HPC

## Abstract

This specification defines what testing and release hygiene must be satisfied to
promote Charmed HPC artifacts along the Charmhub risk ladder
`edge -> beta -> candidate -> stable`. It covers two levels:

- **Single charm**: the criteria that govern promotion of an individual Ubuntu Juju
  charm, using the `canonical/slurm-charms` monorepo (`slurmctld`, `slurmd`,
  `slurmdbd`, `sackd`, `slurmrestd`) as the worked example.
- **Charmed HPC solution (multi-charm)**: the cross-charm criteria that only make
  sense when the constituent charms are deployed together. The solution criteria
  build on the single-charm criteria: a solution promotion assumes each constituent
  charm has independently satisfied the single-charm gates for the same target
  channel, and adds the cross-charm gates on top.

## Rationale

Charmed HPC is composed of multiple cooperating charms that are released and promoted
independently, yet consumed together as a solution. Without a well-defined promotion
process at both levels, it is difficult to know what evidence justifies moving a
specific artifact from one risk channel to the next, or to reason about the maturity
of the solution as a whole.

A shared, reusable set of promotion criteria lets each component maintainer promote
their charm against consistent gates, and lets the solution owner reason about the
combined stack using the **weakest-link rule** (the solution is only as mature as its
least-mature required constituent). This keeps promotions deliberate, auditable, and
grounded in evidence rather than ad-hoc judgement.

The model is grounded in:

- The Charmhub [track + risk channel](https://juju.is/docs/juju/channel) release model.
- The [Charmed HPC contributing guide](https://github.com/canonical/hpc-team/blob/main/CONTRIBUTING.md).
- Common industry progressive-delivery practice (alpha/beta/RC/GA with bake time,
  soak testing, and staged sign-off).

## Specification Part A: Single-charm promotion criteria

### A.1 Purpose and scope

This part defines what testing and release hygiene must be satisfied to promote
a single Ubuntu Juju charm along the Charmhub risk ladder:

```
edge  ->  beta  ->  candidate  ->  stable
```

It is written to be reusable for any single charm, and uses the `canonical/slurm-charms`
monorepo (`slurmctld`, `slurmd`, `slurmdbd`, `sackd`, `slurmrestd`) as the worked
example. Part B layers additional cross-charm gates on top of this one for the full
Charmed HPC solution.

#### A.1.1 What a "promotion" is

A promotion moves a **specific charm revision** from one risk channel to the next
*within the same track* (for example, `latest/edge` -> `latest/beta`, or
`24.11/beta` -> `24.11/candidate`). Promotion never rebuilds the artifact: the same
revision that passed the lower channel is what gets released to the higher one. This
is the standard "build once, promote the artifact" principle.

### A.2 Channel and versioning model

#### A.2.1 Risk channels (the promotion ladder)

| Channel     | Audience                         | Intent                                                      | Stability contract |
|-------------|----------------------------------|------------------------------------------------------------|--------------------|
| `edge`      | Charm developers, CI             | Every merge to the tracked branch; may break at any time    | None |
| `beta`      | Early adopters, integration labs | Feature-complete for the increment; known-good on target base | Deploys and passes full functional tests |
| `candidate` | Release validators, operators evaluating an upgrade | Release candidate after some soak; no known critical defects    | Production-shaped; upgrade and rollback verified |
| `stable`    | Production users                 | Supported, documented, scale-tested, backwards-compatible release         | Full support contract; safe sequential upgrades |

#### A.2.2 Tracks

A Charmhub **track** groups a compatible line of the charm. For Slurm charms, tracks
follow the upstream Slurm major/minor line and the supported Ubuntu base, e.g. a
`YY.MM` track (`track/25.11` in git) plus `latest` for the current development line.

Guidance:

- **One track per supported Slurm major line** that receives independent promotion.
- A track pins a single Ubuntu base (currently `ubuntu@26.04`, `amd64`). Adding a base
  or architecture is a new platform matrix entry that must re-clear the gates below.
- `latest` maps to the `main` git branch; `YY.MM` maps to `track/YY.MM`.

#### A.2.3 Current repository reality (baseline)

The gates below describe the **target** promotion process. Where the repository does
not yet implement a gate, it is marked *(target)*. Today, `slurm-charms` CI:

- Publishes only to `*/edge` automatically (`main` -> `latest/edge`,
  `track/*` -> `<track>/edge`) via `canonical/charming-actions`.
- Has no automated candidate/stable promotion (Charmhub promotion is manual).
- Runs unit tests (`ops.testing`/Scenario) and `jubilant` integration tests, with the
  HA suite gated behind `--run-high-availability`.
- Has **no BDD/Gherkin suite yet**; functional coverage is imperative jubilant tests.
  BDD (`pytest-jubilant-bdd`) is introduced as a *target* gate at candidate and above.

### A.3 Testing dimensions

Each dimension is a class of evidence that a promotion gate can require.

| Dimension            | Question it answers                                              | Tooling in this repo |
|----------------------|-----------------------------------------------------------------|----------------------|
| **Unit**             | Does the charm logic behave correctly in isolation              | `pytest` + `ops.testing` (Scenario/Context) via `just unit` |
| **Integration**      | Does the charm deploy, relate, and run against a live Juju/LXD  | `jubilant` via `just integration` |
| **Functional / BDD** | Does the charm satisfy user-facing behaviors end to end         | `jubilant` today; `pytest-jubilant-bdd` Gherkin scenarios *(target)* |
| **Upgrade / refresh**| Can users move between revisions without data/config loss       | `juju refresh` across revisions in a `jubilant` test *(target for full matrix)* |
| **Multi-charm**      | Does it cooperate with the charms it integrates with            | See Part B |
| **Non-functional**   | Scale, soak, performance, security, HA/resilience               | `--run-high-availability`; `charmed-hpc-benchmarks`; CVE/dependency scans |

#### A.3.1 Definitions

- **Unit tests** exercise charm handlers, config, actions, and libraries with a mocked
  Juju model. Every charm **action** must have unit coverage.
- **Integration tests** deploy the packed charm to a real controller, form its
  relations, and assert active/idle status plus basic workload function (a "smoke"
  deploy at minimum).
- **Functional / BDD tests** express behaviors as `Given/When/Then` scenarios that a
  non-developer stakeholder can read (e.g. "Given a running cluster, When a user submits
  a batch job, Then it completes and returns output").
- **Upgrade / refresh tests** deploy revision N-1 (or previous stable), `juju refresh`
  to the candidate revision, and assert data/config preservation and healthy status.
  Rollback tests refresh back down and assert recovery.
- **Non-functional tests** cover: horizontal/vertical scale, multi-hour soak, a
  performance baseline, HA failover, and a security/CVE scan of the artifact and deps.

### A.4 Per-channel promotion gates

Legend: **[CI]** enforced automatically in CI; **[MAN]** manual sign-off; **[CI/MAN]**
CI-produced evidence that a human reviews. *(target)* = not yet implemented in-repo.

#### A.4.1 Promote to `edge`

Trigger: merge to the tracked branch (`main` or `track/*`). Fully automated.

- [CI] Static checks pass: `just check` (format, lint, typecheck).
- [CI] Unit tests pass: `just unit`.
- [CI] Unit coverage >= **80%** line coverage (proposed default; enforced via the
  coverage report / TICS) *(target: fail the build below threshold)*.
- [CI] Charm packs successfully for every supported base (`just repo stage <charm>`
  + `charmcraft pack`).
- [CI] Smoke integration: deploy + reach `active/idle` on at least one base
  (`just integration` subset).
- [CI] Conventional-commit and `commitlint` checks pass.
- [CI] Auto-published to `<track>/edge` (`canonical/charming-actions/upload-charm`).

#### A.4.2 Promote to `beta`

Intent: feature-complete for this increment and known-good on the target base.

Requires everything in `edge`, plus:

- [CI] **Full** integration suite green on every supported base: `just integration`.
- [CI] Every charm **action** is exercised by an integration or unit test.
- [CI] Every `provides`/`requires` relation is exercised by an integration test
  (relation data flows and both sides settle).
- [CI] **HA / resilience** suite passes: `just integration -- --run-high-availability`
  (leader loss and `slurmctld` failover recover to healthy).
- [CI] **Functional / BDD** acceptance scenarios pass *(target: `pytest-jubilant-bdd`
  Gherkin features covering the charm's primary user journeys)*.
- [CI] Basic upgrade test passes: `juju refresh` from the current `beta` (or `edge`
  N-1) revision to the candidate revision reaches healthy status *(target)*.
- [CI/MAN] Coverage has not regressed versus the current `beta` revision.
- [CI] **Automated security check**: CVE/dependency scan of the artifact and its
  Python/OS dependencies with **no unresolved critical/high** findings.
- [MAN] Draft documentation exists for any new config/action/relation (a paired PR in
  `charmed-hpc-docs`, per the contributing guide).
- [CI] **Nightly run** passes, including the scheduled HA job *(no fixed bake time)*.
- [MAN] No open **critical** or **high** severity bugs against the revision.

#### A.4.3 Promote to `candidate`

Intent: a release candidate under soak; production-shaped and upgrade-safe.

Requires everything in `beta`, plus:

- [CI] **Upgrade matrix** passes *(target)*:
  - refresh from **previous stable** -> candidate,
  - refresh from **previous beta/candidate** -> candidate,
  - **rollback** candidate -> previous revision recovers cleanly.
- [CI] Non-functional baseline captured *(target)*:
  - scale up **and** down of applicable applications (units and resources),
  - soak >= **24h** (single charm) with no leaks/restarts/status flaps,
  - performance baseline recorded (regression budget agreed by maintainers).
- [MAN] Documentation complete: usage, configuration, actions, relations, limitations,
  and any deviation from the non-charmed workload.
- [MAN] Release notes / CHANGELOG entry drafted for the revision.
- [MAN] Backwards-compatibility / interface-stability review: no breaking change to a
  published relation interface without a version/track bump.
- [MAN] Manual QA sign-off on a clean deployment following the documented quickstart.

#### A.4.4 Promote to `stable`

Intent: supported, documented, backwards-compatible general availability.

Requires everything in `candidate`, plus:

- [MAN] **Soak on `candidate`:** >= **14 days** with **zero regressions** and no new
  critical/high defects (proposed default).
- [CI/MAN] Artifact is reproducible from a tagged, signed commit; the exact revision
  promoted is the one that soaked on `candidate`.
- [CI] Upgrade from the **current `stable`** revision to the new revision verified
  (data + config preserved) *(target)*.
- [MAN] Security sign-off (no unresolved critical/high; SECURITY.md contact current).
- [MAN] Documentation published (not just drafted) and release notes finalized.
- [MAN] Approval recorded by the charm maintainer **and** the designated release owner.
- [MAN] Rollback plan documented (which revision to refresh back to, and how).

### A.5 Automation vs. manual sign-off

Recommended posture: automate everything that is deterministic and cheap; reserve
humans for judgement, soak, and scale.

| Gate class                         | edge | beta | candidate | stable |
|------------------------------------|:----:|:----:|:---------:|:------:|
| Static checks / unit / smoke       | CI   | CI   | CI        | CI     |
| Full integration + relations       | -    | CI   | CI        | CI     |
| HA / functional / BDD              | -    | CI   | CI        | CI     |
| Upgrade / rollback matrix          | -    | CI*  | CI        | CI     |
| Soak / scale / performance         | -    | -    | CI+MAN    | MAN    |
| Security / CVE scan                | -    | CI   | CI        | CI+MAN |
| Docs / release notes               | -    | MAN  | MAN       | MAN    |
| Backwards-compat / interface review| -    | -    | MAN       | MAN    |
| Final promotion approval           | auto | MAN  | MAN       | MAN    |

`*` basic single-step upgrade at beta; full matrix at candidate.

Promotion to `beta` and above should be a deliberate action (a maintainer running the
promotion, or an approved workflow_dispatch), never an automatic side effect of a merge.

### A.6 Release-hygiene gates (minimal)

Beyond tests, each promotion at `candidate` and `stable` must confirm:

1. **Documentation**: user-facing changes have a merged/queued `charmed-hpc-docs` PR,
   or a justification for why none is needed (per the contributing guide).
2. **Release notes / CHANGELOG**: a human-readable summary of what changed and any
   operator-visible impact.
3. **Security / dependencies**: CVE and dependency scan clean of critical/high;
   `SECURITY.md` reporting path valid.
4. **Backwards compatibility**: no breaking change to a published relation interface,
   config key, or action without a track/major bump and a migration note.
5. **Provenance**: promoted revision is traceable to a tagged commit and the CI run
   that produced it.

### A.7 Proposed default thresholds

These are starting points. Adjust per charm risk profile and record the chosen values.

| Parameter                              | Proposed default |
|----------------------------------------|------------------|
| Unit line coverage floor               | 80% |
| Bake time on `edge` before `beta`      | None; nightly run + automated security checks must pass |
| Bake time on `beta` before `candidate` | None |
| Soak on `candidate` before `stable`    | 14 days, zero regressions |
| Single-charm soak duration             | >= 24h |
| Min distinct revisions before `stable` | >= 1 that fully cleared `candidate` |
| Blocking security severities           | critical, high |
| Blocking bug severities for promotion  | critical, high |
| Upgrade matrix span                    | previous stable, previous beta/candidate, rollback |

### A.8 Worked example (slurm-charms commands)

Mapping the gates onto the repository's actual toolchain (`justfile` + `repository.py`):

```bash
# Static checks (edge gate)
just check                       # fmt + lint + typecheck

# Unit tests + coverage (edge gate)
just unit                        # runs repository.py unit -> cover/coverage.xml

# Integration smoke / full (edge / beta gate)
just integration -- --charm-base=ubuntu@26.04

# HA / resilience (beta gate)
just integration -- --charm-base=ubuntu@26.04 --run-high-availability

# Stage + pack a single charm (build reproducibility)
just repo stage slurmctld --clean

# Nightly coverage / quality scan (TICS) already runs weekly in CI
```

Channel mapping already implemented in CI:

- Merge to `main`      -> auto-release to `latest/edge`.
- Merge to `track/*`   -> auto-release to `<track>/edge`.
- `edge -> beta -> candidate -> stable` promotions are performed against the gates
  above (currently manual on Charmhub; a guarded promotion workflow is the *target*).

### A.9 Promotion checklist (copy per promotion)

```
Charm: __________________   Revision: ______   From: ______  ->  To: ______  Track: ______

[ ] Static checks pass (just check)
[ ] Unit tests pass; coverage >= threshold (just unit)
[ ] Integration suite green on all supported bases (just integration)
[ ] All actions and relations exercised
[ ] Upgrade test(s) pass for target channel
[ ] HA / functional / BDD pass (beta+)
[ ] Non-functional: scale + soak + perf baseline (candidate+)
[ ] Security / CVE scan clean of critical/high (beta+)
[ ] Docs updated / published; release notes drafted/finalized
[ ] Backwards-compat / interface-stability reviewed
[ ] Bake/soak time satisfied for target channel
[ ] Approvals: maintainer ______  release owner ______  (candidate+)
```

## Specification Part B: Charmed HPC solution promotion criteria (multi-charm)

This part defines what must be true to promote the **Charmed HPC solution** (a set
of cooperating charms) along the risk ladder `edge -> beta -> candidate -> stable`.

It **builds on** Part A, which governs each individual charm. A solution promotion
assumes each constituent charm has independently satisfied the single-charm gates for
the same target channel; this part adds the **cross-charm** gates that only make sense
when the charms are deployed together.

### B.1 Solution scope

#### B.1.1 Constituent charms

| Component            | Repository                       | Role in the solution |
|----------------------|----------------------------------|----------------------|
| Slurm charms         | `canonical/slurm-charms`         | Workload manager: `slurmctld` (hub), `slurmd`, `slurmdbd`, `sackd`, `slurmrestd` |
| Filesystem charms    | `canonical/filesystem-charms`    | Shared filesystem provider/consumer (NFS, CephFS) mounted on cluster nodes |
| SSSD                 | `canonical/sssd-operator`        | Directory-backed authentication / identity across cluster nodes |
| Apptainer            | `canonical/apptainer-operator`   | OCI/container runtime integration for compute nodes |

Supporting (not deployed as part of the solution, but used to validate it):

| Component               | Repository                          | Use |
|-------------------------|-------------------------------------|-----|
| Benchmarks / validation | `canonical/charmed-hpc-benchmarks`  | Non-functional and end-to-end cluster validation |
| BDD step library        | `canonical/pytest-jubilant-bdd`     | Reusable Gherkin step handlers for functional gates |
| Documentation           | `canonical/charmed-hpc-docs`        | Solution documentation and release notes |

#### B.1.2 Relation / integration map

```
        sackd ──sackd──┐
     slurmrestd ─slurmrestd─┐
                            ▼
   slurmd ──slurmd──►  slurmctld  ◄──slurmdbd── slurmdbd ──database──► mysql
                            │
                            ├── cos-agent ──► COS (observability)
                            └── oci-runtime ─► apptainer

   filesystem (server) ──mount_info──► filesystem-client ──mount──► compute/login nodes
   sssd ──► identity/auth on login + compute nodes
```

Key cross-charm surfaces to validate:

- `slurmctld` <-> `slurmd`/`slurmdbd`/`sackd`/`slurmrestd` (core Slurm relations).
- `slurmdbd` <-> `mysql` (accounting database).
- filesystem mount flowing through to compute/login nodes.
- SSSD-provided identity usable by Slurm job submission.
- Apptainer as the OCI runtime for containerized jobs.
- `slurmctld` -> COS for integrated observability.

### B.2 Solution channel model

#### B.2.1 The weakest-link rule

**A solution channel is only as mature as its least-mature required constituent.**

The Charmed HPC solution is considered to be at risk channel `X` only when **every
required charm** is available at channel `X` (or better) on a compatible track, and the
cross-charm gates for `X` in this document pass. For example, the solution cannot be
`candidate` if any required charm is still only at `beta`.

#### B.2.2 Track and version alignment

- Define a **solution version matrix** per solution track: the exact track/channel of
  each constituent charm that composes that solution release.
- All constituents in a given solution release must target a **compatible Ubuntu base**
  and be validated against the **same Juju version(s)**.
- Optional/experimental components (e.g. Apptainer, if a deployment does not use
  containers) may lag, but must be explicitly marked optional in the matrix and excluded
  from the required set for that channel.

A minimal matrix template:

```
Solution release: __________   Solution channel: __________   Ubuntu base: __________
Juju version(s) validated: __________

Component          Track        Channel      Revision
slurm-charms       __________   __________   __________
filesystem-charms  __________   __________   __________
sssd-operator      __________   __________   __________
apptainer-operator __________   __________   __________   (optional: yes/no)
```

### B.3 Cross-charm gates per channel

Legend: **[CI]** automated; **[MAN]** manual; **[CI/MAN]** CI evidence, human review.
*(target)* = capability to be built (e.g. BDD suite, full upgrade matrix).

#### B.3.1 Solution `edge`

- [CI] Every required charm is available at `edge` on a compatible track.
- [CI] Full-stack bundle deploys and reaches `active/idle`: `slurmctld` + `slurmd` +
  `slurmdbd` (+`mysql`) + `sackd` + filesystem + SSSD on the target base.
- [CI] Core Slurm relations settle (all four `slurmctld` relations, `slurmdbd:database`).

#### B.3.2 Solution `beta`

Requires solution `edge`, plus:

- [CI] All required charms at `beta` (weakest-link rule).
- [CI] **End-to-end smoke job**: submit a batch job through the login path
  (`sackd`/`slurmrestd` -> `slurmctld` -> `slurmd`) and confirm it completes.
- [CI] Shared filesystem is mounted on compute + login nodes and is writable from a job.
- [CI] SSSD-provided identity can submit and own a job.
- [CI] **HA / resilience**: `slurmctld` leader failover with in-flight/queued jobs
  preserved, recovering to healthy.
- [CI] **Functional / BDD** acceptance of primary cluster journeys *(target,
  `pytest-jubilant-bdd`)*, e.g.:
  - submit and complete an `sbatch` job that reads/writes the shared filesystem,
  - run a containerized job via Apptainer,
  - authenticate a user through SSSD and run a job as that user,
  - query cluster state through `slurmrestd`.
- [CI] **Automated security check**: CVE/dependency scan across the set with **no
  unresolved critical/high** findings.
- [MAN] Bundle/deployment documentation drafted for any changed topology or relation.
- [CI] **Nightly run** passes, including a full-stack run and the scheduled HA job
  *(no fixed bake time)*.

#### B.3.3 Solution `candidate`

Requires solution `beta`, plus:

- [CI] All required charms at `candidate` (weakest-link rule).
- [CI] **Cross-charm upgrade / refresh** in the supported order *(target)*: refresh the
  solution from the previous solution release to the candidate, one component at a time,
  asserting the cluster stays functional (jobs continue to schedule) throughout, then
  verify a **rollback** path.
- [CI] **Interface compatibility matrix**: each relation interface version offered by a
  candidate charm is accepted by the partner charm's candidate revision (no silent
  break across the set).
- [CI] **Integrated observability**: `slurmctld` -> COS produces metrics, alert rules,
  and dashboards; a log sink receives cluster logs.
- [CI] **Non-functional at solution scale** *(target, via `charmed-hpc-benchmarks`)*:
  - multi-node scale-out of `slurmd` and scale-in,
  - `slurmctld` **HA failover** with jobs surviving the failover,
  - cluster **soak >= 48h** with steady job submission and no leaks/flaps,
  - performance baseline (e.g. scheduling throughput, job turnaround) within budget.
- [MAN] Solution documentation complete (deploy, integrate, operate, upgrade).
- [MAN] Solution release notes drafted, listing the component version matrix.
- [MAN] Security review across the set clean of critical/high.

#### B.3.4 Solution `stable`

Requires solution `candidate`, plus:

- [MAN] All required charms at `stable` (weakest-link rule).
- [MAN] **Soak on solution `candidate`:** >= **14 days**, zero regressions, no new
  critical/high defects (proposed default).
- [CI] Upgrade from the **current solution `stable`** to the new release verified end to
  end (data + accounting DB + config preserved; jobs unaffected) *(target)*.
- [CI/MAN] The frozen component version matrix is reproducible and each revision matches
  what soaked on `candidate`.
- [MAN] Security sign-off across the set.
- [MAN] Solution documentation and release notes published (including the version
  matrix and a supported upgrade path from the previous solution `stable`).
- [MAN] Approval recorded by the solution/release owner **and** each component
  maintainer.
- [MAN] Documented rollback plan for the whole solution.

### B.4 Non-functional expectations (solution level)

| Area          | Expectation (proposed default) |
|---------------|--------------------------------|
| Scale         | Validated `slurmd` scale-out and scale-in at the target node count for the release |
| HA / failover | `slurmctld` leader failover with in-flight/queued jobs preserved |
| Soak          | >= 48h at candidate with continuous job submission; no leaks, restarts, or status flaps |
| Performance   | Baseline captured via `charmed-hpc-benchmarks`; regression budget agreed by maintainers |
| Observability | Metrics, alerts, dashboards, and logs verified through COS |
| Security      | No unresolved critical/high across any constituent artifact or dependency |

### B.5 Roles and sign-offs

| Role                 | Responsibility |
|----------------------|----------------|
| Component maintainer | Confirms their charm meets the single-charm gates at the target channel |
| Solution/release owner | Owns the version matrix, cross-charm gates, and the final go/no-go |
| QA / validation      | Runs and signs off soak, scale, HA, and performance evidence |
| Security             | Reviews CVE/dependency scans across the set |
| Docs                 | Confirms solution docs and release notes are published for the release |

`candidate` and `stable` promotions require the solution/release owner plus the
maintainers of every required component.

### B.6 Solution promotion checklist (copy per promotion)

```
Solution release: __________   From: ______  ->  To: ______   Base: ______   Juju: ______

Weakest-link:
[ ] All required charms available at target channel on compatible tracks
[ ] Component version matrix recorded

Deploy & integrate:
[ ] Full-stack bundle reaches active/idle
[ ] Core Slurm relations + slurmdbd:database settle
[ ] Shared filesystem mounted + writable from a job
[ ] SSSD identity can submit/own a job
[ ] Apptainer container job runs (if in scope)

Functional (beta+):
[ ] End-to-end BDD acceptance journeys pass
[ ] slurmctld HA failover with job survival
[ ] slurmrestd query path verified

Upgrade & compatibility (candidate+):
[ ] Cross-charm upgrade in supported order keeps cluster functional
[ ] Rollback path verified
[ ] Interface-compatibility matrix across the set passes
[ ] Integrated observability (metrics/alerts/dashboards/logs) verified

Non-functional (candidate+):
[ ] Scale out/in verified
[ ] Soak >= 48h clean
[ ] Performance baseline within budget

Hygiene:
[ ] Security scan clean of critical/high across the set (beta+)
[ ] Solution docs + release notes published/drafted per channel
[ ] Nightly run passing (beta+); soak time satisfied for target channel

Approvals (candidate+):
[ ] Solution/release owner ______
[ ] Component maintainers: slurm ____  filesystem ____  sssd ____  apptainer ____
```

## References

- [Charmhub track + risk channel release model](https://juju.is/docs/juju/channel)
- [Charmed HPC contributing guide](https://github.com/canonical/hpc-team/blob/main/CONTRIBUTING.md)
