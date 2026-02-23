# Infrastructure-as-Code & Deployment Rules

> **Audience:** You (the AI assistant). These are binding rules for generating Dockerfiles, CI/CD pipelines, Kubernetes manifests, Terraform, and other IaC configurations.

## 1. Container Permissions

- **PROHIBITED:** Never run Docker containers as the `root` user. Never blindly execute `chmod 777` on directories.
- **INSTEAD:** Always explicitly define a non-root user in Dockerfiles (e.g., `USER node`, `USER appuser`) and set the most restrictive file permissions necessary.

## 2. Minimal Base Images

- **PROHIBITED:** Never use bloated OS images (like `ubuntu:latest` or `node:bullseye`) for production runtime environments.
- **INSTEAD:** Prefer minimal, hardened base images such as Alpine, Distroless, or slim variants (e.g., `node:18-alpine`).

## 3. Secrets in CI/CD Environments

- **PROHIBITED:** Never commit cloud credentials (e.g., AWS Access Keys), database passwords, or private keys into `.env` files, Dockerfiles, or unencrypted CI pipeline configurations.
- **INSTEAD:** Always use your CI provider's secure secrets manager (e.g., GitHub Secrets, AWS Parameter Store) and securely inject them at runtime or pipeline execution time.

## 4. IAM & Cloud Permissions (Terraform/AWS/GCP)

- **PROHIBITED:** Never generate IaC (like Terraform or CloudFormation) that provisions resources with wildcards (e.g., `Action: "*"`, `Resource: "*"`), or opens databases/storage to `0.0.0.0/0`.
- **INSTEAD:** Always follow the Principle of Least Privilege. Specify exact, required actions and constrain networks to internal VPCs or explicitly authorized IP ranges.
