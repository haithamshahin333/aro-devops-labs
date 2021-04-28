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