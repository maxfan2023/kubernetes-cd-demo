# Task 2: Kubernetes Configuration Validation

Validation date: 2026-10-28 (Asia/Shanghai).

## Configuration prepared

I adapted the four YAML files from the previous Kubernetes assignment and placed them under `k8s/`.

- Namespace: `training`.
- Deployment: `cd-demo-deployment`, with three replicas and container `cd-demo`.
- Application label and Service selector: `app: cd-demo`.
- Service: `cd-demo-service`, ClusterIP port 80, targeting the named container port `http`.
- Ingress: `cd-demo-ingress`, class `nginx`, host `cd-demo.local`, forwarding to `cd-demo-service:80`.
- Image: `crpi-uzkholcdutt858o7.cn-guangzhou.personal.cr.aliyuncs.com/maxfan/kubernetes-cd-demo:v1`.
- Private-registry Secret reference: `cd-demo-acr-pull`, in Namespace `training`.

I retained the previous assignment's rolling update strategy and HTTP readiness/liveness probes. Distinct resource names, labels, and the new Ingress host allow the previous exercises to coexist with this application. The Deployment and container names match the later Jenkins and rollback tasks.

## Target cluster

The existing `kind-k8s-chapter1` cluster was selected for verification because it already has a running Nginx Ingress Controller and an `nginx` IngressClass. Its Kubernetes node is Ready, runs Kubernetes v1.37.0, and uses the `linux/arm64` platform, matching the published image.

All commands explicitly select this context; the global default context remains unchanged. The currently selected `docker-desktop` cluster did not list an IngressClass during inspection.

## Server validation

Command:

```bash
kubectl --context=kind-k8s-chapter1 apply --dry-run=server -f k8s/ --request-timeout=15s
```

Actual result:

```text
deployment.apps/cd-demo-deployment created (server dry run)
ingress.networking.k8s.io/cd-demo-ingress created (server dry run)
namespace/training unchanged (server dry run)
service/cd-demo-service created (server dry run)
```

The Kubernetes API accepted all four resources. This was a dry run and did not deploy the application.

## Consistency checks

Additional checks passed for:

- The four resource types and consistent Namespace.
- Deployment/container names matching the later Jenkinsfile.
- Replica count, rolling update strategy, and retained rollout history defaults.
- Matching Deployment selectors, Pod labels, and Service selectors.
- Named HTTP ports and readiness/liveness probes.
- Ingress class, host, backend Service name, and Service port.
- The complete Aliyun image path and private-registry Secret reference.
- File whitespace, final newlines, and balanced README code fences.

No credential-bearing Secret manifest or kubeconfig was written to the repository.

## Runtime verification status

Actual deployment, image pulling by Kubernetes, Pod readiness, Service HTTP access, and Ingress HTTP access are pending provisioning of the private-registry Secret. The README contains commands to reproduce these checks after that prerequisite is satisfied.

These configuration checks do not replace the successful Jenkins build logs, v1/v2 deployment evidence, or rollout/rollback results required by Tasks 5 and 6.
