# ADR 0000 – GitHub is the Source of Truth

| Status | Context Date | Deciders |
|--------|--------------|----------|
| ✅ Accepted | 2025-06-28 | JP Dow, DMA Core Team |

## Context
The team needs one canonical location for **code, docs, issues, ADRs, CI/CD, and release artefacts**.  
Options considered: GitHub, GitLab Self-Hosted, and Azure DevOps. GitHub already hosts the public DMA repo and supports all required workflows.

## Decision
* The **Dungeon-Master-s-Assistant** GitHub **organization and mono-repo `dma`** are the single source-of-truth.  
* All ADRs live under `/adr/`, docs under `/docs/`, code under their respective folders.  
* All feature work must land via Pull Request with green CI on **`main`**.

## Consequences
* CI/CD, GitHub Actions, and GitHub Pages are first-class tooling.  
* Any external wiki or mirror repo must explicitly document itself as *secondary*.  
* Future migration would require a new ADR.
