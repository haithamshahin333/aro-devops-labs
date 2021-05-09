# ARO DevOps Labs

## Lab 2: Build, Bake, and Deploy App in GitHub Actions

### Deploy Openshift Resources with Helm Chart

1. Switch over to `lab-2` branch - `git checkout lab-2`.

2. Create a new project for this section - run `oc new-project lab-2`

3. View the helm chart templates rendered with the `values.yaml` config by running `helm template express-app charts/app-chart/.`

    > Info: View how these resources are almost identical to those created by `oc new-app`. We are leveraging the file in `charts/app-chart/values.yaml` to provide paramter values to the templates stored in `charts/app-chart/templates`. We can customize these values to modify our manifests without having to make direct changes to the templates themselves.

4. To deploy the resourcs to the `lab-2` project, run `helm template express-app charts/app-chart/. | oc apply -f -`. Confirm that you see the following:

    ```
    service/node-express-app created
    deployment.apps/node-express-app created
    buildconfig.build.openshift.io/node-express-app created
    imagestream.image.openshift.io/node-express-app created
    route.route.openshift.io/node-express-app created
    ```

    > Note: You will see a build error occur. This is because we specified a build type of binary but never passed an artifact to build against. This is what we will do in the next step.

5. To deploy the app, we need to start the build and pass in our source code for the s2i process to begin. Run `oc start-build node-express-app --from-dir=src --follow`.

    > Note: Take notice that our app source code is now in the `src` directory. This is not required, but done to make the repo cleaner.

6. Once this is complete, confirm that you can navigate to the app through the route on the openshift console.

### Build and Deploy App through GitHub Actions Pipeline

1. We will now build a basic CI/CD pipeline using GitHub Actions and GitHub Hosted runners. We will be leveraging the OpenShift GitHub Actions tasks found [here](https://github.com/redhat-actions). 

2. To support our workflow, we will need to create a [Service Account](https://github.com/redhat-actions/oc-login/wiki/Using-a-Service-Account-for-GitHub-Actions) so that we can securely connect to our cluster with a principal that has just the right amount of priviledges needed for our actions.

3. Run the following bash script:

    ```
    # First, name your Service Account (the Kubernetes shortname is "sa")
    export SA=github-actions-sa
    
    # and create it.
    oc create sa $SA

    # Now, we have to find the name of the secret in which the Service Account's apiserver token is stored.
    # The following command will output two secrets. 
    export SECRETS=$(oc get sa $SA -o jsonpath='{.secrets[*].name}{"\n"}') && echo $SECRETS
    # Select the one with "token" in the name - the other is for the container registry.
    export SECRET_NAME=$(printf "%s\n" $SECRETS | grep "token") && echo $SECRET_NAME

    # Get the token from the secret. 
    export ENCODED_TOKEN=$(oc get secret $SECRET_NAME -o jsonpath='{.data.token}{"\n"}') && echo $ENCODED_TOKEN;
    export TOKEN=$(echo $ENCODED_TOKEN | base64 -d) && echo $TOKEN
    ```

4. Run the following to grant that account edit access to the project: `oc policy add-role-to-user edit -z $SA`

5. Add the following secrets to your repo for the GitHub actions workflow to function:

    ```
    OPENSHIFT_SERVER        # this is the api url - run `oc cluster-info` to see this url
    OPENSHIFT_TOKEN         # this is the SA token - run `echo $TOKEN` to see this value
    OPENSHIFT_NAMESPACE     # this is the namespace/project being used - this should be `lab-2`
    ```
6. Make a change to the app in `src/public/index.html` and push the change to branch `lab-2`. You should now see the workflow run and start the S2I build in OpenShift. From there, the deployment will be triggered with the latest built image and the deployed app should reflect your change.