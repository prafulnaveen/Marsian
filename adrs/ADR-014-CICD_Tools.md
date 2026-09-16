# ADR-014 — Choice of Google Cloud Build for CI/CD Pipelines

## Context

The estate system comprises 30+ microservices in a GKE cluster requiring continuous deployment:

**Deployment Requirements:**
- **Multiple Services**: Ticketing (10 services), ride monitoring (4 services), animal health (4 services), plus infrastructure services
- **Multiple Environments**: Development, staging, production
- **High Frequency**: Multiple deployments per day (bug fixes, features, hotfixes)
- **Quality Gates**: Code review, unit tests, integration tests before deployment
- **Security Scanning**: Container image scanning, secret detection, vulnerability scanning
- **Rollback Capability**: Ability to quickly revert failed deployments

**Build Requirements:**
- Compile and test microservices (Go, Python, Node.js, Java)
- Build container images (Docker)
- Push to container registry (Google Container Registry / Artifact Registry)
- Run integration tests against test deployment
- Deploy to GKE via kubectl apply or Helm charts

**Pipeline Stages:**
1. **Trigger**: Git push to main branch → auto-trigger build
2. **Build**: Compile code, run unit tests
3. **Test**: Run integration tests, security scanning
4. **Build Image**: Create Docker image
5. **Registry**: Push to Artifact Registry
6. **Deploy Staging**: Deploy to staging GKE cluster
7. **Manual Approval**: Human review before production
8. **Deploy Production**: Deploy to production GKE cluster
9. **Verify**: Smoke tests post-deployment

**Scale Requirements:**
- 30+ builds/day
- Build time <10 minutes per service
- Support 5+ concurrent builds
- Infrastructure cost reasonable

**Integration Points:**
- **Git**: GitHub repository (source control)
- **GKE**: Deploy to Kubernetes cluster
- **Artifact Registry**: Store container images
- **Cloud KMS**: Decrypt secrets during build
- **BigQuery**: Store build metrics (build time, success rate)

## Decision

**We will use Google Cloud Build as the primary CI/CD platform.**

Cloud Build integrates natively with GCP services (GKE, Artifact Registry, Cloud KMS), supports multi-language builds, includes security scanning, and scales automatically—ideal for orchestrating 30+ microservice deployments.

## Key Differentiators

- **Native GCP Integration**  
  Cloud Build integrates directly with Cloud Source Repositories, GitHub, Artifact Registry, GKE, Cloud KMS, Cloud IAM. No third-party plugins needed. Authentication via Workload Identity.

- **Serverless Scaling**  
  Cloud Build automatically scales build workers based on queue. 50 concurrent builds during peak; scales down during low usage. No infrastructure to manage.

- **Container-Based Build Steps**  
  Build steps run as containers. Each step can use different Docker image (Go builder, Python, security scanner, kubectl). Flexible, reproducible build environments.

- **Native Kubernetes Deploy**  
  Cloud Build includes kubectl step; deploy directly to GKE with declarative YAML. No separate deployment tool needed.

- **Source Control Integration**  
  Automatic triggers on GitHub push, pull request, or tag. No webhooks to configure; native integration.

- **Build Configuration as Code**  
  cloudbuild.yaml checked into repository. Build configuration versioned with code; no separate "build server configuration."

- **Security Scanning**  
  Built-in vulnerability scanning (container images, dependencies). Detect vulnerabilities before deployment.

- **Secret Management**  
  Integrates with Cloud KMS for decrypting secrets during build. Secrets never stored as plaintext in build config.

- **Cost-Effective**  
  Pay per build minute (first 120 minutes/month free; then ~$0.003/minute). No fixed instance cost. Suitable for variable build frequency.

- **Deployment Strategies**  
  Supports canary deployments, blue-green deployments, rollback via Cloud Build's deployment manager. Manual approval gates before production.

- **Notification Integration**  
  Send build notifications to Slack, email, Pub/Sub. Instant feedback on build success/failure.

- **Artifact Versioning**  
  Automatic artifact tagging (git commit hash, git tag, branch name). Traceability between code version and deployed container image.

## Alternatives Considered

- **Jenkins**  
  Open-source CI/CD server:
  - **Self-Hosted Complexity**: Requires Jenkins server deployment, plugin management, backup/recovery. Operational burden.
  - **Scaling Complexity**: Adding build capacity requires managing Jenkins agents/nodes. Cloud Build auto-scales.
  - **Infrastructure Cost**: Jenkins server and agents cost money; must size for peak capacity. Cloud Build pay-per-use.
  - **Security Management**: Jenkins plugins frequently have vulnerabilities; patching and testing required. Cloud Build managed by Google.
  - **Not Recommended**: Jenkins viable for on-premises; cloud-native systems should use managed CI/CD (Cloud Build).

