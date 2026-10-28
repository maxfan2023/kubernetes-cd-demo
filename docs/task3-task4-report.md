# Task 3 and Task 4: Jenkins Pipeline and Configuration

Date: 2026-10-28 (Asia/Shanghai).

> This report records the initial configuration-only checks. A subsequent authorized Pipeline script run completed successfully, including image push, a 3/3 rollout and HTTP verification. The pull Secret has now been configured. See [the full execution report](task3-script-test.md) for the current result.

## Task 3: Create the Jenkinsfile

I prepared a Declarative `Jenkinsfile` in the project root. It uses the existing local Jenkins built-in node, which can run Git, Docker Desktop, Docker Buildx, and kubectl. The pipeline adds `/opt/homebrew/bin` and `/usr/local/bin` to its PATH because Jenkins runs as a macOS background service.

The pipeline uses these project settings:

| Setting | Value |
| --- | --- |
| Registry host | `crpi-uzkholcdutt858o7.cn-guangzhou.personal.cr.aliyuncs.com` |
| Image repository | Registry host followed by `/maxfan/kubernetes-cd-demo` |
| Image platform | `linux/arm64` |
| Kubernetes context | `kind-k8s-chapter1` |
| Namespace | `training` |
| Deployment | `cd-demo-deployment` |
| Container | `cd-demo` |

The pipeline has five stages:

1. **Checkout:** Check out the configured Git repository and derive a 12-character image tag from the current Git commit. Automatic checkout is disabled so that the source is checked out once in this explicit stage.
2. **Validate:** Check that the application and four Kubernetes YAML files exist, confirm the page contains the application version, and check the required command-line tools.
3. **Build and Push Image:** Build the application image, authenticate using Jenkins Credentials, and push the commit-tagged image to the private Aliyun repository.
4. **Deploy to Kubernetes:** Bind the kubeconfig secret file, apply the Namespace, check the private-image pull Secret, apply the final manifests, and wait for the Deployment rollout.
5. **Verify:** Confirm that the Deployment references the expected image and display application resources, Service EndpointSlices, and rollout history.

Timestamps, a 20-minute timeout, and `disableConcurrentBuilds()` make the execution easier to investigate and prevent overlapping builds of this job.

### Build an image compatible with the registry and cluster

The existing kind node uses ARM64. The build explicitly selects `linux/arm64` and disables provenance and SBOM attestations because the earlier Aliyun push rejected the attached OCI build metadata:

```sh
docker buildx build --platform "$IMAGE_PLATFORM" \
    --provenance=false --sbom=false --load \
    -t "$IMAGE_REPOSITORY:$IMAGE_TAG" ./app

docker push --platform "$IMAGE_PLATFORM" "$IMAGE_REPOSITORY:$IMAGE_TAG"
```

The application page remains `Application version: v1`. The page version identifies application content; the commit-derived image tag identifies the source revision used to build it. [Docker Buildx reference](https://docs.docker.com/reference/cli/docker/buildx/build/)

### Keep registry credentials out of Git and console output

The push stage binds `container-registry-credentials` using `usernamePassword`. It passes the password through `docker login --password-stdin` with command tracing disabled. A temporary Docker configuration holds the login data and is removed on success or failure. The Docker Desktop endpoint is captured before using that temporary configuration.

The deployment stages bind `kubeconfig-training` using the Jenkins secret-file binding. The pipeline does not print the kubeconfig or copy it into the application repository. [Jenkins Credentials Binding reference](https://www.jenkins.io/doc/pipeline/steps/credentials-binding/)

### Preserve meaningful rollout history

The source `k8s/deployment.yaml` retains its initial `:v1` image. During each pipeline run, `kubectl set image --local` renders a temporary manifest containing the final commit-tagged image. The pipeline applies that Deployment once.

This avoids applying the original `:v1` Deployment immediately before applying another image, which could create an intermediate rollout revision. Clean revisions will make the later v1-to-v2 update and rollback easier to verify. The original YAML is not rewritten. [kubectl set image reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_set/kubectl_set_image/)

## Task 4: Configure Jenkins

I created the Pipeline job [kubernetes-cd-demo](http://localhost:8080/job/kubernetes-cd-demo/) with the following settings:

| Jenkins field | Configuration |
| --- | --- |
| Definition | Pipeline script from SCM |
| SCM | Git |
| Repository URL | `https://github.com/maxfan2023/kubernetes-cd-demo.git` |
| Git credentials | Existing `github-token` credential |
| Branch specifier | `*/main` |
| Script Path | `Jenkinsfile` |
| Lightweight checkout | Enabled |

The Jenkinsfile must be committed and pushed to the remote branch before Jenkins can load it. [Jenkins Pipeline from SCM](https://www.jenkins.io/doc/book/pipeline/getting-started/#in-scm)

I configured the exact credential IDs required by the assignment:

| Credential ID | Type | Configuration |
| --- | --- | --- |
| `container-registry-credentials` | Username with password | Reused the existing `aliyun-docker-registery-token` credential inside Jenkins. |
| `kubeconfig-training` | Secret file | Selected only the existing `kind-k8s-chapter1` context from `k8s_config_file`. |

The original credentials were retained. The new kubeconfig excludes unrelated contexts and retains the selected identity's existing cluster permissions; no new Kubernetes RBAC policy was introduced. Passwords, tokens, and kubeconfig contents were not added to the repository or printed in the report.

## Validation completed

The following checks passed without scheduling a deployment build:

- The installed Jenkins Declarative converter accepted the Jenkinsfile.
- All five multiline shell blocks passed POSIX shell syntax checks.
- The exact Buildx shell block built a local validation image successfully with attestations disabled; Docker inspection confirmed `linux/arm64`. The temporary `task34-check` tag was removed after validation.
- The real `kubectl set image --local` command rendered the intended commit-tagged image, and the rendered resources passed Kubernetes server dry-run validation.
- A live test from the Jenkins controller process used `container-registry-credentials` to log in to Aliyun successfully. Its temporary Docker configuration was removed afterward.
- A live test from the Jenkins controller process used `kubeconfig-training` to access `namespace/training`; the identity was also allowed to patch Deployments in `training`.
- The saved Job's SCM branch, script path, and lightweight-checkout configuration were verified. Its build count remained zero.
- Simulated Docker push success and failure both removed the temporary login configuration and preserved the Docker endpoint.
- A simulated deployment applied the rendered Deployment exactly once.
- A simulated missing `cd-demo-acr-pull` Secret stopped execution before application workload manifests were applied.
- A simulated image mismatch caused the Verify stage to fail.

The shell simulations used dummy commands and dummy credentials. They exercise command flow and failure handling. The separate live credential tests establish registry login and selected Kubernetes access from Jenkins. These checks do not prove a real image push, a complete Jenkins pipeline run, Pod readiness, or HTTP availability.

## Remaining work for Tasks 5 and 6

The GitHub repository is currently empty and the local application files have not been committed. No Jenkins application build was scheduled, no new image was pushed, and the application was not deployed during Tasks 3 and 4.

Before the first deployment, the `training` Namespace still needs its `cd-demo-acr-pull` private-registry Secret. A previous attempt to transfer local Docker credentials into that cluster was rejected by automatic approval review, and that transfer was not performed in this task. The Jenkins push credential does not automatically provide image-pull access to Kubernetes.

Task 5 will commit and push the v1 project, run the Jenkins job, collect successful build and rollout evidence, update the page to v2, and repeat the deployment checks. Task 6 will collect rollout history, perform the rollback, and verify the restored page version. Those runtime and submission results remain pending.
