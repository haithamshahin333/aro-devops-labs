# ARO DevOps Labs

## References

- https://nvie.com/posts/a-successful-git-branching-model/
- https://docs.microsoft.com/en-us/azure/container-registry/container-registry-image-tag-version

## Lab 10: Expanded Git Workflow with Container Management

Pre-reqs:

  - Ensure that you've completed the ACR Integration from Lab 9

### Update Build Workflow with Base Image

> Info: Up to this point, we've leveraged the S2I build process only. We will update the buildconfig to now leverage a base image stored in our container registry and use a Dockerfile to build our app.

1. Run `oc new-project lab-10`.

2. Run `helm install lab-10-release charts/app-chart/. -f charts/app-chart/values-dev.yaml`.

  > Info: Notice the Dockerfile in the `express-mongo-app` folder. This was the new piece that allows us to run a docker build in the cluster.

### Implement Pull Request Workflow with Ephemeral Environment on Feature Branch

### Implement Merge Workflow into Develop Branch (Build Image and Deploy to DevBranch-QA and DevBranch-Staging)

### Implement Merge Workflow into Main Branch (Promote Image to MainBranch-Dev and MainBranch-Staging)

### Implement Release Workflow to Promote to Production

### Helper ACR Commands

```
# Show Repository Manifests
az acr repository show-manifests -n $ACR_NAME -g $RG_NAME --repository lab-10


```