- **GitLab CI/CD**  
  GitLab's built-in CI/CD:
  - **Platform Lock-in**: Requires using GitLab as Git host. Repository in GitHub doesn't work (though GitLab can mirror). Not suitable if GitHub is source control.
  - **Self-Hosted Version**: GitLab CI runners require self-hosting (same as Jenkins); managed SaaS available but vendor lock-in.
  - **Not Aligned**: Estate system on GitHub + GCP; GitLab CI adds multi-platform complexity.

- **GitHub Actions**  
  GitHub's built-in CI/CD:
  - **Source Control Integration**: Native to GitHub; no setup needed. Very convenient for GitHub repos.
  - **GCP Integration Limited**: GitHub Actions integrates with GCP via plugin; not as native as Cloud Build. Authentication more complex.
  - **Comparison to Cloud Build**: Both excellent; GitHub Actions better if GitHub is primary platform. Cloud Build better if GCP services central.
  - **Recommendation**: For GitHub repos, GitHub Actions reasonable choice. For tight GCP integration, Cloud Build slightly better.

- **CircleCI**  
  SaaS CI/CD platform:
  - **Good Alternative**: CircleCI excellent CI/CD platform; supports GitHub, Bitbucket, etc.
  - **GCP Integration**: Integrates with GCP but via plugin; not as native as Cloud Build.
  - **Cost**: CircleCI pricing similar to Cloud Build but less flexible.
  - **Ecosystem**: Both viable; Cloud Build slightly better for GCP-centric projects.

- **Travis CI**  
  CI/CD platform (now owned by Idera):
  - **Legacy Status**: Travis CI was popular 10 years ago; modern teams prefer GitHub Actions, CircleCI, or Cloud Build.
  - **Funding Issues**: Travis CI had financial difficulties; platform stability uncertain.
  - **Not Recommended**: Cloud Build, GitHub Actions, or CircleCI are better choices.

- **Spinnaker**  
  Open-source deployment/CD platform:
  - **Deployment Focus**: Spinnaker specializes in deployment (blue-green, canary) after build. Doesn't replace build step (CI).
  - **Setup Complexity**: Spinnaker requires significant setup and expertise. Not suitable for MVP.
  - **Recommendation**: Use Cloud Build for CI/CD; add Spinnaker later if advanced deployment strategies needed.

- **ArgoCD**  
  GitOps continuous delivery tool:
  - **GitOps Model**: ArgoCD pulls deployment manifest changes from Git. Different model than Cloud Build's push-based.
  - **Complementary**: ArgoCD can work alongside Cloud Build (Cloud Build builds images; ArgoCD deploys based on Git config).
  - **Not a Complete Solution**: ArgoCD doesn't replace build step; would need Cloud Build anyway.

- **Tekton**  
  Open-source CI/CD framework:
  - **Similar to Cloud Build**: Tekton provides CI/CD pipelines; can run on Kubernetes.
  - **Self-Hosted**: Tekton requires Kubernetes cluster management; Cloud Build is serverless.
  - **Smaller Ecosystem**: Cloud Build has larger community and more GCP integrations.
  - **Not Recommended**: Cloud Build simpler for this use case.

## Why Cloud Build is Better Than Alternatives

| Criterion | Cloud Build | Jenkins | GitHub Actions | CircleCI | Tekton |
|-----------|-------------|---------|-----------------|----------|--------|
| **GCP Integration** | Native | Via plugin | Via plugin | Via connector | Native (K8s) |
| **Serverless** | Yes | No | Yes | Yes | No (requires K8s) |
| **Cost Model** | Pay-per-minute | Fixed (infrastructure) | Pay-per-minute | Pay-per-minute | Fixed (infrastructure) |
| **Setup Complexity** | Low (hours) | High (days) | Low (GitHub users) | Low (hours) | Medium (Kubernetes) |
| **GitHub Integration** | Excellent | Via plugin | Native | Native | Via plugin |
| **Artifact Registry Integration** | Native | Via plugin | Via plugin | Via connector | Native |
| **GKE Deployment** | Native (kubectl) | Via plugin | Via plugin | Via connector | Native |
| **Security Scanning** | Built-in | Via plugin | Via plugin | Via plugin | Via plugin |
| **Scaling** | Automatic | Manual | Automatic | Automatic | Manual |
| **Multi-Language Support** | Excellent | Excellent | Excellent | Excellent | Excellent |
| **Deployment Strategies** | Good (blue-green, canary) | Via plugin | Via Actions | Good | Good |
| **Cost (50 builds/day, 10 min avg)** | ~$9/day | $200+/mo (infra) | ~$5/day | ~$15/day | $200+/mo (infra) |

**Why Cloud Build wins for estate system:**

