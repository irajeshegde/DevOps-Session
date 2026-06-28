# DevOps Session Demo

This repository contains a demonstration project for DevOps workflows.

Contents:

1. [db-service/](db-service/) — FastAPI + PostgreSQL microservice with a Dockerfile and docker-compose for local testing
2. [infra/](infra/) — Bicep template provisioning Azure Container Registry, Container Apps environment, and Postgres Flexible Server
3. [.github/workflows/](.github/workflows/) — GitHub Actions pipeline (`infra.yml` to provision Azure infra, `deploy.yml` to build and deploy the app)
4. [infra/DEPLOYMENT_STEPS.md](infra/DEPLOYMENT_STEPS.md) — step-by-step guide to deploy this to Azure

Use this repo to explore Dockerfile creation, Postgres-backed microservices, and CI/CD to Azure Container Apps.

## Prompts used to generate this repo

This repo was built with Claude Code using the following prompts, in order:

1. *"Generate a database microservice with dockerfile in it which is ready to test on my system. I want to use postgres"* — produced `db-service/` (FastAPI app, Dockerfile, docker-compose.yml with a Postgres container, requirements.txt).
2. *"I want to deploy this on Azure, build a CI/CD pipeline in a folder for me and steps to implement it"* — followed by clarifying answers (Azure Container Apps, GitHub Actions, Azure Database for PostgreSQL Flexible Server) — produced `infra/main.bicep`, `.github/workflows/infra.yml`, `.github/workflows/deploy.yml`, and `infra/DEPLOYMENT_STEPS.md`.
3. *"Update this file and put an effective prompt that was used to generate the files, and put token usage and history of Claude Code"* — produced this section, [Session history](#session-history), and [Token usage](#token-usage) below.

An effective reusable version of the combined ask:

> Generate a FastAPI microservice backed by PostgreSQL, with a Dockerfile and docker-compose.yml ready to run locally. Then add a CI/CD pipeline to deploy it to Azure Container Apps using GitHub Actions and OIDC login, with the database hosted on Azure Database for PostgreSQL Flexible Server, provisioned via a Bicep template. Include a step-by-step deployment guide covering Azure AD federated credentials, required GitHub secrets/variables, and how to verify the deployment.

## Session history

| # | Request | Outcome |
|---|---------|---------|
| 1 | Generate a Postgres-backed microservice with a Dockerfile, ready to test locally | Created `db-service/` (FastAPI app under `app/`, `Dockerfile`, `docker-compose.yml`, `requirements.txt`, `.dockerignore`, `README.md`) |
| 2 | Deploy it on Azure with a CI/CD pipeline in a folder, plus implementation steps | Clarified target stack via questions (Container Apps / GitHub Actions / Postgres Flexible Server), then created `infra/main.bicep`, `.github/workflows/infra.yml`, `.github/workflows/deploy.yml`, `infra/DEPLOYMENT_STEPS.md` |
| 3 | Document the prompts, token usage, and history in the README | Updated this file |

Note: this directory was not yet a git repository as of this session, so no commits/history exist beyond this file list — see `infra/DEPLOYMENT_STEPS.md` step 0 for `git init` instructions.