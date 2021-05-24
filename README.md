# ARO DevOps Labs

## Lab 5: Deploy Mongo DB

### Deploy Mongo DB with Helm

1. In this lab, we will deploy a Mongo DB instance alongside a new app in the `express-mongo-app` that requires the DB. The app is sourced from the following [repo](https://github.com/Azure-Samples/js-e2e-express-mongodb).

2. We will leverage the [bitnami helm chart](https://github.com/bitnami/charts/tree/master/bitnami/mongodb) to deploy Mongo Db. Run the following commands to add the repo and deploy an instance of Mongo DB:

    ```
    oc new-project lab-5

    helm repo add bitnami helm repo add bitnami https://charts.bitnami.com/bitnami

    helm install my-release bitnami/mongodb \
        --set auth.enabled=false \                               # removing the need for username/password for ease of use
        --set persistence.enabled=false \                        # removing the pvc for ease of use
        --set podSecurityContext.enabled=false \                 # disabling pod security context config
        --set containerSecurityContext.enabled=false             # disabling container security context config
    ```

3. To run a quick test to confirm it will integrate with the application, run the following commands to deploy an instance of the express-mongo-app.

    ```
    cd express-mongo-app

    oc new-app . --name=mongo-app

    oc start-build mongo-app --from-dir=. --follow
    ```

    > Info: Once the build is complete, you should see the app up and running. However, it is not yet connected to the Mongo DB instance. You can confirm this by clicking into the container's logs and seeing the output from the app (which reports an error connecting to the DB).

4. To setup the connection to the Mongo DB instance, we will update the Deployment with an environment variable in the console. Select the deployment for the app in the console and then select the Environment tab. We will add the following environment variable to the deployment:

    ```
    DATABASE_URL: mongodb://my-release-mongodb:27017
    ```

5. Once you save the update, you should find the app is redeployed. Select on the container logs and you should see a successful DB connection.

6. Run `oc create route edge app-route --service=mongo-app` to create a route for the app. You can interact with the UI to create name/job pairs that will be persisted in the DB.

    > Info: Test the DB persistence by restarting the app after creating a few records. You should find that when the app redeploys, the prior records appear in the UI.

### Deploy App with Mongo DB Connection from Helm Chart

1. To automate our environment variable connection to the deployed instance of Mongo DB, we will add the `DATABASE_URL` env var to our template.

2. Go to `charts/app-chart/values.yaml` and see the `databaseConnectionString` setting that has been added as a config value. Confirm that the connection string is the same as what was used in the previous section when you added it manually in the console.

3. In the `charts/app-chart/templates/deployment.yaml` file, you will see that an env-var field for the `DATABASE_URL` config has been added and references that value.

4. Run `helm install mongo-app-helm charts/app-chart/.` from the root of the project. Then run `oc start-build node-express-app --from-dir=express-mongo-app/.` to start the build.

5. Confirm in the console that the app was deployed with the new environment variable by selecting on the Deployment and viewing the Environment tab. Once the app is deployed, confirm the connection has been successfully made by viewing the logs and using the app.

### Add Mongo DB to Helm Chart

1. We will add the bitnami Mongo DB chart as a dependency to our app helm chart to be able to deploy Mongo alongside our app through helm.

2. Open `charts/app-chart/Chart.yaml` and uncomment the following at the bottom of the file:

    ```
    # dependencies:
    # - name: mongodb
    #   version: "10.15.2"
    #   repository: "https://charts.bitnami.com/bitnami"
    ```

3. Run `helm dependency build charts/app-chart/.` - after it runs, you should see a `charts` directory within `charts/app-chart` that holds the Mongo DB helm package. This directory will not be committed to the repo (the `.gitignore` file includes a match on `*.tgz`). Think of this as the `node_modules` folder but for Helm.

4. Since we now have the Mongo DB chart as a subchart, we can configure it in our `values.yaml` file. Uncomment the following in the `charts/app-chart/values.yaml` file to include the config settings needed for deploying Mongo DB.

    ```
    # mongodb:
    #   fullnameOverride: mongodb #override name so that it isn't pinned to release name
    #   auth:
    #     enabled: false
    #   persistence:
    #     enabled: true
    #   containerSecurityContext:
    #     enabled: false
    #   podSecurityContext:
    #     enabled: false
    ```

5. Update the `databaseConnectionString` value in the same file to be `mongodb://mongodb:27017` - this is because we specified the `fullNameOverride` in our Mongo DB settings, so the service name when Mongo DB will be deployed is going to be `mongodb`.

6. Before redeploying with helm, run `helm uninstall mongo-app-helm` to delete the prior app deployment.

7. Run `helm install mongo-app-with-db-helm charts/app-chart/.` from the root of the project. Then run `oc start-build node-express-app --from-dir=express-mongo-app/.` to start the build.

8. In the console, confirm that both the app and Mongo DB are deployed. Verify the connection is successful to the DB from the app.