1. **Native GCP Integration**: Artifact Registry, GKE, Cloud KMS, Cloud IAM all built-in. No plugin configuration.

2. **Serverless Scaling**: Automatic scaling from 0 to 100 concurrent builds. No infrastructure management.

3. **Cost-Efficient**: Pay-per-minute means variable builds don't waste money. Fixed cost inappropriate for 30 services with variable deployment frequency.

4. **Container-Based Steps**: Each build step runs in container with chosen image. Flexible, reproducible environments.

5. **Security Scanning**: Built-in vulnerability scanning prevents deploying vulnerable images.

6. **GitHub Integration**: Excellent support for GitHub (pushes, pull requests, tags as triggers).

7. **Deployment Automation**: kubectl and Helm support enable fully automated deployments without separate tool.

## Risks & Trade-offs

| Risk Area | Description | Mitigation |
|-----------|-------------|-----------|
| **Build Time Variability** | Build times vary (cache misses, resource contention); slow builds might exceed SLA | Implement caching (Docker layer caching, dependency caching), measure build time trends, investigate slow builds, optimize dependencies |
| **Concurrent Build Limits** | Cloud Build has soft quotas on concurrent builds; excessive builds might queue | Monitor concurrent build count, request quota increase if needed, implement build prioritization (critical vs. non-critical builds) |
| **Secret Exposure Risk** | Secrets decrypted during build; could leak in logs or artifacts if misconfigured | Never log secrets, use Cloud KMS with fine-grained permissions, scan build logs for accidental secret exposure, implement secret rotation |
| **Deployment Failures** | Failed deployments (bad manifest, resource conflicts, quota exceeded) could leave cluster in bad state | Implement automated rollback on deploy failure, use health checks post-deploy, implement smoke tests, practice deployments in staging |
| **Image Registry Quota** | Pushing too many images (one per build) could exceed registry storage quota | Implement image pruning (delete old images), tag images by git commit + branch (avoid duplicates), monitor registry size |
| **Build Configuration Drift** | cloudbuild.yaml drifts from actual build process; changes made via UI instead of code | Enforce code review for cloudbuild.yaml changes, disable UI configuration (code-only), document build process in README |
| **Cost Uncertainty** | If build times increase or frequency spikes, costs could exceed budget | Monitor build minutes via Cloud Build dashboards, set alerts on estimated cost, optimize slow builds, review build frequency |
| **Dependency Updates** | New dependency versions could introduce breaking changes; builds fail if dependencies not pinned | Pin dependency versions (Docker image versions, npm packages, Python packages), test dependency updates in staging, implement automated security updates for critical packages |
| **Artifact Cleanup** | Old container images accumulate in registry; storage costs grow | Implement image retention policy (keep last 10 images, delete older), archive old images to Cloud Storage, automate cleanup |
| **Multi-Service Coordination** | If 30 services deployed frequently, coordinating deployments complex (dependency ordering, compatibility) | Implement service versioning strategy, use semantic versioning for APIs, test multi-service deployments in staging, implement parallel deployments where possible |

## Conclusion

Cloud Build is the optimal CI/CD platform for the estate system because:

1. **Native GCP Integration**: Seamless integration with Artifact Registry, GKE, Cloud KMS eliminates plugin configuration.

2. **Serverless Autoscaling**: Automatic scaling from 0-100 concurrent builds. No infrastructure management or capacity planning.

3. **Cost-Efficient**: Pay per build minute (~$0.003/min). 50 builds/day × 10 min = $1.50/day. Jenkins infrastructure would cost $200+/month.

4. **Container-Based Build Steps**: Each step runs in Docker container. Reproducible, flexible build environments without tool dependencies.

5. **Security Built-In**: Vulnerability scanning before deploy, secret management via Cloud KMS, audit logging.

6. **Fast Setup**: Native GitHub integration, no plugins, cloudbuild.yaml as code. Operational within hours.

7. **Deployment Automation**: kubectl and Helm support enable fully automated deployments. Blue-green/canary via manual approval gates.

**Recommendation**:
- Store cloudbuild.yaml in each service repo; define build pipeline as code
- Implement build stages: compile → unit test → build image → security scan → deploy staging → manual approval → deploy production
- Use Docker layer caching to speed builds
- Tag images by git commit hash + branch (e.g., `ride-service:main-abc123`)
- Store secrets in Cloud KMS; decrypt during build only
- Implement automated rollback on deploy failure via health checks
- Send build notifications to Slack (#deployments channel)
- Monitor build times; optimize slow builds
- Use Artifact Registry for container image storage
- Implement image retention policy (keep last 10 images per service)
- Test full CI/CD pipeline with staging deployments before going production
- Document build process in service README
- Rotate secrets monthly; update Cloud KMS keys
