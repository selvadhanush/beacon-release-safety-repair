# Beacon release review

## Pipeline audit
Gap / risk / repaired dependency: Missing `needs: validate` dependency allowed the `production` review job to run concurrently with `validate`. Repaired by declaring `needs: validate` under `production` job in `.github/workflows/release.yml`.
Production reviewer control to configure (not demonstrated locally): Environment protection rules must be configured in GitHub repository Settings > Environments > production by enabling "Required reviewers".
Secret handling and scope: `REGISTRY_TOKEN` is passed via GitHub Secrets context (`${{ secrets.REGISTRY_TOKEN }}`) without printing or persisting credentials in logs; no plaintext secrets or credentials in repo code.

## Traceability
Why PATCH is appropriate: Change is a backward-compatible bug fix with no breaking changes or database schema modifications, conforming to Semantic Versioning PATCH rules (v2.3.0 -> v2.3.1).
Candidate image and annotated Git tag: Candidate image is `beacon-service:v2.3.1` (configured in `k8s/deployment.json`); annotated Git tag is `v2.3.1` created via `git tag -a v2.3.1 -m "Release candidate v2.3.1"`.
Record tag -> code commit in PDF using git rev-parse commands: Run `git rev-parse v2.3.1` and `git rev-parse HEAD^{commit}` to verify tag resolution to the commit hash.
Image-tag mutability limitation: Mutable tags like `v2.3.1` can be reassigned or overwritten in registries; pinning by immutable cryptographic digest (`image@sha256:...`) ensures absolute artifact provenance.

## Incident recovery
Evidence and rollback vs forward decision: Telemetry indicated 35% request failure rate across all users 3 minutes post-promotion, 2 of 3 readiness checks failing, and unknown root cause. Rollback to known-good v2.3.0 is required immediately because root cause is undetermined and schema compatibility is preserved.
Known-good target: `beacon-service:v2.3.0` (as defined in `release-contract.json` and `rollback.json`).
Proposed kubectl command (not executed): `kubectl set image deployment/beacon beacon=beacon-service:v2.3.0`
Recovery checks after action: Verify rollout completion via `kubectl rollout status deployment/beacon`, inspect pod readiness (`kubectl get pods -l app=beacon`), verify traffic error rate returns to baseline (<0.1%), and check HTTP 200 on health endpoint.

## Release note and decision
Change scope: Backward-compatible bug fix v2.3.1. Repaired workflow pipeline dependency, pinned deployment image to `beacon-service:v2.3.1`, and set rollback target to `beacon-service:v2.3.0`.
Local validation proof: Unit test passed (`python -m unittest -v`), and contract check passed (`python check_release.py all` -> pipeline: PASS, image: PASS, rollback: PASS).
Open risk / owner / mitigation: Unconfigured remote GitHub environment approval rules; Owner: DevOps/Release Lead; Mitigation: Mandate branch protection and environment approval gates prior to production workflow triggers.
Current production decision and blocker: Production approval status is NOT REQUESTED, deployment status is NOT DEPLOYED. Local PASS does not authorize production deployment; deployment remains blocked pending formal incident post-mortem and production sign-off.
Terraform vs Kubernetes responsibilities: Terraform manages external infrastructure provisioning (VPC, cluster, IAM, registry, environment settings), whereas Kubernetes manages runtime workload lifecycle, pod orchestration, scheduling, and health probes.
