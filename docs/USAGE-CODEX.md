# Using Service Approval with Codex

Codex runs the shared Agent Skills pipeline: Intake → Assess → Research →
Validate → Map → Recommended Guidance → Summarize. The DSQL/custom CCC
[end-to-end test](TESTING-CODEX.md) completed with connected MCP tools and
explicit validation commands.

## Setup

Install the [common prerequisites](../README.md#prerequisites), including the
repository's Python requirements and `uv`. Configure the two required research
servers using [MCP configuration for Codex](MCP.md#codex).

Codex reads `AGENTS.md` and discovers skills under `.agents/skills/`. This
repository already supplies that discovery symlink. Open the clone in Codex to
work on the harness itself. See [OpenAI's skill documentation](https://developers.openai.com/codex/skills/)
for discovery locations and symlink support.

For a customer run, create a separate workspace so inputs, downloaded threat
models, generated state, and review ZIPs stay outside the source checkout.
The following macOS/Linux example starts from the cloned repository root:

```bash
SA_REPO="$(pwd)"
SA_RUN="$HOME/service-approval-runs/dsql"
mkdir -p "$SA_RUN/.agents"
ln -s "$SA_REPO/skills" "$SA_RUN/.agents/skills"
ln -s "$SA_REPO/AGENTS.md" "$SA_RUN/AGENTS.md"
```

Use a new run directory, or inspect existing links before reusing one. Open
`SA_RUN` as the workspace in Codex. The links expose shared instructions without
copying customer material into Git. On Windows, creating directory symlinks
requires Developer Mode or suitable permissions; use the corresponding
PowerShell `New-Item -ItemType SymbolicLink` commands with absolute target paths.

In the run workspace, ask Codex to resolve the skill symlink to the installed
repository and use:

- The repository virtual environment's Python interpreter.
- The external run workspace as the working directory for every pipeline command.
- Absolute paths for repository scripts and bundled references. For module
  commands (`python -m tools.validate...`), put the repository on `PYTHONPATH`.

For example, in a shell where `SA_REPO` and `SA_RUN` are set as above:

```bash
cd "$SA_RUN"
export PATH="$SA_REPO/.venv/bin:$PATH"
export PYTHONPATH="$SA_REPO${PYTHONPATH:+:$PYTHONPATH}"
```

Set these for each execution environment that runs commands; a shell export does
not configure an already-running Codex session. Agents must pass the equivalent
working directory, interpreter, and module path on their tool calls. Do not
change into the source checkout to run pipeline commands: some final validators
discover runtime state directly from the current working directory.

## Verify setup

Ask Codex to confirm that it can:

1. Read the linked `AGENTS.md` and `.agents/skills/service-approval/SKILL.md`.
2. Import `yaml` and `jsonschema` with the selected Python interpreter.
3. Search AWS documentation through both required MCP servers and read an
   applicable official AWS page with the available documentation tools.
4. Reach GitHub for Intake's published threat-model retrieval.

Use the existing connected tools if their names carry a client or plugin prefix.
Match their documented capabilities to the skills' requested operations.
Keep the canonical MCP log entries required by the orchestrator. Setup searches
do not substitute for the service research performed during the run.

The install scripts do not register Codex hooks. Complete the validation steps
below even when no automatic hook fires.

## Run the pipeline

Example prompt for an external run workspace:

```text
Read AGENTS.md and follow .agents/skills/service-approval/SKILL.md.
Run the complete pipeline for Amazon Aurora DSQL with --framework=custom
and --framework-file=/absolute/path/to/customer-inputs/ccc-rdms-catalog.md.
Use this external workspace as the working directory and resolve scripts and
references from the linked skill's repository. Use that repository's Python
virtual environment and put the repository on PYTHONPATH for module commands.
Run every phase, review gate, and explicit validation command.
At completion, give me the summary and package the run outputs into a ZIP
in this workspace for review.
```

Replace the framework path with your own file. These are skill arguments in a
prompt, not Codex CLI flags. Intake retrieves the published AWS Threat Model;
do not supply an unverified local model to replace it.

The orchestrator can use Codex's native subagents for independent research and
review. It respects the available concurrency and waits at phase boundaries.
Where subagents are unavailable, it uses the sequential path and retains the
same checks and review requirements.

## Validation

Codex must perform the checks in each `SKILL.md`, including the mapping review,
reference/enforcement checks, generation review, and conflict review. Explicit
execution provides validation; it does not install automatic Codex hooks.

After each state write, run `tools/validate/validate_state.py <absolute-file-path>`.
After each generated control write, run
`tools/validate/validate_controls.py <absolute-file-path>`.
Also recheck the published model and phase prerequisites as instructed by the
skills. Fix failing output before downstream phases consume it.

Run all commands from the external runtime workspace with the repository Python
environment. For example, after Map completes:

```bash
python3 "$SA_REPO/tools/validate/check_phase_complete.py" --slug dsql --phase 04-map
```

Before reporting completion, verify all phase outputs and run final integrity
and cross-artifact checks explicitly. This final block complements the per-phase
checks and reviews; it does not replace them:

```bash
set -e
for phase in 00-intake 01-assess 02-research 03-validate 04-map 05-recommended-guidance 06-summarize; do
  python3 "$SA_REPO/tools/validate/check_phase_complete.py" --slug dsql --phase "$phase"
done
python3 "$SA_REPO/tools/validate/validate_pipeline_integrity.py"
python3 "$SA_REPO/tools/validate/validate_cross.py" \
  .service-approval/dsql/05-recommended-guidance
python3 "$SA_REPO/tools/validate/hook_stop.py"
```

Replace `dsql` with the canonical slug for another service. Capture each check's
result in the run diagnostics. A Stop-hook exit of zero on an empty directory,
or a `snoozed` result after repeated failures, does not prove success. The phase
checks, direct integrity check, and direct cross-artifact check must pass.

Optional artifact engines and Tier 2 AWS checks have their own prerequisites.
Disclose unavailable tools and skipped checks rather than reporting them as passed.

## Outputs and resume

The summary is `.service-approval/dsql/06-summarize/SUMMARY-REPORT.md` inside the
external workspace. Recommended controls are under
`.service-approval/dsql/05-recommended-guidance/`; logs, phase state, and the pinned
source snapshot remain alongside them.

To resume, reopen the same workspace and ask Codex to follow the orchestrator's
resume procedure. It revalidates the original intake and snapshot before
continuing from the last complete phase. Do not alter the snapshot or intake seal
to make a different workspace pass. An explicit new run retrieves the latest
published model.

Keep customer inputs, model snapshots, model-derived runtime state, and output
ZIPs outside Git, including PR attachments. A passing workflow does not mean every
framework objective is covered; review the unresolved items and generated guidance.
The recorded test did not deploy resources or validate policies against a live
AWS account.
