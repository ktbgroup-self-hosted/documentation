# Homelab Infrastructure Documentation

This documents the architecture, standards, and operational practices for my homelab,
treated as a production environment to minimise maintenance overhead and maximise reliability.

## Contents

- [Docker Standards & Practices](./docker.md) — compose structure, naming conventions, environment management
- [Secrets Management](./secrets.md) — how secrets are injected and managed
- [Networking](./networking.md) — ingress, DNS, inter-stack networking
- [Security](./security.md) — users, permissions, container hardening
- [Runbooks](./runbooks/) — operational procedures
  - [Adding a New Stack](./runbooks/new-stack.md)
  - [Staging & Promotion](./runbooks/staging.md)
  - [Server Provisioning](./runbooks/provisioning.md)

## Principles

- **Everything is code** — all configuration lives in Git, nothing is configured manually
- **Least privilege** — containers, users, and env vars only get what they need
- **Low maintenance** — prefer boring, well-understood tools over cutting edge
- **Parity** — staging mirrors production as closely as possible
- **Secrets never in Git** — no credentials, tokens, or sensitive values in any repository
