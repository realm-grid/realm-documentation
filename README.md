# RealmGrid Documentation

Technical documentation for the RealmGrid game server hosting platform.

## Contents

- [Architecture Overview](./architecture/README.md)
- [Subscription Lifecycle](./subscriptions/README.md)
- [Server Management](./servers/README.md)
- [Billing & Payments](./billing/README.md)
- [Infrastructure](./infrastructure/README.md)

## Quick Links

| Component | Description |
|-----------|-------------|
| `realm-functions` | Azure Functions backend (Python) |
| `realm-admin` | Admin dashboard (React/TypeScript) |
| `realm-web` | Customer-facing website |
| `realm-infra` | Terraform & K8s infrastructure |
| `realm-e2e-tests` | End-to-end test suite |

## Environment

| Environment | Purpose |
|-------------|---------|
| `local` | Local development |
| `dev` | Development testing |
| `test` | Integration testing |
| `acc` | Acceptance/staging |
| `prod` | Production |
