# Kubernetes Continuous Deployment Demo

Kubernetes Day 2 coursework: build an Nginx application image, publish it to Aliyun Container Registry, and deploy it through a Jenkins pipeline. Tasks 1-5 have been completed. Task 6, rollback, remains pending.

The source page is **v2**. On 2026-10-28, the verified v1 commit was deployed again for screenshots: Jenkins build **#4 SUCCESS**, Deployment revision **6**, and **3/3 Ready**. The job has been restored to `main`; its next build will publish the current v2 source.

## Project files

| File | Purpose |
| --- | --- |
| [app/index.html](app/index.html) | Application page and visible version. |
| [app/Dockerfile](app/Dockerfile) | Copies the page into `nginx:1.25-alpine` and exposes port 80. |
| [k8s/namespace.yaml](k8s/namespace.yaml) | Training namespace. |
| [k8s/deployment.yaml](k8s/deployment.yaml) | Three replicas, rolling updates, HTTP probes, and private image-pull configuration. |
| [k8s/service.yaml](k8s/service.yaml) | ClusterIP Service on port 80. |
| [k8s/ingress.yaml](k8s/ingress.yaml) | Nginx Ingress route for `cd-demo.local`. |
| [Jenkinsfile](Jenkinsfile) | GitHub SCM release pipeline. |
| [Jenkinsfile.script-test](Jenkinsfile.script-test) | Local-source version of the pipeline for the independent Script test job. |

Build logs, reports, and runtime snapshots are retained locally in `docs/`. That directory is excluded from Git. Local IDE files, `main.py`, environment files, and kubeconfig files are also excluded.

## Build and run locally

Start Docker Desktop, then run these commands from the parent of the cloned repository:

```bash
cd kubernetes-cd-demo
docker build -t kubernetes-cd-demo:v2 ./app
docker run --rm -d --name kubernetes-cd-demo \
  -p 127.0.0.1:18080:80 kubernetes-cd-demo:v2
curl -fsS http://127.0.0.1:18080/
```

The current source returns:

```text
Kubernetes Continuous Deployment Demo
Application version: v2
```

`EXPOSE 80` declares the container's HTTP port. The `-p` option makes it accessible through the local loopback address. Stop the test container when finished:

```bash
docker stop kubernetes-cd-demo
```

## Kubernetes configuration

This setup uses the existing course kind cluster and its Nginx Ingress Controller. Commands select the context explicitly and preserve the global kubectl context.

| Setting | Value |
| --- | --- |
| Context | `kind-k8s-chapter1` |
| Namespace | `training` |
| Deployment | `cd-demo-deployment` |
| Container | `cd-demo` |
| Pod label and Service selector | `app: cd-demo` |
| Replicas | `3` |
| Service | `cd-demo-service`, ClusterIP, port 80 |
| Ingress | `cd-demo-ingress`, class `nginx` |
| Ingress host | `cd-demo.local` |
| Image platform | `linux/arm64` |
| Image-pull Secret | `cd-demo-acr-pull` |

The dedicated `app: cd-demo` selector keeps this Service separate from the earlier Nginx exercises.

### Private registry access

```text
REGISTRY_HOST=crpi-uzkholcdutt858o7.cn-guangzhou.personal.cr.aliyuncs.com
IMAGE_REPOSITORY=crpi-uzkholcdutt858o7.cn-guangzhou.personal.cr.aliyuncs.com/maxfan/kubernetes-cd-demo
```

The Deployment references `training/cd-demo-acr-pull` through `imagePullSecrets`. Provision this Secret securely before deploying to another cluster or namespace. Jenkins registry credentials authorize image pushes; Kubernetes needs its own image-pull credentials. Keep passwords, tokens, kubeconfig contents, and Secret values outside Git.

### Initial v1 deployment

The source Deployment YAML retains the initial `:v1` image. Applying it directly deploys that image. To publish the current source using its Git commit tag, run the Jenkins SCM job described below.

From the repository root, after provisioning the image-pull Secret:

```bash
kubectl --context=kind-k8s-chapter1 apply -f k8s/namespace.yaml
kubectl --context=kind-k8s-chapter1 -n training get secret cd-demo-acr-pull
kubectl --context=kind-k8s-chapter1 apply --dry-run=server -f k8s/
kubectl --context=kind-k8s-chapter1 apply -f k8s/
kubectl --context=kind-k8s-chapter1 -n training rollout status \
  deployment/cd-demo-deployment --timeout=180s
```

The server dry-run validates resources. Image pulls, readiness, and HTTP responses are checked after deployment.

To inspect the currently running version:

```bash
kubectl --context=kind-k8s-chapter1 -n training get \
  deployments,pods,replicasets,services,ingresses -l app=cd-demo -o wide
kubectl --context=kind-k8s-chapter1 -n training get endpointslices \
  -l kubernetes.io/service-name=cd-demo-service
kubectl --context=kind-k8s-chapter1 -n training rollout history \
  deployment/cd-demo-deployment
```

### Verify HTTP through the Service

Keep the port-forward running in one terminal:

