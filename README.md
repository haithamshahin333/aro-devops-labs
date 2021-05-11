# ARO DevOps Labs

## Lab 3: Deploy Nexus & SonarQube

### Deploy Nexus

1. Switch over to `lab-3` branch - `git checkout lab-3`.

2. Create a new project for this lab - run `oc new-project lab-3`.

3. We will leverage the work done by [redhat-cop](https://github.com/redhat-cop/helm-charts) to have access to a repository of charts for tools that can run on OpenShift.

    > Note: We will be installing mostly default config settings - in your own environment, you'll want to update these as needed for a more production-ready/hardened approach to deploying these tools.

4. Run `helm repo add redhat-cop https://redhat-cop.github.io/helm-charts` to add the helm repository.

5. Run `helm search repo redhat-cop` to see all the different charts available in the repo. We will focus on `redhat-cop/sonatype-nexus` and `redhat-cop/sonarqube`.

    > Info: View the default values.yaml settings for a chart by running `helm show values redhat-cop/sonatype-nexus`. These values are what you may need to update to run properly in your environment as well as when you look to harden the deployment.

6. Run `helm install nexus-lab-3 redhat-cop/sonatype-nexus` to install an instance of nexus using the default values.

7. Once nexus is up and running, open the route and modify the default admin username/password by signing in with the default credentials and then selecting the 'admin' username. You'll see a form where you can update the password with a new password.

### Integrate into GitHub Actions Pipeline (Proxy for NPM Registry)

> Info: In this exercise, we'll just add Nexus as a proxy to the NPM Registry. It can be used for much more, so take a look at what you might need (for example, publish internal packages to Nexus).

1. Add a secret to your repo which will hold the route to the proxy. Name the secret `NEXUS_PROXY` and obtain the route through the Nexus UI by copying the URL for the npm proxy repository.

2. Push the change and view the action run. When it completes, navigate to your Nexus instance and select the repository. You should see the dependencies that were used for the app all in the repository.

