# Agent Skills specification for the first Voidstation alpha

## Decision

Voidstation should implement Agent Skills as a filesystem package and instruction-disclosure format. It must not pretend that the format supplies an RPC contract or a security model. The alpha should add a small Voidstation-owned runtime around it: catalog and activation in a Conversation, per-User ownership checks, an explicit script runner, confirmation gates, job events, cancellation, and isolated execution.

The built-in `proof` skill is the first end-to-end check. It prints a fixed JSON result and has no filesystem, network, secret, or user-data capability.

## Research record

- **Checked:** 2026-09-04 UTC.
- **Specification version:** none is published. The authoritative repository identifies `docs/specification.mdx` as the authority for format requirements, and says that its guides, examples, tests, and reference implementation do not add format requirements.[^authority]
- **Snapshot used:** `agentskills/agentskills` `main` at [`69ef37e9424c0a7ea9dd2293b559e43ec8176379`](https://github.com/agentskills/agentskills/tree/69ef37e9424c0a7ea9dd2293b559e43ec8176379), commit date 2026-08-09. The live specification is [agentskills.io/specification](https://agentskills.io/specification).
- **Interpretation rule:** "must" below means the authoritative format page says `must` or marks the field required. The client implementation guide is useful design guidance, not another layer of the standard.

## What Voidstation must accept and produce

### Package and metadata

A compliant skill package is a directory with `SKILL.md` at its root. `SKILL.md` must start with YAML frontmatter and continue with Markdown instructions.[^directory][^format]

The frontmatter contract is small:

| Field | Status | Constraint |
| --- | --- | --- |
| `name` | required | Non-empty, at most 64 characters, lowercase letters, numbers, and hyphens. It cannot begin or end with a hyphen, contain `--`, and must match the parent directory name. |
| `description` | required | Non-empty, at most 1,024 characters. The specification recommends saying what the skill does and when to use it. |
| `license` | optional | License name or reference to a bundled license file. |
| `compatibility` | optional | Non-empty when present and at most 500 characters. It describes environmental needs. |
| `metadata` | optional | String-key to string-value mapping for extra metadata. |
| `allowed-tools` | optional, experimental | Space-separated pre-approved tool names. Support varies by client. |

The Markdown body has no required headings, data shape, input form, output form, or token-level syntax. The format page recommends steps, examples, and edge cases. It recommends, rather than requires, a body below 500 lines and about 5,000 tokens.[^body][^disclosure]

There is one ambiguity worth preserving in the alpha's diagnostics. The frontmatter table says "lowercase letters, numbers, and hyphens," while the `name` subsection says "unicode lowercase alphanumeric characters (`a-z`, `0-9`)". The linked `skills-ref` demo accepts Unicode letters. The prose's parenthetical range and examples are ASCII. Until the specification resolves this, alpha should accept only the unambiguous ASCII expression `^[a-z0-9]+(?:-[a-z0-9]+)*$`, report a clear compatibility diagnostic for other names, and track Unicode-name support separately.[^name][^ref-validator]

### Resources and file references

`scripts/`, `references/`, and `assets/` are named organization conventions, not required package members. A package may contain any other files and directories. The specification describes scripts as executable code, references as on-demand documentation, and assets as static templates, images, or data. It defines no script manifest, interpreter, executable-bit rule, resource schema, or asset API.[^optional-directories]

Skill-authored references should be relative to the skill root. The specification recommends keeping them one level from `SKILL.md` and avoiding nested reference chains.[^references] Voidstation should resolve only normalized descendants of that root. It must reject `..`, absolute paths, and symlinks that leave the copied skill root.

### Discovery and loading

The format page describes progressive disclosure as its loading model: load each available skill's `name` and `description` at startup, load the full `SKILL.md` when activated, then load resources only when needed.[^disclosure]

It does **not** prescribe discovery locations, recursive scanning, package installation, precedence, collision policy, a registry, a prompt representation, an activation API, automatic trigger matching, or whether a host uses a model, a slash command, or both. The implementation guide explicitly calls `.agents/skills/` a convention and says paths and scopes depend on the client.[^discovery-guide]

For this alpha, discover direct child directories with an exactly-cased `SKILL.md` in these ordered roots:

1. Immutable built-in packages shipped with Voidstation.
2. The authenticated User's skill directory, `users/<user-id>/skills/` in Voidstation's managed data volume.

A built-in name cannot be shadowed. A duplicate in the User root is invalid and shown to that User as a diagnostic. Do not scan host-wide home directories, project trees, or remote registries in alpha. That keeps a self-hosted server from accidentally ingesting a repository's instructions and gives each User an obvious ownership boundary.

At Conversation start, create the catalog only from packages the current User may see. Provide `name`, `description`, and the activation handle. Do not inject bodies or resource contents. On activation, inject the body, the package identity, and a bounded list of resource paths. Preserve an activated skill's instruction message when compacting the Conversation and do not inject it twice. This follows the implementation guide's recommended three-tier model and its context advice.[^activation-guide][^context-guide]

### Validation

The format page points implementers to `skills-ref validate` for frontmatter and naming validation.[^validation] The repository calls `skills-ref` a demonstration artifact, not a production SDK or a source of further requirements. Voidstation should therefore own a small validator for the format rules above, and run `skills-ref validate` as a compatibility check in CI, not as the service's only parser.[^authority]

The alpha validator should reject a package from the catalog when `SKILL.md` is missing, frontmatter is absent or invalid YAML, either required field is absent or wrong-typed, a required constraint fails, or the name collides. It should retain the diagnostic for the owning User. It should warn, not grant behavior for, unsupported optional fields. In particular, it must never treat `allowed-tools` as a permission grant because the standard marks it experimental and does not define its semantics.[^frontmatter]

`skills-ref` currently rejects unknown top-level keys, accepts lowercase `skill.md` as a convenience, and accepts Unicode names. Those are demo-library choices and conflict with or exceed the authoritative page. They are useful compatibility observations, not alpha format rules.[^ref-parser][^ref-validator]

## What the standard leaves to Voidstation

None of the following has a normative Agent Skills package or host contract:

| Concern | Standard position | Alpha decision |
| --- | --- | --- |
| Invocation | No invocation protocol or trigger contract. | Support model-selected activation from the catalog and explicit `/skill <name>` activation in a Conversation. An explicit request wins over model selection. |
| Input and output | No typed input, result, streaming, or artifact contract. | The conversation is the input to instructions. Script execution has a Voidstation JSON tool contract and returns stdout, stderr events, exit status, and declared output files. |
| Function or tool schemas | No schema for a skill as a callable function. | Expose host-owned `activate_skill`, `read_skill_resource`, and `run_skill_script` tools. Packages remain Markdown plus files, never RPC definitions. |
| Permissions | `allowed-tools` is experimental and has no portable enforcement model. | Capabilities belong to an individual run, not the package. Resource reads stay inside the copied package. Script execution, workspace access, network access, and output export are separate capabilities. |
| User confirmation | No confirmation flow. | Require an in-Conversation approval for every non-zero-capability action. Show the exact script, arguments, package version or digest, requested capabilities, writable paths, network state, and timeout. Approval covers one run only. |
| Progress | A skill author may write a checklist or a script may use stderr, but no event protocol exists.[^scripts-guide] | Persist ordered job events: `queued`, `awaiting_confirmation`, `running`, `progress`, and one terminal event. Stream events to the owning Conversation. |
| Cancellation | No cancellation protocol or cleanup promise. | The owner may cancel queued, awaiting, or running jobs. Do not start cancelled queued jobs. Send `SIGTERM`, wait a fixed grace period, then kill the sandbox. Record `cancelled`; never claim side effects were rolled back. |
| Errors | Guidance recommends useful script errors and exit codes, but defines no error envelope.[^scripts-guide] | Return a machine-readable code plus a safe message: `invalid_skill`, `not_found`, `forbidden`, `confirmation_required`, `confirmation_declined`, `execution_failed`, `timed_out`, `cancelled`, or `environment_unavailable`. Keep raw diagnostics in the owner's job record. |
| Isolation | No isolation, filesystem, secrets, or network model. | Run each script in a fresh sandbox with a read-only package copy, a per-job writable output directory, a distinct Unix identity, no inherited secrets, and network disabled unless the User confirms it. Set CPU, memory, disk, wall-time, and output-size limits. |
| Per-User access | No identity or tenancy model. | Every skill, activation, job, event, artifact, approval, and cancellation check carries `user_id`. Only that User can list, invoke, read, or cancel it. |
| Long-running work | No durable job, reconnection, timeout, or resume protocol. | Persist jobs before execution. A running job continues through a browser reconnect, emits heartbeat or progress events, has a default wall-time limit, and ends in one terminal state. Alpha does not resume a process after a server restart. |

The implementation guide discusses explicit user activation, dedicated activation tools, permission enforcement, resource allowlisting, and protecting skill instructions during compaction. They are sound choices, but the source labels the page a guide and the repository reserves format authority to the specification.[^activation-guide][^context-guide][^authority]

## Alpha runtime contract

Keep the package contract separate from the execution contract.

```text
activate_skill(name)
  -> { name, instructions, skillRoot, resources[] }

read_skill_resource(name, relativePath)
  -> { relativePath, content | binaryArtifactRef }

run_skill_script(name, relativePath, args[])
  -> { jobId, state }

job event
  -> { jobId, sequence, state, message?, timestamp }

job result
  -> { jobId, state, exitCode?, stdout?, artifacts[], error? }
```

`name` in every tool call is constrained to the current User's catalog. `relativePath` must identify a regular file below `scripts/` or a resource below the copied package root, depending on the tool. The runner never accepts an arbitrary shell string. It selects an approved interpreter by extension, supplies `args` as an argument array, and sets the working directory to the sandbox's package copy. A package's Markdown can still instruct the assistant how to use the result. It cannot expand the runner's capabilities.

Activation itself has no external side effect and needs no confirmation. `run_skill_script` moves to `awaiting_confirmation` unless the package is Voidstation-built-in and the selected action requests no capability beyond executing the fixed proof script. The UI must still show activation, run, result, and failure events in the Conversation.

## Implementation plan and acceptance checks

1. **Package catalog and validator.** Implement the two discovery roots, exact `SKILL.md` parsing, ASCII name rule, frontmatter validation, digesting, collision policy, and owner-only diagnostics. Add fixtures for valid minimal packages, every required-field failure, optional metadata, path escape attempts, collisions, and malformed YAML.
2. **Conversation activation.** Add catalog disclosure at Conversation start, `/skill <name>`, `activate_skill`, body injection, resource inventory, activation audit events, and compaction protection. Test automatic selection and explicit activation against an authorized catalog only.
3. **Owned job service.** Implement the three host tools, job and event persistence, state transition guards, resource path checks, safe output capture, and result rendering. Test every terminal state and verify a second User gets `forbidden` for a skill, job, event, and artifact.
4. **Confirmation and sandbox runner.** Add per-run confirmation, capability summaries, rootless fresh sandboxes, quotas, deadline handling, cancellation escalation, no-secrets environment, and network-off default. Test declined approval, timeout, cancellation before start, cancellation while running, and a script attempting path traversal or network access.
5. **Built-in proof package.** Ship `builtin-skills/proof/SKILL.md` and `scripts/prove.sh`. The package name and directory are `proof`; its script prints exactly `{"proof":"voidstation-skill-invocation","version":1}` followed by a newline, takes no arguments, reads and writes no files, and makes no network request. Exercise discovery, explicit activation, execution, result rendering, and cancellation UI with this package.
6. **Deployment gate.** The production image includes the built-in package and supported interpreter, runs the validator fixture suite, and refuses script execution with `environment_unavailable` if the sandbox runtime is missing. Publish the capability and lifecycle rules in the operator documentation before enabling user-supplied scripts.

## Follow-up questions made sharp by this research

1. Should Voidstation submit a specification discussion asking whether valid names are ASCII-only or Unicode lowercase alphanumeric, then switch alpha only after an upstream answer?
2. Is a rootless container runtime an acceptable required dependency for self-hosted alpha script execution, or must Voidstation support a second sandbox backend before user-supplied scripts are enabled?
3. Which User action grants workspace read, workspace write, or network access, and should any grant persist beyond one job?
4. What does cancellation mean for a capability that has already made an external request? The job state can be cancelled, but the action may have completed.

[^authority]: [Agent Skills repository instructions, "Authority and boundaries"](https://github.com/agentskills/agentskills/blob/69ef37e9424c0a7ea9dd2293b559e43ec8176379/AGENTS.md#authority-and-boundaries).
[^directory]: [Specification, "Directory structure"](https://agentskills.io/specification#directory-structure), [snapshot](https://github.com/agentskills/agentskills/blob/69ef37e9424c0a7ea9dd2293b559e43ec8176379/docs/specification.mdx#directory-structure).
[^format]: [Specification, "SKILL.md format"](https://agentskills.io/specification#skillmd-format), [snapshot](https://github.com/agentskills/agentskills/blob/69ef37e9424c0a7ea9dd2293b559e43ec8176379/docs/specification.mdx#skillmd-format).
[^frontmatter]: [Specification, "Frontmatter"](https://agentskills.io/specification#frontmatter), [snapshot](https://github.com/agentskills/agentskills/blob/69ef37e9424c0a7ea9dd2293b559e43ec8176379/docs/specification.mdx#frontmatter).
[^name]: [Specification, "name field"](https://agentskills.io/specification#name-field), [snapshot](https://github.com/agentskills/agentskills/blob/69ef37e9424c0a7ea9dd2293b559e43ec8176379/docs/specification.mdx#name-field).
[^body]: [Specification, "Body content"](https://agentskills.io/specification#body-content), [snapshot](https://github.com/agentskills/agentskills/blob/69ef37e9424c0a7ea9dd2293b559e43ec8176379/docs/specification.mdx#body-content).
[^optional-directories]: [Specification, "Optional directories"](https://agentskills.io/specification#optional-directories), [snapshot](https://github.com/agentskills/agentskills/blob/69ef37e9424c0a7ea9dd2293b559e43ec8176379/docs/specification.mdx#optional-directories).
[^references]: [Specification, "File references"](https://agentskills.io/specification#file-references), [snapshot](https://github.com/agentskills/agentskills/blob/69ef37e9424c0a7ea9dd2293b559e43ec8176379/docs/specification.mdx#file-references).
[^disclosure]: [Specification, "Progressive disclosure"](https://agentskills.io/specification#progressive-disclosure), [snapshot](https://github.com/agentskills/agentskills/blob/69ef37e9424c0a7ea9dd2293b559e43ec8176379/docs/specification.mdx#progressive-disclosure).
[^validation]: [Specification, "Validation"](https://agentskills.io/specification#validation), [snapshot](https://github.com/agentskills/agentskills/blob/69ef37e9424c0a7ea9dd2293b559e43ec8176379/docs/specification.mdx#validation).
[^discovery-guide]: [Adding skills support, "Step 1: Discover skills"](https://agentskills.io/client-implementation/adding-skills-support#step-1-discover-skills), [snapshot](https://github.com/agentskills/agentskills/blob/69ef37e9424c0a7ea9dd2293b559e43ec8176379/docs/client-implementation/adding-skills-support.mdx#step-1-discover-skills).
[^activation-guide]: [Adding skills support, "Steps 3 and 4"](https://agentskills.io/client-implementation/adding-skills-support#step-3-disclose-available-skills-to-the-model), [snapshot](https://github.com/agentskills/agentskills/blob/69ef37e9424c0a7ea9dd2293b559e43ec8176379/docs/client-implementation/adding-skills-support.mdx#step-3-disclose-available-skills-to-the-model).
[^context-guide]: [Adding skills support, "Step 5: Manage skill context over time"](https://agentskills.io/client-implementation/adding-skills-support#step-5-manage-skill-context-over-time), [snapshot](https://github.com/agentskills/agentskills/blob/69ef37e9424c0a7ea9dd2293b559e43ec8176379/docs/client-implementation/adding-skills-support.mdx#step-5-manage-skill-context-over-time).
[^scripts-guide]: [Using scripts, "Designing scripts for agentic use"](https://agentskills.io/skill-creation/using-scripts#designing-scripts-for-agentic-use), [snapshot](https://github.com/agentskills/agentskills/blob/69ef37e9424c0a7ea9dd2293b559e43ec8176379/docs/skill-creation/using-scripts.mdx#designing-scripts-for-agentic-use).
[^ref-parser]: [`skills-ref` parser](https://github.com/agentskills/agentskills/blob/69ef37e9424c0a7ea9dd2293b559e43ec8176379/skills-ref/src/skills_ref/parser.py).
[^ref-validator]: [`skills-ref` validator](https://github.com/agentskills/agentskills/blob/69ef37e9424c0a7ea9dd2293b559e43ec8176379/skills-ref/src/skills_ref/validator.py).
