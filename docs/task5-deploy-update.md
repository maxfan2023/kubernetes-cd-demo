# Task 5: Deploy and Update the Application

Date: 2026-10-28. Build start times below use Asia/Shanghai (UTC+08:00); timestamps in saved Jenkins logs and Kubernetes snapshots use UTC.

## Result

Both releases completed through the configured **Pipeline script from SCM** job. The repository was pushed to GitHub, Jenkins loaded the root `Jenkinsfile` from `main`, checked out the corresponding commit, built and pushed an ARM64 image to Aliyun ACR, deployed it to Kubernetes, and completed all five stages successfully.

| Page | Git commit / image tag | Jenkins build | Deployment revision | Ready | Build started (UTC+08:00) |
| --- | --- | --- | --- | --- | --- |
| v1 | [`1ff38fb17176`](https://github.com/maxfan2023/kubernetes-cd-demo/commit/1ff38fb17176f82335e325294c45edab0767b009) | [#1](http://localhost:8080/job/kubernetes-cd-demo/1/) SUCCESS | 4 | 3/3 | 2026-10-28 12:31:37 |
| v2 | [`64f2a5d31ea5`](https://github.com/maxfan2023/kubernetes-cd-demo/commit/64f2a5d31ea5a3da6ae53a5779c9d3d00fdef378) | [#2](http://localhost:8080/job/kubernetes-cd-demo/2/) SUCCESS | 5 | 3/3 | 2026-10-28 12:33:46 |

Target: `kind-k8s-chapter1` / `training`, Deployment `cd-demo-deployment`, container `cd-demo`, Service `cd-demo-service`, Ingress host `cd-demo.local`.

Repository: [maxfan2023/kubernetes-cd-demo](https://github.com/maxfan2023/kubernetes-cd-demo). The full source commits are:

- v1: `1ff38fb17176f82335e325294c45edab0767b009`.
- v2: `64f2a5d31ea5a3da6ae53a5779c9d3d00fdef378`.

The image repository is:

```text
crpi-uzkholcdutt858o7.cn-guangzhou.personal.cr.aliyuncs.com/maxfan/kubernetes-cd-demo
```

Image tags are the first 12 characters of each Git commit, as implemented in Task 3. The v1/v2 names identify the page content. The pushed image digests, also verified against every running Pod's `imageID`, are:

| Page | Image tag | Registry digest |
| --- | --- | --- |
| v1 | `1ff38fb17176` | `sha256:78d432789f28456bb37d93221bff9627a05cd0ce4c71b4a3d14d02324f8ccc72` |
| v2 | `64f2a5d31ea5` | `sha256:bd0917b06a12aeb128ec4005cc2d8acb094f848c571ba069b942d64732b100a1` |

## Release procedure and verification

1. Checked the existing SCM Job configuration, live cluster, pull Secret name, Git status, and empty remote `main`. The local default kubectl context was `docker-desktop`; all Kubernetes commands explicitly selected `kind-k8s-chapter1`.
2. Scanned the selected publication files for private keys, credential tokens and embedded kubeconfig fields. Committed the prepared v1 project with explicit paths and pushed it to `origin/main`. The unrelated staged `main.py` and its unstaged contents were preserved and excluded from all release commits.
3. Triggered the formal SCM Job, build #1. Saved its successful console log and verified that it checked out the exact v1 commit. After rollout, verified Deployment generation, desired/updated/ready/available replicas, three running Pods, their commit-tagged images and image digests, and three matching ready Service endpoints.
4. Requested the page from each Pod with `wget http://127.0.0.1/`. Separately forwarded the application Service and Ingress Controller Service to temporary loopback ports, and requested the Ingress path using `Host: cd-demo.local`. All responses exactly matched the saved v1 HTML; Service and Ingress returned HTTP 200. Temporary forwards were stopped.
5. Changed only the page version from v1 to v2, committed it, pushed `main`, and triggered formal build #2. Repeated the same build, rollout, digest, endpoint and full-page HTTP checks against v2.
6. Saved ReplicaSet snapshots before and after the update and a read-only sample of ReplicaSet transitions during the v2 rollout. The v1 ReplicaSet scaled to zero; the v2 ReplicaSet reached three ready replicas. The runtime remains on v2.

The application change in the v2 release commit was:

```diff
-  <p>Application version: v1</p>
+  <p>Application version: v2</p>
```

## Rolling update evidence

| ReplicaSet | Page | Revision | After v1 rollout (desired/ready) | After v2 rollout (desired/ready) |
| --- | --- | --- | --- | --- |
| `cd-demo-deployment-86d87f45f7` | v1 | 4 | 3/3 | 0/0 |
| `cd-demo-deployment-798549d49b` | v2 | 5 | Not created | 3/3 |

Revision numbers start at 4 here because earlier local Script tests already created revisions. The existing history was retained. The Task 3 pipeline renders the final commit-tagged Deployment before applying it, so the v2 release produced one new revision (4 → 5).

[Live transition samples](task5/v2/rollout-watch.txt) and [v2 resource output](task5/v2/runtime.txt) show the rolling replacement. These point-in-time and rollout observations do not constitute a continuous availability/load test.

## Saved submission evidence

| Evidence | v1 | v2 |
| --- | --- | --- |
| Jenkins console log | [build.log](task5/v1/build.log) | [build.log](task5/v2/build.log) |
| Build metadata and pushed digest | [build.json](task5/v1/build.json) | [build.json](task5/v2/build.json) |
| Rollout, Pods, ReplicaSets and HTTP checks | [runtime.txt](task5/v1/runtime.txt) | [runtime.txt](task5/v2/runtime.txt) |
| Machine-readable assertions | [evidence.json](task5/v1/evidence.json) | [evidence.json](task5/v2/evidence.json) |
| Deployment snapshot | [deployment.json](task5/v1/deployment.json) | [deployment.json](task5/v2/deployment.json) |
| Pod snapshot | [pods.json](task5/v1/pods.json) | [pods.json](task5/v2/pods.json) |
| ReplicaSet snapshot | [replicasets.json](task5/v1/replicasets.json) | [replicasets.json](task5/v2/replicasets.json) |
| Ready Service endpoints | [endpointslices.json](task5/v1/endpointslices.json) | [endpointslices.json](task5/v2/endpointslices.json) |
| Verified HTML | [page.html](task5/v1/page.html) | [page.html](task5/v2/page.html) |

The console logs were copied from the completed Jenkins build records with Jenkins display annotations removed. Original log whitespace and UTC timestamps were retained. No credentials, kubeconfig contents or Secret values are included in these records.

To inspect the current deployment without changing it:

```bash
kubectl --context=kind-k8s-chapter1 -n training rollout status \
  deployment/cd-demo-deployment --timeout=180s
kubectl --context=kind-k8s-chapter1 -n training get pods -l app=cd-demo
kubectl --context=kind-k8s-chapter1 -n training get replicasets -l app=cd-demo
kubectl --context=kind-k8s-chapter1 -n training rollout history \
  deployment/cd-demo-deployment
```

## Troubleshooting notes and remaining scope

- The SCM checkout and both releases succeeded. The existing `github-token`, `container-registry-credentials`, `kubeconfig-training` and Kubernetes `cd-demo-acr-pull` configuration were sufficient.
- Explicit context selection ensured these operations targeted the course kind cluster despite the unrelated default context.
- Explicit Git path selection prevented the unrelated staged PyCharm `main.py` from entering the published project. Its index entry and working-tree hash were checked after the release work.
- Source `k8s/deployment.yaml` retains its original `:v1` image. Reapplying it directly would deploy that initial image; Jenkins substitutes the intended commit tag into a temporary manifest for each release.
- The report and evidence are committed separately after the two application release commits. A later documentation-only repository head does not change the running image; the deployed source commit remains the v2 commit listed above.
- Task 6 has not run. No rollback was performed. Revision 4 and its v1 ReplicaSet remain available as the next task's rollback target. The current runtime is the verified v2 release.
