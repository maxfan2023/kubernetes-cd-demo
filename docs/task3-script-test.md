# Task 3: Full Jenkins Pipeline Script Test Without GitHub

Date: 2026-10-28 (Asia/Shanghai).

## Actual result

The independent Jenkins job **kubernetes-cd-demo-script-test**, build **#1**, finished with **SUCCESS**.

- Build: <http://localhost:8080/job/kubernetes-cd-demo-script-test/1/>
- Console: <http://localhost:8080/job/kubernetes-cd-demo-script-test/1/console>
- Definition: **Pipeline script**, with the Groovy sandbox enabled.
- Image tag: `4490e5590699`.
- Registry repository: `crpi-uzkholcdutt858o7.cn-guangzhou.personal.cr.aliyuncs.com/maxfan/kubernetes-cd-demo`.
- Pushed image digest: `sha256:78d432789f28456bb37d93221bff9627a05cd0ce4c71b4a3d14d02324f8ccc72`.
- Cluster/context: `kind-k8s-chapter1`.
- Namespace: `training`.
- Deployment: `cd-demo-deployment`, **3/3 Ready**.
- Service: `cd-demo-service`, with three Ready endpoints.
- Ingress: `cd-demo-ingress`, host `cd-demo.local`.
- Application content: `Application version: v1`.
- Deployment rollout revision: `1`.

## How the script ran without GitHub

The production `Jenkinsfile` uses `checkout scm`, which needs an SCM-backed Job and a remote commit. For this test, I saved an inline version as `Jenkinsfile.script-test` and used a separate Job with Definition **Pipeline script**.

Only the Checkout stage differs. It cleans the dedicated test Job workspace, copies `app/`, `k8s/`, and the production `Jenkinsfile` from the local application directory, then creates a Git snapshot commit inside that workspace. The pipeline derives its image tag from this local commit.

The application repository at `/Users/max/PycharmProjects/PythonProject/kubernetes-cd-demo` was not committed or pushed. No source code was uploaded to GitHub. The original `kubernetes-cd-demo` SCM Job and production `Jenkinsfile` were retained.

The remaining stages are identical to the production Jenkinsfile:

1. **Validate:** File checks and Git, Docker, Buildx, and kubectl checks passed.
2. **Build and Push Image:** The ARM64 image was built with provenance/SBOM attestations disabled and successfully pushed to Aliyun under the local snapshot tag.
3. **Deploy to Kubernetes:** The final image was rendered into a temporary Deployment manifest before applying it. The pipeline created the application Service, Ingress, and Deployment, then completed the rollout.
4. **Verify:** The running image matched the expected repository and tag. All three Pods were Running and Ready, and the Service had three Ready endpoints.

This is an actual Jenkins execution of the deployment stages. Remote GitHub checkout remains untested by this inline run.

## Private registry prerequisite

The user explicitly authorized reusing the Jenkins Registry credential to create `training/cd-demo-acr-pull` in the local kind cluster. That Secret was created before the build and is referenced by the existing Deployment YAML. Credential values were passed directly to the Kubernetes API and were not added to the repository or printed in the logs.

## HTTP verification after the build

From a Ready application Pod, I made HTTP requests through both paths:

```text
http://cd-demo-service.training.svc.cluster.local/
http://ingress-nginx-controller.ingress-nginx.svc.cluster.local/
  with Host: cd-demo.local
```

Both requests succeeded, and both complete response bodies exactly matched the local `app/index.html`, including `Application version: v1`.

## Evidence

- [Successful Jenkins console log](task3-script-test-build-1.log).
- [Kubernetes runtime and HTTP validation](task3-script-test-runtime.txt).
- [Inline Pipeline script](../Jenkinsfile.script-test).

The console log preserves Jenkins' UTC timestamps; the report date above uses Asia/Shanghai. Jenkins console annotations were removed for readability. This run supplements the earlier configuration-only Task 3/4 validation.

## Repeating the local test

Open the test Job and use **Build Now**. To recreate it manually, choose **Pipeline script**, paste `Jenkinsfile.script-test`, keep **Use Groovy Sandbox** selected, and save. The script expects the same local source path, built-in Jenkins node, Docker Desktop socket, and the two configured Jenkins credential IDs.

Each run creates a snapshot commit only in this Job's workspace, pushes its resulting image tag, and deploys it to the local `training` Namespace. The local-source version is intended for this macOS environment.

Task 5's GitHub SCM publication and v2 update, and Task 6's rollback, remain separate work. This run provides actual v1 deployment evidence without uploading source code.
