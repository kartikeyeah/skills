---
name: clay-plugin-eval
description: Set up Clay and run a NeoSigma paired experiment that compares the Clay plugin with a no-plugin control. Use when a user wants Codex to create their Clay Vault, run the fixed plugin dataset, and generate the comparison report.
---

# Clay plugin experiment

Set up one user's Clay access, run the fixed dataset with and without Clay, and return the reward comparison.

## Defaults

- Dataset ID: `2ea88908-b577-4df5-814c-4f6a8ebd5ee8` (fixed)
- Project ID: `129f3404-2f7f-4df8-9fcc-0164ffe92d73`
- Environment ID: `00b9da33-0bd2-440f-a740-6098200e5721`
- Runtime: `codex:gpt-5.6-sol`
- Artifacts: `offer.md`, `icp.md`, and `leads.csv`

Use the default project, environment, and runtime unless the user provides replacements. Ask for source task IDs, and run the entire dataset only when the user explicitly requests it.

After resolving overrides, use these shell variables in one persistent terminal session:

```bash
NEOSIGMA_PROJECT_ID="129f3404-2f7f-4df8-9fcc-0164ffe92d73"
NEOSIGMA_ENVIRONMENT_ID="00b9da33-0bd2-440f-a740-6098200e5721"
RUNTIME="<selected-runtime>"
NEOSIGMA_DATASET_ID="2ea88908-b577-4df5-814c-4f6a8ebd5ee8"
```

## Set up Clay and its Vault

1. Confirm `neosigma` and `jq` are available and `NEOSIGMA_API_KEY` and
   `OPENAI_API_KEY` are set. If the CLI is missing or unauthenticated, follow
   [NeoSigma CLI getting started](https://docs.neosigma.ai/cli/getting-started).
   Never print either key.

2. If the Clay plugin is not installed, follow the official
   [Clay setup](https://github.com/clay-run/agent-plugins/blob/main/GETTING_STARTED.md):

   ```bash
   codex plugin marketplace add clay-run/agent-plugins
   ```

   Ask the user to install **Clay** from **Plugins**, then run `clay:setup`. If
   the new plugin is not visible yet, restart Codex and resume this skill. If
   `clay` is already available and current, do not reinstall it.

3. Resolve the selected project and its workspace:

   ```bash
   PROJECT_JSON="$(neosigma projects get --project-id "$NEOSIGMA_PROJECT_ID")"
   NEOSIGMA_WORKSPACE_ID="$(printf '%s' "$PROJECT_JSON" | jq -r '.project.workspace_id')"
   WORKSPACE_JSON="$(neosigma workspaces get --workspace-id "$NEOSIGMA_WORKSPACE_ID")"
   ```

   Before continuing, show the user the project name and ID and the workspace
   name and ID. Let the user replace the project or environment at this point.

4. Create an isolated Clay login for this user's Vault:

   ```bash
   CLAY_EVAL_CONFIG_HOME="$(mktemp -d)"
   CLAY_CONFIG_HOME="$CLAY_EVAL_CONFIG_HOME" clay login --device
   CLAY_CONFIG_HOME="$CLAY_EVAL_CONFIG_HOME" clay whoami
   ```

   The user completes the Clay sign-in. Do not expose or copy the credential
   anywhere except the Vault import.

5. Create the Vault in the resolved workspace and import the isolated login:

   ```bash
   NEOSIGMA_VAULT_ID="$(
     neosigma vaults create \
       --workspace-id "$NEOSIGMA_WORKSPACE_ID" \
       --display-name "Clay plugin GTM Bench experiment" |
     jq -r '.vault.id'
   )"

   neosigma vaults credentials import \
     --vault-id "$NEOSIGMA_VAULT_ID" \
     --format clay_cli_config \
     --config-json "$(jq -c . "$CLAY_EVAL_CONFIG_HOME/clay/config.json")"
   ```

   Delete only the temporary directory created in step 4 after the import
   succeeds. Do not create a second Vault when resuming the same run.

## Run and report

1. Initialize the current directory with the selected project, environment, and
   new Vault. If `.neosigma/plugin-experiment.json` already differs, ask before
   replacing it with `--force`.

   ```bash
   neosigma experiments plugin init \
     --plugin clay \
     --project "$NEOSIGMA_PROJECT_ID" \
     --environment "$NEOSIGMA_ENVIRONMENT_ID" \
     --vault "$NEOSIGMA_VAULT_ID"
   ```

2. Run a selected task first with `--dry-run`. Repeat `--task` for more tasks, or
   omit it only when the user requested the full dataset.

   ```bash
   neosigma experiments plugin run \
     --dataset "$NEOSIGMA_DATASET_ID" \
     --task "<source-task-id>" \
     --runtime "$RUNTIME" \
     --artifact offer.md \
     --artifact icp.md \
     --artifact leads.csv \
     --attempts 1 \
     --max-concurrency 2 \
     --dry-run
   ```

   After validation succeeds, run the same command without `--dry-run`.

3. Read `experiment.id` from the response into `NEOSIGMA_EXPERIMENT_ID`, then
   poll `neosigma experiments plugin show "$NEOSIGMA_EXPERIMENT_ID"` until
   completion or failure. Do not start a replacement experiment automatically.

4. Save the detailed results and create the report:

   ```bash
   neosigma experiments plugin results "$NEOSIGMA_EXPERIMENT_ID" --json \
     > clay-plugin-results.json

   neosigma experiments plugin report "$NEOSIGMA_EXPERIMENT_ID" \
     --html ./clay-plugin-report.html
   ```

Report the selected project and workspace, experiment ID, status, named reward
deltas, failed trials, trace session IDs, and both output paths. Do not declare
a global winner.
