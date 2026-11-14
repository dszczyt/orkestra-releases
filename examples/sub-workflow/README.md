# Lead intake with a sub-workflow

`workflow.yaml` scores a lead by running `scoring/workflow.yaml` as a child run, then saves it to a CRM. If scoring fails, the lead is kept, unscored.

- `orkestra/operation/workflow` starts the child run with the input it computes (`{lead: .input.lead}`). The step's output is the child run's result: the output of the child's last step.
- A failed child run fails the step with `_error.type: child_run`, carrying the child run's `run_id`, `status`, and the `_error` that failed it as `cause`. A `status: failed` transition routes it like any failure.
- The child run has its own tasks, its own `task()` scope and its own configmaps. It sees the parent only through its input.

## Pinning

`orkestra validate workflow.yaml` prints one hash covering both files: editing `scoring/workflow.yaml` changes it. A server run is pinned to that hash, so a child run uses the exact definition its parent was started with. Register both files over the API with `includes`:

```bash
curl -X POST "$ORKESTRA_URL/v1/workflows" -H "Authorization: Bearer $ORKESTRA_TOKEN" \
  -d "$(jq -n --rawfile w workflow.yaml --rawfile s scoring/workflow.yaml \
        '{workflow: $w, includes: {"scoring/workflow.yaml": $s}}')"
```

## Test it

```bash
orkestra test workflow.yaml workflow_test.yaml
```

The child run executes in process against the same mocks. `.runs` lists the child runs, and `.steps` and `.history` hold the root run's steps only.
