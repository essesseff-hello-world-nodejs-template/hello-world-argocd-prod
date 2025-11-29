# hello-world - Argo CD Application (PROD)

This repository contains the Argo CD Application manifest for the **PROD** environment of the hello-world essesseff app.

## Repository Structure

```
hello-world-argocd-prod/
├── app-of-apps.yaml                    # Root Application (apply this to Argo CD)
├── argocd/
│   └── hello-world-prod-application.yaml   # PROD environment Application manifest (auto-synced)
├── argocd-repository-secret.yaml        # Argo CD repository secrets (configure before applying)
└── README.md                           # This file
```

## Architecture

- **Deployment Model**: Trunk-based development (single `main` branch)
- **Manual Deploy**: Enabled (via essesseff UI with RBAC)
- **OTP Protection**: Required for PROD deployments
- **GitOps**: Managed by Argo CD with automated sync

## Quick Start

### Deploy to Argo CD

1. **Configure Argo CD repository access** (if not already done):
   ```bash
   # Edit argocd-repository-secret.yaml with your GitHub Argo CD machine username and token
   # ***BE SURE TO DELETE THE FILE AFTERWARDS SO AS NOT TO COMMIT THE FILE CONTENTS TO GITHUB***
   kubectl apply -f argocd-repository-secret.yaml
   ```
   
   This creates secrets for Argo CD to access:
   - `hello-world-argocd-prod` repository (to read Application manifests)
   - `hello-world-config-prod` repository (to read Helm charts and values)

2. **Apply the root Application (app-of-apps.yaml)**:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/essesseff-hello-world-go-template/hello-world-argocd-prod/main/app-of-apps.yaml
   ```
   
   This root Application watches the `argocd/` directory in this repository and automatically applies `hello-world-prod-application.yaml`. Once applied, any changes to `argocd/hello-world-prod-application.yaml` will be automatically synced by Argo CD. Changes to other files in the root directory (README.md, etc.) will not trigger re-syncs.

3. **Verify in Argo CD UI**:
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   # Access: https://localhost:8080
   ```
   
   You should see:
   - `hello-world-argocd-prod` - Root Application (watches this repository)
   - `hello-world-prod` - Environment Application (auto-synced by root Application)

## Application Details

- **Name**: `hello-world-prod`
- **Namespace**: `argocd`
- **Source Repository**: `hello-world-config-prod`
- **Destination Namespace**: `essesseff-hello-world-go-template`
- **Sync Policy**: Automated with prune and self-heal enabled

## Deployment Process

### Manual Deployment (Requires OTP)

1. **Release Engineer** deploys to PROD (requires OTP approval)
2. essesseff validates OTP and updates `hello-world-config-prod/values.yaml`
3. Argo CD syncs PROD Application automatically

## Repository URLs

- **Source**: `https://github.com/essesseff-hello-world-go-template/hello-world`
- **Config PROD**: `https://github.com/essesseff-hello-world-go-template/hello-world-config-prod`
- **Argo CD PROD**: `https://github.com/essesseff-hello-world-go-template/hello-world-argocd-prod` (this repo)

## essesseff Integration

This setup requires the essesseff platform for deployment orchestration:

- **RBAC enforcement**: Role-based access control for deployments
- **Approval workflows**: Manual approvals for PROD deployments
- **OTP protection**: One-time password required for PROD deployments
- **Deployment policies**: Enforced promotion paths (Stable → STAGING → PROD)
- **Audit trail**: Complete history of all deployments and approvals

## Argo CD Configuration

### Reduce Git Polling Interval (Optional)

By default, Argo CD polls Git repositories every ~3 minutes (120-180 seconds). To reduce this to 60 seconds for faster change detection:

```bash
kubectl patch configmap argocd-cm -n argocd --type merge -p '{"data":{"timeout.reconciliation":"60s","timeout.reconciliation.jitter":"10s"}}'
```

This will:
- Set base polling interval to 60 seconds
- Add up to 10 seconds of jitter (total: 60-70 seconds)
- Allow Argo CD to detect changes in `argocd/hello-world-prod-application.yaml` more quickly

**Note**: For even faster detection, consider configuring webhooks from GitHub to Argo CD for near-instant change detection.

## How It Works

1. **essesseff manages** image lifecycle and promotion decisions
2. **essesseff updates** `values.yaml` files in config repos (e.g., `hello-world-config-prod/values.yaml`) with approved image tags
3. **Argo CD detects** changes via Git polling (default: ~3 minutes, configurable to 60 seconds)
4. **Argo CD syncs** Application automatically (auto-sync enabled)
5. **Kubernetes resources** are updated with new image versions

## See Also

- [essesseff Documentation](https://essesseff.com/docs) - essesseff platform documentation
- [Argo CD Documentation](https://argo-cd.readthedocs.io/) - Argo CD documentation

