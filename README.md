# ARO DevOps Labs

## References

- https://nvie.com/posts/a-successful-git-branching-model/
- https://docs.microsoft.com/en-us/azure/container-registry/container-registry-image-tag-version

## Lab 10: Git Workflow with Container Governance

Pre-reqs:
- Ensure that you've completed the ACR Integration from Lab 9

### Update Build Workflow with Base Image

> Info: Up to this point, we've leveraged the S2I build process. We will update the buildconfig to now leverage a base image stored in our container registry and use a Dockerfile to build our app.

### Implement Pull Request Workflow with Ephemeral Environment on Feature Branch

### Implement Merge Workflow into Develop Branch (Build Image and Deploy to DevBranch-QA and DevBranch-Staging)

### Implement Merge Workflow into Main Branch (Promote Image to MainBranch-Dev and MainBranch-Staging)

### Implement Release Workflow to Promote to Production