```bash
kubectl --context=kind-k8s-chapter1 -n training port-forward \
  --address 127.0.0.1 service/cd-demo-service 18081:80
```

In another terminal, request the page or open [the local Service URL](http://127.0.0.1:18081/):

```bash
curl -fsS http://127.0.0.1:18081/
```

Check the visible version against the intended release. The screenshot deployment described above returns `Application version: v1`. Stop the forward with `Ctrl+C` when finished.

### Verify HTTP through Ingress

An Ingress resource requires a running Controller and a reachable entry point. The following temporary forward uses the existing Controller Service:

```bash
kubectl --context=kind-k8s-chapter1 -n ingress-nginx port-forward \
  --address 127.0.0.1 service/ingress-nginx-controller 18082:80
```

In another terminal, use the Host header to match the Ingress rule without changing local DNS:

```bash
curl -fsS -H 'Host: cd-demo.local' http://127.0.0.1:18082/
```

The response should match the Service response. Stop the forward with `Ctrl+C` when finished.

## Jenkins pipeline

The [SCM job](http://localhost:8080/job/kubernetes-cd-demo/) uses the root `Jenkinsfile` and the local Jenkins `built-in` node. That node needs Git, Docker Desktop, Buildx, and kubectl. The pipeline adds the Homebrew and local binary directories to `PATH`.

| Stage | Action |
| --- | --- |
| Checkout | Check out SCM and use the first 12 characters of the Git commit as the image tag. |
| Validate | Check the required files, version marker, and command-line tools. |
| Build and Push Image | Build a `linux/arm64` image and push it to Aliyun. |
| Deploy to Kubernetes | Apply the resources with the final commit-tagged image and wait for rollout completion. |
| Verify | Check the deployed image and report resources, endpoints, and rollout history. |

The pipeline enables timestamps, prevents concurrent builds, and applies a 20-minute timeout. Builds disable provenance and SBOM attestations, and pushes select `linux/arm64` for the registry configuration used here. Registry login uses a temporary Docker configuration that is removed afterward.

Before applying the Deployment, `kubectl set image --local` renders a temporary manifest with the final image tag. This produces a single Deployment update per release.

### Job and credentials

| Jenkins setting | Value |
| --- | --- |
| Definition | Pipeline script from SCM |
| SCM | Git |
| Repository | `https://github.com/maxfan2023/kubernetes-cd-demo.git` |
| Git credential | `github-token` |
| Branch specifier | `*/main` |
| Script path | `Jenkinsfile` |
| Lightweight checkout | Enabled |

| Credential ID | Type | Purpose |
| --- | --- | --- |
| `container-registry-credentials` | Username with password | Authenticate to Aliyun for image pushes. |
| `kubeconfig-training` | Secret file | Access `kind-k8s-chapter1`. |

The selected kubeconfig retains its existing Kubernetes identity and permissions. Its credential name does not limit that identity to the `training` namespace.

### Local Pipeline script test

The independent [Script test job](http://localhost:8080/job/kubernetes-cd-demo-script-test/) uses `Jenkinsfile.script-test`. Its Checkout stage copies local application files and creates a snapshot commit in the test workspace. The remaining stages build, push, deploy, and verify the application. Set its `source_dir` to the application checkout on the Jenkins host before using it elsewhere.

Script build [#1](http://localhost:8080/job/kubernetes-cd-demo-script-test/1/) succeeded with v1, three ready replicas, and matching Service and Ingress pages. That run used local source and did not upload source to GitHub.

## Release history

Task 5 published v1 through the SCM job, changed the page to v2, pushed the new commit, and ran the pipeline again. Image tags identify source commits; v1 and v2 identify page content.

| Page | Source commit / image tag | Jenkins build | Deployment revision | Result |
| --- | --- | --- | --- | --- |
| v1 | `1ff38fb17176` | [#1](http://localhost:8080/job/kubernetes-cd-demo/1/) | 4 | SUCCESS, 3/3 Ready, HTTP 200 |
| v2 | `64f2a5d31ea5` | [#2](http://localhost:8080/job/kubernetes-cd-demo/2/) | 5 | SUCCESS, 3/3 Ready, HTTP 200 |
| v1 screenshot run | `1ff38fb17176` | [#4](http://localhost:8080/job/kubernetes-cd-demo/4/) | 6 | SUCCESS, 3/3 Ready, HTTP 200 |

Each successful SCM release was checked against the exact source commit, pushed image digest, all three Pods, ready Service endpoints, and full-page Service and Ingress responses. Earlier Script tests account for the revision numbers preceding the formal releases. The v1 screenshot run was pinned to its original commit; the job was then restored to `main` with lightweight checkout enabled.

To publish a new page version, commit only the intended source change, push it, and run the SCM job:

```bash
git add app/index.html
git commit -m "Make the updated application version visible"
git push origin main
```

For a one-time build pinned to a commit SHA, use full SCM checkout. The screenshot run initially failed before deployment because lightweight checkout treated the SHA as a branch name; disabling lightweight checkout allowed the pinned release to succeed.

Task 6 remains pending. It requires a separate rollback operation and verification of the restored page version.
