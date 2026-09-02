# Solution Steps

1. Edit `.github/workflows/ci.yml` to make deployments deterministic and safe by: (1) tagging the built Docker image with the exact `github.sha`, (2) deploying only on `push` events to `refs/heads/main`, (3) forcing compose to recreate using the locally loaded image, and (4) adding a bounded health-poll with retry/sleep plus rollback on failure.

2. Update the `build` job to replace any non-commit tags (like `latest`) with `inventory-sync:${{ github.sha }}` and ensure the resulting image tarball contains that commit-tagged image.

3. Update the `deploy` job to add an `if:` gate that only allows it for `github.event_name == 'push'` and `github.ref == 'refs/heads/main'`, then: download the artifact, `docker load` it, and run `docker compose up -d` with `APP_TAG='${{ github.sha }}'` (and `--force-recreate --pull never`) so the running container comes from the build artifact.

4. Update the `verify` job similarly with the same main-push gate, then implement bounded polling against `http://127.0.0.1:8000/health` using a `for` loop with `seq` and `sleep` between attempts; write `.ci_runs/verify_result.json` with the SHA and whether it became healthy.

5. Ensure the verification pipeline is robust: if health polling fails, run a rollback step (`docker compose down --remove-orphans`) with `if: ${{ failure() }}`.

6. Add a final reporting step with `if: ${{ always() }}` that writes a clear deployment report including the commit SHA and health/outcome keywords to `deploy-report.json` and `.ci_artifacts/deploy-report/deploy-report.json`.

7. Verify the workflow text expectations from the provided tests: the workflow must reference `github.sha`, must not reference `:latest` or `APP_TAG=latest`, must contain `docker save` and `docker load`, must include curl polling for `/health` with bounded retry/sleep, and must contain teardown (`docker compose down`) plus an `always()`-guarded report step.

