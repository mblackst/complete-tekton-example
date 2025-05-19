# Tekton Buildah Pipeline with Quay.io Integration on K3s

This guide walks you through setting up and running a Tekton pipeline on K3s that builds a container image using Buildah and pushes it to a [Quay.io](https://quay.io) repository.

---

## ✅ Prerequisites

- A running **K3s** cluster
- `kubectl` and `tkn` CLI tools installed
- A [Quay.io](https://quay.io) account with:
  - A repository named `hello-app`
  - A **robot account** created for automated pushes

---

## 🔐 1. Create a Docker Registry Secret in K3s

Use your Quay robot credentials (username & token):

```bash
kubectl create secret docker-registry quay-credentials \
  --docker-server=quay.io \
  --docker-username=mblackst1+builderbot \
  --docker-password=CLS5PFQ3KB3VP6TKGEY45RFSJ6ASLMRVVECVQU8WFLKQWHWHB8ID4HNRSWBP6V4G \
  --docker-email=dummy@example.com
```

> ⚠️ Replace with your actual credentials. The email field is not used but required syntactically.

---

## 📝 2. Tekton Pipeline YAML

Save the following as `buildah-quay-pipeline.yaml`:

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: buildah-quay-pipelinerun
spec:
  pipelineSpec:
    workspaces:
      - name: shared-data
    tasks:
      - name: write-dockerfile
        workspaces:
          - name: shared-data
            workspace: shared-data
        taskSpec:
          workspaces:
            - name: shared-data
          steps:
            - name: create-dockerfile
              image: busybox
              script: |
                #!/bin/sh
                mkdir -p $(workspaces.shared-data.path)
                echo 'FROM alpine' > $(workspaces.shared-data.path)/Dockerfile
                echo 'CMD ["echo", "Hello from Buildah-built image"]' >> $(workspaces.shared-data.path)/Dockerfile
                echo "✅ Dockerfile written"

      - name: build-and-push
        runAfter: [write-dockerfile]
        workspaces:
          - name: shared-data
            workspace: shared-data
        taskSpec:
          workspaces:
            - name: shared-data
          steps:
            - name: buildah-push
              image: quay.io/buildah/stable
              securityContext:
                privileged: true
              workingDir: $(workspaces.shared-data.path)
              volumeMounts:
                - name: quay-secret
                  mountPath: /auth
              env:
                - name: REGISTRY_AUTH_FILE
                  value: /auth/.dockerconfigjson
              script: |
                #!/bin/sh
                set -e
                IMAGE="quay.io/mblackst1/hello-app:latest"
                echo "Building image ${IMAGE} with Buildah..."
                buildah --storage-driver=vfs bud -t "${IMAGE}" .
                echo "Pushing image ${IMAGE} to Quay.io..."
                buildah --storage-driver=vfs push "${IMAGE}" "docker://${IMAGE}"
                echo "✅ Image pushed successfully."

          volumes:
            - name: quay-secret
              secret:
                secretName: quay-credentials
                items:
                  - key: .dockerconfigjson
                    path: .dockerconfigjson

  workspaces:
    - name: shared-data
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Mi
```

---

## 🚀 3. Deploy and Run the Pipeline

Apply the pipeline YAML:

```bash
kubectl apply -f buildah-quay-pipeline.yaml
```

Check the logs:

```bash
tkn pipelinerun logs buildah-quay-pipelinerun -f
```

Expected output includes:

```
✅ Dockerfile written
Building image quay.io/mblackst1/hello-app:latest with Buildah...
Successfully tagged quay.io/mblackst1/hello-app:latest
✅ Image pushed successfully.
```

---

## 🎉 Result

You now have a container image built by Tekton + Buildah and pushed to:

```
https://quay.io/repository/mblackst1/hello-app
```

---

## 🧩 Next Steps (Optional)

- Use a Git clone task to source a real app
- Add dynamic tags (e.g., date or Git SHA)
- Deploy the pushed image via `kubectl` or another Tekton task
- Set up a webhook trigger

Let me know if you'd like help extending the pipeline!


kubectl apply -f new-buildah-hello-pipeline.yaml
tkn pipelinerun logs -f -n default buildah-hello-pipelinerun


kubectl create secret docker-registry quay-credentials \
  --docker-server=quay.io \
  --docker-username=mblackst1+builderbot \
  --docker-password=CLS5PFQ3KB3VP6TKGEY45RFSJ6ASLMRVVECVQU8WFLKQWHWHB8ID4HNRSWBP6V4G \
  --docker-email=dummy@example.com


