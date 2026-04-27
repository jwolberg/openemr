## Where it fits
OpenEMR UI
  ↓
Clinical Co-Pilot Module
  ↓
Patient-scoped backend endpoint
  ↓
OpenEMR data access / API / services
  ↓
Patient Context Builder
  ↓
LLM Agent Service
  ↓
Citation + Verification Layer
  ↓
Audit Log
  ↓
Response shown inside OpenEMR


## Module level in OpenEMR

openemr/
  interface/
    modules/
      custom_modules/
        clinical_copilot/
          info.txt
          table.sql
          moduleSettings.php
          public/
            js/
            css/
          src/
            ...


# OpenEMR functions
User login/session.
Patient chart context.
Patient record storage.
Role/access control integration.
UI shell.
Audit linkage.
Source-of-truth data.

# Clinical Co-pilot functions
Rendering the assistant panel.
Calling a patient-scoped backend endpoint.
Building a source-labeled patient context packet.
Calling the LLM service.
Verifying/citing claims.
Returning a safe answer.
Logging every request.

# External AI Functions
Direct medical info only
No identifiable patient properties


# Architecture
Phase 1: Production-lite
------------------------------------------------
User
  |
HTTPS
  |
Compute Engine VM
  |
OpenEMR Docker container
  |
Cloud SQL MySQL

AI Clinical Co-Pilot:
OpenEMR -> Cloud Run agent API -> LLM provider
                         |
                    eval/logging layer



## Deployment Decision

For the initial public deployment, this project will use a Compute Engine VM running the OpenEMR Docker Compose stack.

This is intentionally a Phase 1 deployment choice, not the final production architecture.

### Why this choice

The project deadline requires a working public deployment quickly. OpenEMR has already been proven locally using Docker Compose, so deploying the same containerized stack to a VM minimizes risk and preserves parity between local development and the public demo environment.

A more production-oriented Cloud SQL deployment was considered, but it adds additional setup risk around database networking, OpenEMR bootstrap configuration, secrets, and file persistence. Those are valuable production concerns, but they are not the highest-risk item for the immediate project milestone.

### Phase 1 Architecture

- Google Compute Engine VM
- Docker and Docker Compose
- OpenEMR container
- MariaDB container
- Persistent Docker volumes / persistent disk
- HTTPS reverse proxy
- Synthetic patient data only

### Phase 2 / Scale Architecture

If the project continues beyond the demo milestone, the target architecture is:

- GKE Autopilot for the OpenEMR application container
- Cloud SQL for MySQL
- Secret Manager for credentials and API keys
- managed file/document storage
- Google Cloud Load Balancer
- Cloud Logging and Monitoring
- AI Clinical Co-Pilot service deployed separately on Cloud Run or GKE

### Upgrade Path

The Phase 1 deployment is designed so durable state can be externalized later:

1. Export the MariaDB database from the Docker Compose deployment.
2. Import the database into Cloud SQL.
3. Move OpenEMR file/document storage to managed storage.
4. Move secrets out of `.env` and into Secret Manager.
5. Deploy the OpenEMR container to GKE.
6. Move DNS from the VM to the GKE load balancer.
7. Keep the AI agent as a separate service connected through a stable API boundary.

This provides a fast path to the project milestone while preserving a credible path to a scalable production architecture.