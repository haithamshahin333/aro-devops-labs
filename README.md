# ARO DevOps Labs

## Lab 1: Deploy ARO with Basic Express/Node JS App

### Deploy ARO

1. Run the `deploy-aro.sh` script if you have yet to deploy an ARO cluster within your subscription.

    >Note: Be sure to add the `pull-secret.txt` file in the same directory as `deploy-aro.sh` so that the cluster is deployed with the sample images/templates.

### Instantiate app in ARO with oc new-app

2. Once the cluster is up an running, fork this repo with your GitHub account and then clone the forked repo into a local directory.

3. Navigate to the root folder and run `npm install` to pull down dependencies.

4. Run `npm start` to run the app locally. Navigate to `http://localhost:8080` and you'll find the app running.

5. Now, we will create the resources needed in the cluster to run the app in a new project.

6. Run `oc new-project test-app` to create your `test-app` project.

7. Run `oc new-app . --name=node-express-app` - notice the output and view the following resources that were made in Openshift:

    ```
    --> Creating resources ...
    imagestream.image.openshift.io "node-express-app" created
    buildconfig.build.openshift.io "node-express-app" created
    deployment.apps "node-express-app" created
    service "node-express-app" created
    ```

8. Run `oc start-build express-node-app --from-dir . --follow` to start the image build in Openshift.

    >Info: This is the s2i process in Openshift. Openshift recognized this as a node project because of the package.json file in the directory and is able to select the proper base image and build steps to build a container image. No Dockerfile needed for this case!

9. Run `oc expose service/node-express-app` to create a Route resource and expose the app endpoint.

10. Run `oc get route/node-express-app` to get the URL for your app.

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

5. To deploy the app, we need to start the build and pass in our source code for the s2i process to begin. Run `oc start-build node-express-app --from-dir=src`.

6. Once this is complete, confirm that you can navigate to the app through the route on the openshift console.

### Build and Deploy App through GitHub Actions Pipeline



## Lab 3: Deploy Nexus, SonarQube

## Lab 4: Container Scanning & E2E Testing

## Lab 5: Operators & GitOps with Argo CD