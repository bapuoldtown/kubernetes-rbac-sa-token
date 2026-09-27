# CLAUDE.md

Guidance for Claude Code working in this repository.

## What this is

A hands-on lab teaching production-grade Kubernetes security, one concept per stage.
The pattern in every stage: **learn the secure pattern → apply it → confirm the guardrail
is live → understand the trade-off.** No adversarial cluster-breaking; controls are
demonstrated constructively.

## Structure

- One folder per stage: `s1-pod-security/`, `s2-psa/`, `s3-rbac/`, and so on.
- Each stage uses its own namespace (`s1-pod`, `s2-psa`, `s3-rbac`, …) and is torn down
  with `kubectl delete ns` when done, so each concept is learned in isolation.
- Manifests are plain, readable YAML. A per-stage `README.md` documents the concept,
  the weak case, and the fix.

## Working conventions

- Explain heavy concepts in plain language, one step at a time — one file/one thing,
  run it, then the next.
- Every security control is shown working (verify commands + expected output), not just
  applied.
- Prefer minimal, readable manifests over abstractions or generators.
- Target environment is a local kind cluster.

## Safety

- This lab targets a local kind context only. Do not run destructive commands against
  any other kube context.
- Never commit Terraform state (`*.tfstate`) or kubeconfig files — they contain the
  cluster's private keys. See `.gitignore`.