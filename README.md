# Hands-on Guide: CI/CD Modernization on OpenShift

This repository contains the source code, manifests, and a step-by-step guide for the concepts demonstrated during our CI/CD architecture session. 

> **Architecture Note (Monorepo vs. Polyrepo):**  
> For demonstration and educational purposes, this repository uses a **Monorepo** approach. Application source code (`apps/`), pipeline definitions (`platform/`), and deployment configurations (`gitops/`) are all centralized here to provide a seamless, self-contained view of the entire lifecycle. In a production environment, this would be split into a **Polyrepo** architecture to strictly separate developer responsibilities from platform and operations scopes.

## Demonstration Goals
1. Understand the traditional OpenShift workflow: `oc new-app`, `BuildConfig`, `ImageStream`, and the `s2i` strategy.
2. Identify the limitations of the traditional model for strictly governed environments.
3. Introduce **OpenShift Pipelines (Tekton)** for Continuous Integration (CI).
4. Introduce **OpenShift GitOps (ArgoCD)** for Continuous Deployment (CD).
5. Demonstrate the migration path: from a tightly coupled S2I model to a GitOps **Greenfield** model using Dockerfiles.

---

## Phase 1: The Legacy Ecosystem (S2I and BuildConfigs)

To illustrate our starting point, we begin by creating a project and deploying an application the traditional way.
```bash
oc new-project demo-builds
```

