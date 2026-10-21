# Centralized CI Pipelines

Repo CI tập trung cho toàn bộ Delivery Platform. Mỗi source repo (backend, web, mobile) gọi reusable workflow từ đây để build và push Docker image lên Google Artifact Registry.

## Kiến trúc

```
┌─────────────────┐     ┌──────────────────────────────────┐     ┌─────────────┐
│  Source Repos    │────▶│  cicd.centralized-pipelines (CI)  │────▶│  Artifact   │
│  (backend, web, │     │  Reusable Workflows               │     │  Registry   │
│   mobile)        │     │  - build-java.yml                 │     │  (GAR)      │
│                  │     │  - build-node.yml                 │     └──────┬──────┘
│                  │     │  - build-flutter.yml              │            │
│                  │     │  - build-react-native.yml         │            │ poll
└─────────────────┘     └──────────────────────────────────┘     ┌──────▼──────┐
                                                                  │  ArgoCD     │
                                                                  │  Image      │
                                                                  │  Updater    │
                                                                  └──────┬──────┘
                                                                         │ commit
                                                                  ┌──────▼──────┐
                                                                  │  gitops-cd  │
                                                                  │  Helm Values│
                                                                  └──────┬──────┘
                                                                         │ sync
                                                                  ┌──────▼──────┐
                                                                  │  ArgoCD     │
                                                                  │  → GKE      │
                                                                  └─────────────┘
```

## Reusable Workflows

| Workflow | Stack | Dùng cho |
|---|---|---|
| `build-java.yml` | Java 17 + Maven | Backend microservices (Spring Boot) |
| `build-node.yml` | Node 22 + Docker | Web apps (React/Vite) |
| `build-flutter.yml` | Flutter + Docker | Mobile app khách hàng |
| `build-react-native.yml` | React Native | Shipper app |

## Cách sử dụng từ Source Repo

Tại source repo, tạo workflow gọi reusable workflow:

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main, staging]

jobs:
  build:
    uses: hoanganhnth/cicd.centralized-pipelines/.github/workflows/build-java.yml@main
    with:
      service_name: api-gateway
      java_version: "17"
    secrets:
      GCP_SA_KEY: ${{ secrets.GCP_SA_KEY }}
```

## Secrets cần thiết (trên từng source repo)

| Secret | Mô tả |
|---|---|
| `GCP_SA_KEY` | Service Account JSON key có quyền `artifactregistry.writer` |
