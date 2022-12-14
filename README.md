# Flux-CI-CD
Cloud Computing project using GitHub, Kubernetes and Flux for a CI/CD pipeline.

## Proposal

We plan to set up a CI/CD pipeline using GitHub, DockerHub, Kubernetes and Flux. This would include automatic testing and image generation, live deployment updates using these images, deployment manifest updates with new version numbers and deployment manifest synchronization between GitHub and the running cluster.

![Flow Diagram](assets/project_diagram-1.png)

![Actor Diagram](assets/project_diagram-2.png)

As shown, the image name would depend on whether a tag was used or not. The kubernetes cluster would only update to versions with the same major version and ignore all non semver named images, i.e. main-j7gjk-12034 would be ingored.

As a basis we would use the deployment demo presented in the lecture. We would update the CI pipeline to produce the naming discussed above. We would then add Flux for manifest synchronization and [Flux image updates](https://fluxcd.io/flux/guides/image-update/) for automatic version updating.

We believe this project should be categorized as advanced, because it extends the project "GitOps" described in the "Project Ideas" pdf by also incorporating automatic image updates.

### Intended Development Usage

This would be intended for usage with [Trunk-based development](https://cloud.google.com/architecture/devops/devops-tech-trunk-based-development) shown in the image below.

![Trunk-based development](https://cloud.google.com/static/architecture/devops/images/devops-tech-trunk-based-development-typical-trunk-timeline.svg)

### Demo

For demo purposes, we could create a new repository and show all steps involved or use an existing repository to demonstrate the (push tag -> generate image -> update manifests -> update deployments) pipeline, or both.

### Work split

| Member      | Tasks |
| ----------- | ----------- |
| Christoph Pfleger | Create Proposal (find resources, check feasibility, design illustrations), Adapting CI pipeline, add basic Flux reconciliation |
| Luis Nachtigall | Discussion on how to implement Proposal with Development workflow, Add image updates (to Kubernetes cluster and GitHub manifests) |

### Extra Ideas

If this is seen as too small or to unrealistic, there is an additional idea we could implement:
- Adding a staging cluster that automatically loads images from normal pushes for testing

## Setup

Increase nodes to 2, otherwise it won't run.

### 1. Create Flux Files

```
set GITHUB_TOKEN=
set GITHUB_USER=

flux bootstrap github ^
    --components-extra=image-reflector-controller,image-automation-controller ^
    --owner=%GITHUB_USER% ^
    --repository=Flux-CI-CD ^
    --branch=main ^
    --path=cluster ^
    --interval=1m0s ^
    --read-write-key ^
    --personal

kubectl get events -n flux-system --field-selector type=Warning
```

### 2. Add automatic image updates

Create an image repository object, which periodically checks for new images.

```
flux create image repository app ^
    --image=chripp/app ^
    --interval=1m0s ^
    --export > ./cluster/app-registry.yaml
```

Create an image policy, which specifies how images are updated. In this case "^v1.0.0" means that it updates minor and bug versions, but not major versions.

```
flux create image policy app ^
    --image-ref=app ^
    --select-semver="^v1.0.0" ^
    --export > ./cluster/app-policy.yaml
```

Wait for reconcilliation, then check that everything works with the command below.

```
flux get image policy app
```

In cluster/deployment.yaml after "image: chripp/app:v1.0.0" add:
```
# {"$imagepolicy": "flux-system:app"}
```

At this point automatic image updates already work. However, while the live environment would get updated, the github repository would not reflect that. Therefore, there is one last step.

Add an automation object which periodically updates the config files in the github repo.

```
flux create image update flux-system ^
    --git-repo-ref=flux-system ^
    --git-repo-path="./cluster" ^
    --checkout-branch=main ^
    --push-branch=main ^
    --author-name=fluxcdbot ^
    --author-email=fluxcdbot@users.noreply.github.com ^
    --commit-template="{{range .Updated.Images}}{{println .}}{{end}}" ^
    --interval=1m0s ^
    --export > ./cluster/demo-automation.yaml
```

Again wait for reconcilliation, then check for errors.

```
flux get images all --all-namespaces
```

### 3. Add Github Action

Move setup/CI.yml to .github/workflows/


## Remove

```
flux uninstall
kubectl delete deployment demo
```
