# AGENTS.md

This repository hosts the development specifications for [Charmed HPC](https://github.com/canonical/charmed-hpc-docs). Each specification is a self-contained Markdown document under `specs/` that proposes, defines, or documents a design decision for the Charmed HPC ecosystem.

This file is the entry point for AI coding agents (and humans) working with the specs. Read it first; then read the individual specs it points you to.

## How to use this repo

- **Specs are authoritative.** When a spec and a downstream charm disagree, the spec wins unless the spec has been superseded by a later one.
- **Specs are not summaries.** They contain full rationale, evaluated alternatives, and code samples. Do not paraphrase them into a shorter document — read them directly.
- **Implementation status is not tracked here.** This repo records *what was decided*, not *what has been built*. To check whether a spec has been implemented, consult the relevant charm repository (e.g. `slurm-charms`, `charmed-hpc-libs`).
- **Index numbers are stable.** A spec's `UHPCNNN` index is permanent. Gaps in the sequence (e.g. UHPC007, UHPC014, UHPC015) are reserved or withdrawn numbers — do not reuse them.

## Spec index

| Index | Title | Summary | Builds on |
|-------|-------|---------|-----------|
| [UHPC001](specs/UHPC%20001%20-%20Dynamic%20nodes%20in%20Charmed%20HPC/uhpc-001.md) | Dynamic compute nodes in Charmed HPC | Refactor Slurm charms to use dynamic compute node enlistment so `slurmctld` and `slurmd` can scale without `slurm.conf` edits. | — |
| [UHPC002](specs/UHPC%20002%20-%20slurmctld%20high-availability%20implementation%20in%20Charmed%20HPC/uhpc-002.md) | slurmctld high-availability implementation in Charmed HPC | Implement active-passive HA for `slurmctld` using a shared `StateSaveLocation` via `filesystem-client`; update related charms to accept multiple controller addresses. | UHPC001 |
| [UHPC003](specs/UHPC%20003%20-%20Release%20policy%20and%20notes%20for%20Charmed%20Slurm/uhpc-003.md) | Release policy for Charmed Slurm | Define the release cadence and support policy for Charmed Slurm. | — |
| [UHPC004](specs/UHPC%20004%20-%20Replace%20node-configured%20action/uhpc-004.md) | Replace `node-configured` action | Replace the existing `node-configured` action on `slurmd` with a more scalable alternative. | UHPC005 |
| [UHPC005](specs/UHPC%20005%20-%20Use%20unit%20name%20as%20node%20name/uhpc-005.md) | Use unit name as node name | Use Juju unit names (instead of hostnames) as Slurm node names for `slurmd` units. | UHPC001 |
| [UHPC006](specs/UHPC%20006%20-%20User%20email%20notifications%20in%20Charmed%20Slurm/uhpc-006.md) | User email notifications in Charmed Slurm | Add job-status email notifications via Slurm-Mail and a new `smtp` relation to `slurmctld`. | — |
| [UHPC008](specs/UHPC%20008%20-%20%60charmed-hpc-libs%60%20for%20HPC%20charm%20development/uhpc008.md) | `charmed-hpc-libs` for HPC charm development | Formally define and rename `hpc-libs` to `charmed-hpc-libs`; clarify what belongs in the library vs. local workspaces. | — |
| [UHPC009](specs/UHPC%20009%20-%20Distributing%20Slurm%20interfaces%20as%20Python%20packages/uhpc009.md) | Distributing Slurm interfaces as Python packages | Distribute Slurm interface implementations as individual Python packages in the `slurm-charms` monorepo. | UHPC008 |
| [UHPC010](specs/UHPC%20010%20-%20Observer%20design%20pattern%20in%20HPC%20charms/uhpc010.md) | Observer design pattern in HPC charms | Adopt the observer pattern in HPC charms to reduce `charm.py` complexity and enforce public APIs via forward references. | UHPC008 |
| [UHPC011](specs/UHPC%20011%20-%20Common%20justfile%20format%20for%20Charmed%20HPC%20repositories/uhpc011.md) | Common justfile format for Charmed HPC repositories | Define a common set of `just` recipes shared across all Charmed HPC repositories. | — |
| [UHPC012](specs/UHPC%20012%20-%20Debian%20packaging%20s2n-tls%20for%20TLS%20support%20in%20Slurm/uhpc012.md) | Debian packaging s2n-tls for TLS support in Slurm | Plan for packaging `s2n-tls` as a Debian package for use by Slurm. | — |
| [UHPC013](specs/UHPC%20013%20-%20SSH%20access%20for%20login%20nodes/uhpc013.md) | SSH access for login nodes | Survey options for publicly exposing Charmed HPC login nodes over SSH without relying on Juju's internal SSH key management. | — |
| [UHPC016](specs/UHPC%20016%20-%20Charm%20configuration%20observer/uhpc016.md) | Charm configuration observer | Add a `ConfigObserver` to `charmed-hpc-libs` so charms using the _conditions_ pattern can interact with `juju config` data through a common interface. | UHPC010 |

## Spec structure

Every spec under `specs/` follows this structure. The headings and order make specs predictable for both people and agents:

1. **YAML front matter** — `index` (for example, `UHPC017`) and `title`.
2. **`## Abstract`** — a concise statement of the proposal or decision.
3. **`## Rationale`** — the problem, users, and constraints that motivate the change.
4. **`## Specification`** — the normative design. Describe the intended behavior, interfaces, responsibilities, and relevant migration or compatibility requirements. Include code samples when they clarify the design.
5. **`## Further information`** — references, related specifications, and relevant external discussions.

Add sections such as `## Scope`, `## Risks`, `## Future work`, or `### Considerations` when they make the design clearer. Keep the specification itself authoritative: distinguish requirements from examples, implementation notes, and open questions.

## Conventions for writing a new spec

When drafting a specification:

- **Check whether a new spec is needed.** Read the index and related specs first. Amend an existing spec when the proposal changes or clarifies an existing decision; do not create a second spec for the same decision.
- **Start from the problem and desired outcome.** Define who needs the change, what problem it solves, the scope and non-goals, and the constraints that shape the design.
- **Use existing specs as design constraints.** Follow the "Builds on" relationships and cross-link related decisions. A downstream repository may provide implementation context, but its current behavior does not override an existing spec or define a new design by itself.
- **Pick the next free index.** Use the next `UHPCNNN` number after the highest existing one. Do not fill gaps in the sequence.
- **Create one folder per spec.** Folder name format: `UHPC NNN - <Sentence-style title>`. Capitalize only the first word and any proper nouns or acronyms. Example: `UHPC 017 - My new thing`.
- **Use the current filename convention for new specs.** Name the file `uhpc-NNN.md` with a three-digit, hyphenated index. Earlier specs from UHPC008 onward use the legacy `uhpcNNN.md` form; do not rename them just to make them consistent.
- **Include the required front matter.** Every spec starts with:
  ```yaml
  ---
  index: UHPC017
  title: My new thing
  ---
  ```
- **Document important alternatives.** Explain non-obvious alternatives that were considered and why they were rejected. Record risks, trade-offs, open questions, and future work when they affect how the decision should be understood or implemented.
- **Cross-link related specs.** Use relative Markdown links to other specs in `specs/`. URL-encode spaces and special characters in folder names (for example, `UHPC%20008`).
- **Update the index.** Add the new spec, its one-sentence summary, and its "Builds on" relationships to the index table in this file when the spec is opened for review.

A new spec should provide enough information for another team to implement the decision without inferring the design from a downstream repository. If the design depends on a decision that has not been made, record that as an open question instead of silently choosing an implementation-specific answer.

## When you're asked to implement a spec

Treat the specification and the specs it builds on as the design authority. The implementation repositories are downstream consumers of that design.

1. **Read the design first.** Read the target spec end-to-end, including evaluated alternatives, risks, future work, and further information. Then read every spec listed in its "Builds on" entry and any directly related specs.
2. **Extract an implementation plan.** Identify the normative requirements, affected components and repositories, public interfaces, compatibility or migration requirements, and the tests needed to demonstrate the behavior.
3. **Inspect downstream repositories for context only.** Use their current code to locate integration points, understand existing behavior, and identify what needs to change. Do not treat existing implementation as the source of truth or silently preserve behavior that conflicts with the spec.
4. **Resolve conflicts explicitly.** If the spec is ambiguous or the existing repositories expose a constraint the spec does not address, do not invent a design from the code. Record the issue, ask for clarification when it affects the decision, or make the smallest conservative interpretation and document it. If the decision itself needs to change, update or supersede the spec before implementing the new behavior.
5. **Implement and validate in the downstream repository or repositories.** Keep implementation changes in those repositories; do not rewrite the design spec merely to match the current code. Add or update tests and documentation as required by the spec.
6. **Report deviations and follow-up work.** Clearly identify any requirement that could not be implemented, any intentional deviation, and any spec amendment or cross-repository work that is still needed.
