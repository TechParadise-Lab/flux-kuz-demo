# AKS Image Automation with Flux and Kustomize

This guide is a simple, step-by-step exercise for learning image automation in AKS using your existing Flux/Kustomize lab layout.

## 1. Why image automation?

Flux image automation turns a registry tag update into a Git commit and then a deployment update in the cluster. The flow is:

1. a new image tag appears in the registry
2. Flux sees the new tag via `ImagePolicy`
3. Flux updates the Git manifest files via `ImageUpdateAutomation`
4. Flux reconciles the cluster with the new image from Git

That means the cluster is always driven by Git, including image updates.

## 2. What this repo already has

Your current Kustomize layout is:

```
kustomize-lab/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    └── prod/
        ├── kustomization.yaml
        └── replica-patch.yaml
```

This is a great starting point. Image automation can update the image in `base/deployment.yaml` or via Kustomize `images:` in `base/kustomization.yaml`.

## 3. Prerequisites

- AKS cluster already created
- `kubectl` installed and connected to AKS
- `flux` CLI installed
- GitHub repo containing this code
- A container registry and image tags you can push to (Azure Container Registry, Docker Hub, GitHub Container Registry, etc.)
- Flux bootstrapped to your GitHub repo and AKS cluster

## 4. Prepare your app manifest for automation

### Option A: update the Deployment image directly

In `kustomize-lab/base/deployment.yaml`, change the image to a registry image you control.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: myregistry.azurecr.io/web:1.0.0
          ports:
            - containerPort: 80
```

This makes the image easily discoverable for Flux.

### Option B: use Kustomize image replacement

A more declarative pattern is to define the image in `base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
images:
  - name: myregistry.azurecr.io/web
    newTag: 1.0.0
```

Then your deployment image can remain the same or use the fully qualified name. Flux will update the `newTag` field in the kustomization file.

> Tip: using `images:` is helpful when the same image is referenced in multiple resources or when you want the image update to remain isolated to the Kustomization file.

## 5. Bootstrap Flux if you haven't already

If Flux is not installed yet, bootstrap it to point to a cluster path such as `clusters/aks`.

```bash
flux bootstrap github \
  --owner=<github-user-or-org> \
  --repository=<repo-name> \
  --branch=main \
  --path=./clusters/aks \
  --personal \
  --token-auth
```

This creates Flux control-plane resources in the cluster and a `clusters/aks/flux-system` path in Git.

## 6. Add image automation resources

Create three Flux resources in your Git repo under `clusters/aks` or `clusters/aks/flux-system`:

1. `ImageRepository` — points to the container registry
2. `ImagePolicy` — selects which tag is newest
3. `ImageUpdateAutomation` — writes the manifest update back to Git

### Example: `clusters/aks/image-repository.yaml`

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageRepository
metadata:
  name: web-repo
  namespace: flux-system
spec:
  interval: 1m
  image: myregistry.azurecr.io/web
  secretRef:
    name: registry-credentials  # optional for private registries
```

### Example: `clusters/aks/image-policy.yaml`

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImagePolicy
metadata:
  name: web-policy
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: web-repo
  policy:
    semver:
      range: ">= 1.0.0"
```

### Example: `clusters/aks/image-update-automation.yaml`

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: web-image-automation
  namespace: flux-system
spec:
  checkout:
    gitRepositoryRef:
      name: flux-system
  interval: 1m
  update:
    strategy: Setters
  commit:
    authorName: fluxcdbot
    authorEmail: fluxcdbot@example.com
  git:
    push:
      branch: main
  # Make sure this path points to the directory containing the Kustomize manifests you want to update.
  path: ./kustomize-lab/base
  # Alternative: ./kustomize-lab if you want the automation to update higher-level files
```

> Note: `strategy: Setters` is the default for Flux 2 image automation and works with plain YAML/JSON updates. If you use Kustomize `images:` fields in `kustomization.yaml`, Flux will update those fields directly.

## 7. Connect image automation to the right Git path

Flux will commit changes to the Git repository path you specify in `ImageUpdateAutomation.spec.path`.

For your repo, the likely path is:

- `./kustomize-lab/base` if you want to update the Deployment manifest or `base/kustomization.yaml`
- `./kustomize-lab` if your image is defined at a higher level

If you want dev and prod to share the same image update, update the base manifest or the base kustomization.

## 8. Push a new image tag and watch the flow

1. Build and push a new image tag to your registry:

```bash
docker build -t myregistry.azurecr.io/web:1.0.1 .
docker push myregistry.azurecr.io/web:1.0.1
```

2. Flux detects the new tag via the `ImageRepository` and `ImagePolicy`.
3. Flux updates your Git manifest file automatically.
4. Flux reconciles the cluster and deploys the new image.

## 9. Check the status

Use these commands to inspect Flux image automation status:

```bash
flux get image repository -n flux-system
flux get image policy -n flux-system
flux get image update automation -n flux-system
flux get kustomization -n flux-system
```

Check the updated Git commit in your repo to see exactly what changed. The commit should update either:

- `kustomize-lab/base/deployment.yaml`
- or `kustomize-lab/base/kustomization.yaml`

Then check the cluster:

```bash
kubectl get deploy -n dev
kubectl get deploy -n prod
kubectl describe deployment web -n dev
kubectl describe deployment web -n prod
```

## 10. How this fits your current Kustomize overlays

- `base/` remains the canonical application definition.
- `overlays/dev` and `overlays/prod` contain environment-specific metadata.
- Image automation can update the shared base image, which then flows into both overlays.
- `prod` still uses `replica-patch.yaml` to change replica count only.

This keeps your GitOps model clean: image changes are handled in the base app definition, while environment differences stay in overlays.

## 11. Example GitOps path layout

A good repo layout is:

```
clusters/
└── aks/
    ├── flux-system/
    ├── image-repository.yaml
    ├── image-policy.yaml
    ├── image-update-automation.yaml
    ├── dev-kustomization.yaml
    └── prod-kustomization.yaml
kustomize-lab/
├── base/
└── overlays/
```

`dev-kustomization.yaml` and `prod-kustomization.yaml` are the Flux `Kustomization` objects that tell Flux which overlay to apply.

## 12. Quick concept summary

- `ImageRepository` watches the registry.
- `ImagePolicy` decides which tag is "latest" by your rules.
- `ImageUpdateAutomation` edits Git and pushes the updated manifest.
- Flux reconciles the updated manifest and deploys the new image to AKS.

## 13. Learning check

After completing this exercise, you should understand:

- how Flux tracks container images
- how Flux updates Git manifests automatically
- why image automation is a GitOps pattern, not a direct `kubectl set image`
- how Kustomize overlays keep environment differences separate from image updates

---

Use this file as your step-by-step exercise guide while you explore Flux image automation with the `kustomize-lab` setup.