### 1.1 The `oc new-app` Command
We can deploy applications starting from a local `ImageStream` or an image from the official catalog (https://catalog.redhat.com/en/search?searchType=Containers). *NOTE: demo branch is used as main branch*

Example using a local ImageStream (remove `--dry-run -oyaml` for creating the actual objects):

```bash
oc new-app \
  openshift/httpd:latest~https://github.com/CSA-RH/openshift-cicd-sample.git#demo \
  --name demo-apache \
  --context-dir apps/apache-s2i \
  --dry-run \
  -o yaml
```

Example using a UBI8 directly from the Red Hat Container Calatog (https://catalog.redhat.com/en/search?q=httpd-24) (Red Hat Universal Base Image) base image (remove `--dry-run -oyaml` for creating the actual objects). *NOTE: demo branch is used as main branch*

```bash
oc new-app \
    registry.access.redhat.com/ubi8/httpd-24~https://github.com/CSA-RH/openshift-cicd-sample.git#demo \
    --name=demo-apache \
    --context-dir apps/apache-s2i \
    --dry-run \
    -o yaml
```

### 1.2 The Concept of Triggers
The main engine of the traditional model relies on triggers. A `Deployment` or a `BuildConfig` executes automatically when it detects a change.

- **ImageStream Triggers**: If the base image changes, the build is triggered. We can simulate this by updating the UBI version (from 8 to 10). This example only works if we have picked an external image and the ImageStream is generated in the same namespace:

```bash
oc tag registry.access.redhat.com/ubi10/httpd-24:1791973027 httpd-24:latest -n demo-builds
```

This change on the `latest` tag will automatically trigger a new build and a subsequent deployment.

- Webhooks (Generic Triggers): Allow starting builds from external systems (like GitHub) via HTTP calls:

```bash
curl -X POST -k \
  -H "Authorization: Bearer $(oc whoami -t)" \
  -H "Content-Type: application/json" \
  -d '{}' \
   <WEBHOOK_URL>
```

### 1.3 Advantages and Limitations of the Classic Model

**Advantages**:
- Very low learning curve (deployments can be done directly from source code).
- Relies on secure, Red Hat-certified base images.
- Out-of-the-box automation thanks to built-in triggers.

**Limitations** (Why we need to evolve):
- Fat images: The builder image ultimately becomes the runtime image, unnecessarily increasing the artifact size and the attack surface in Production.
- Opacity: The build history resides inside the `ImageStream` as `ImageStreamTags`, complicating external auditing.
- GitOps Incompatibility: Triggers modify the Deployment by dynamically injecting specific image SHAs into the cluster. This breaks the GitOps philosophy, as the live state in the cluster diverges from the state declared in the Git repository or we are forced to exclude the `image` field from the GitOps strategy, which is the main driver of versioning for app management.

## Phase 2: Introduction to OpenShift Pipelines (Tekton)
The modernization strategy relies on decoupling. We will use Tekton (Pipelines) to centrally orchestrate the build process and ArgoCD for the deployment.

### 2.1 Our First Pipeline (demo-first)
In the demo, we visualize how the Pipeline Builder in OpenShift allows us to create workflows visually without writing complex YAML. We demonstrate parallel task execution (e.g., greeting and cluster-info) converging into a final task. Simply construct a Pipeline with the Pipeline Builder to execute very basic commands. 

> **NOTE**: For demonstration and brevity purposes, we have omitted most of the RBAC adjustments and configuration needed for ArgoCD and Tekton service accounts. Please apply the least privilege principle for those in production environments.

As a reference, here an example of the pipeline

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: demo-first
  namespace: demo-builds
spec:
  finally:
    - name: final
      params:
        - name: SCRIPT
          value: echo "This is the end"
        - name: VERSION
          value: latest
      taskRef:
        params:
          - name: kind
            value: task
          - name: name
            value: openshift-client
          - name: namespace
            value: openshift-pipelines
        resolver: cluster
  tasks:
    - name: greetings
      params:
        - name: SCRIPT
          value: echo "Hello world"
        - name: VERSION
          value: latest
      taskRef:
        params:
          - name: kind
            value: task
          - name: name
            value: openshift-client
          - name: namespace
            value: openshift-pipelines
        resolver: cluster
    - name: cluster-info
      params:
        - name: SCRIPT
          value: oc cluster-info
        - name: VERSION
          value: latest
      taskRef:
        params:
          - name: kind
            value: task
          - name: name
            value: openshift-client
          - name: namespace
            value: openshift-pipelines
        resolver: cluster
```

## Phase 3: Deconstructing an S2I App towards GitOps
Instead of letting `oc new-app` handle everything implicitly, we use Tekton to govern the flow.

This demo pipeline (`demo-buildconfig-apache`) maintains compatibility with the current legacy code but extracts the output and automatically generates a Pull Request in our GitOps repository, allowing ArgoCD to securely take control.

 **Crucial GitOps Step: Removing the OpenShift Image Trigger** 
 
 When migrating an application that was originally deployed via oc new-app (S2I) to be managed by ArgoCD, there is a critical cleanup step required on the Deployment manifest.

Legacy deployments contain an internal OpenShift annotation that watches for ImageStream changes:

```yaml
metadata:
  annotations:
    image.openshift.io/triggers: '[{"from":{"kind":"ImageStreamTag","name":"demo-apache:latest"},"fieldPath":"spec.template.spec.containers[?(@.name==\"demo-apache\")].image"}]'
```

**This annotation MUST be removed from the manifest stored in your GitOps repository.**

If not removed, it will cause an infinite reconciliation loop:

1. OpenShift's internal BuildEngine detects a new image and mutates the Deployment in the cluster.

2. ArgoCD detects this mutation as Drift (the cluster state no longer matches the Git state).

3. ArgoCD's Self-Heal mechanism overrides the OpenShift mutation to restore the Git state.

4. The cycle repeats indefinitely, or the application remains eternally `OutOfSync`.

By removing this annotation, we strip OpenShift of its implicit deployment power, explicitly transferring full deployment authority to ArgoCD and the Git repository, which is the fundamental premise of GitOps.

You can find the Application GitOps object for this example [here](/gitops/apps/demo-buildconfig-apache.yaml)

**Custom Tasks Created:**

1. `ct-tag-imagestream`: Extracts the exact Git commit SHA and tags the OpenShift image, moving away from the bad practice of using the latest tag.

2. `ct-github-open-pr`: Clones the GitOps repository, injects the new image tag into the deployment manifest using yq, commits the changes, and opens a GitHub PR unattended.

(The complete YAML definitions for these tasks can be found in the `platform/tasks/` directory of this repository).

**Pipeline Flow (`demo-buildconfig-apache`):**

1. Triggers the legacy build process (`oc start-build`).

2. Tags the resulting image appropriately.

3. Opens the GitOps PR for human or automated approval.

## Phase 4: The Greenfield Model (Dockerfile + GitOps)
For new projects (Greenfield) or refactored applications, we can consider the Multi-stage Dockerfiles using Buildah, which in turn provides more flexibility (*With great power comes great responsibility*).

Example Application: **Spring Petclinic**

Original Repository: https://github.com/bizcochillo/spring-petclinic

The pipeline (`demo-dockerfile-petclinic`) defines the new enterprise standard:

1. `git-clone`: Downloads the source code and the Dockerfile.

2. `buildah`: Builds the image directly inside the cluster (daemonless) natively and securely, pushing it to the internal OpenShift registry.

3. `ct-tag-imagestream`: Tags the image uniquely.

4. `ct-github-open-pr`: Modifies the manifests in the GitOps repository and opens the PR.

Once this PR is approved and merged in GitHub, **OpenShift GitOps (ArgoCD)** detects the drift and synchronizes the new version of Petclinic into the cluster in a deterministic and fully auditable manner.

