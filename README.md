# ARO DevOps Labs

## Lab 7: Config Maps and Secrets

### Provision Dev, Test, Prod Environments

1. To create a DevOps setup that can promote an image across environments, first we will create the different projects in OpenShift and deploy the required resources that will be used to build and run the application.

2. First, let's create three projects for our environments:

    ```
    oc new-project lab-7-dev
    oc new-project lab-7-test
    oc new-project lab-7-prod
    ```

3. Review the `charts/app-chart` directory to see the updates made to the helm chart. Deploy the associated values file for each environment:

    ```
    oc project lab-7-dev
    helm template dev charts/app-chart/. | oc apply -f -

    oc project lab-7-test
    helm template test charts/app-chart/. | oc apply -f -

    oc project lab-7-pord
    helm template prod charts/app-chart/. | oc apply -f -
    ```

4. Let's 'run' the pipeline steps manually to see how we will promote images across environments. First, build the image in dev by running the following: 

    ```
    oc project lab-7-dev
    oc start-build node-express-mongo-app --from-dir=express-mongo-app/. --follow
    ```

    > Info: Once this runs to completion, confirm in the console that the app is running as expected in the lab-7-dev project.

5. Once up and running in the dev project, run the following to promote the image to the test environment:

    ```
    oc tag node-express-mongo-app:latest lab-7-test/node-express-mongo-app:latest
    ```

    > Info: This will tag the imagestream in the lab-7-test project with the image built in the lab-7-dev project. Once that imagestream is updated, the deployment in lab-7-test is triggered to start.

6. Run the same command for the prod project:

    ```
    oc tag node-express-mongo-app:latest lab-7-prod/node-express-mongo-app:latest
    ```

    > Info: In the values-prod.yaml file, the mongodb name/connection is slightly different. This mocks a scenario where the prod deployment of the app would not be connecting to an ephemeral DB by showing how the app's configuration is dynamic across environments.

### Run in GitHub Actions

1. To setup to run our pipeline, let's deploy nexus and sonarqube to our labs-7-dev project:

```
oc project lab-7-dev
helm install nexus-lab-7 redhat-cop/sonatype-nexus
helm install sonarqube-lab-7 redhat-cop/sonarqube
```

2. Run the following commands to create a service account that has the priviledges to work across these three projects:

```
oc create sa github-actions-sa

oc adm policy add-cluster-role-to-user edit -z github-actions-sa        # Service Account is editor for all projects

# Now, we have to find the name of the secret in which the Service Account's apiserver token is stored.
# The following command will output two secrets. 
export SECRETS=$(oc get sa $SA -o jsonpath='{.secrets[*].name}{"\n"}') && echo $SECRETS
# Select the one with "token" in the name - the other is for the container registry.
export SECRET_NAME=$(printf "%s\n" $SECRETS | grep "token") && echo $SECRET_NAME

# Get the token from the secret. 
export ENCODED_TOKEN=$(oc get secret $SECRET_NAME -o jsonpath='{.data.token}{"\n"}') && echo $ENCODED_TOKEN;
export TOKEN=$(echo $ENCODED_TOKEN | base64 -d) && echo $TOKEN
```

3. Populate the required GitHub secrets to support the pipeline and then make a commit back to your lab-7 branch to run. You should see the pipeline execute in a similar fashion to what we ran locally with the image tagging, which is how the image gets promoted throughout the different environments.