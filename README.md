# ARO DevOps Labs

## Lab 6: Config Maps and Secrets

### Create a Config Map

1. We will create a config map to store the `DATABASE_URL` and `PORT` environment variables.

2. Run the following to create a configmap:

    ```
    oc create configmap app-config \
        --from-literal=CONFIG_MAP_PORT=8080 \
        --from-literal=CONFIG_MAP_DATABASE_URL=mongodb://mongodb:27017
    ```

    > Info: we prefixed the variables above with `CONFIG_MAP_` for our testing purposes, but this is not needed and will be taken off to actually use these values later in this lab.

3. View the data in the console.

4. In the console, add the config map as an environment variable by going to the deployment and specifying the config map.

5. You should see the pod redeploy - navigate to the terminal and run `printenv | grep CONFIG_MAP`. You should see the two variables configured above set as env vars in the container.

### Use Config Map in Helm Chart

1. View the `values.yaml` file and notice the config section.

2. Run `helm template test . -s templates/deployment.yaml` and `helm template test . -s templates/cm.yaml` to see how changes affect the rendered template.

3. To test, run `oc new-project helm-test-cm` and then run `helm install app .`

4. Start the build with `oc start-build node-express-mongo-app --from-dir=app/js-e2e-express-mongodb/.`

5. Run `oc logs -f deployment/node-express-mongo-app` to view the log output

### Create a Secret

1. We will create a secret to store the `DATABASE_URL` and `PORT` environment variables.

2. Run the following to create a secret:

    ```
        oc create secret generic app-secret \
            --from-literal=SECRET_PORT=8080 \
            --from-literal=SECRET_DATABASE_URL=mongodb://mongodb:27017
    ```

    > Info: we prefixed the variables above with `SECRET_` for our testing purposes, but this is not needed and will be taken off to actually use these values later in this lab.

3. View the data in the console.

4. In the console, add the secret as an environment variable by going to the deployment and specifying the secret.

5. You should see the pod redeploy - navigate to the terminal and run `printenv | grep SECRET`. You should see the two variables configured above set as env vars in the container.

### Use Secret in Helm Chart as Volume

1. View the `values.yaml` file and notice the config section.

2. Run `helm template test . -s templates/deployment.yaml` and `helm template test . -s templates/secret.yaml` to see how changes affect the rendered template.

3. To test, run `oc new-project helm-test-secret` and then run `helm install app .`

4. Start the build with `oc start-build node-express-mongo-app --from-dir=app/js-e2e-express-mongodb/.`

5. Run `oc logs -f deployment/node-express-mongo-app` to view the log output

6. View the terminal and run `cd /etc/secret-volume` to see the secret data

### Connect to Cosmos DB (Mongo DB API)

1. Create a basic Cosmos DB instance in the Azure Portal

2. Create a secret to store the connection string:

    ```
    oc create secret generic cosmos-db-app-secret \
        --from-literal=DATABASE_URL=<cosmosdb connection string - copy from portal>
    ```
3. In the portal, add the secret to the deployment so that the DATABASE_URL environment variable is picked up with the cosmos db connection string.

4. Verify the app is up and running. Create a basic record and then view the record in the data explorer on the cosmos db portal in Azure.