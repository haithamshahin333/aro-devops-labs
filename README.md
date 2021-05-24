# ARO DevOps Labs

## Lab 5: Deploy Mongo DB

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

4. To setup the connection to the Mongo DB instance, we will update the Deployment with an environment variable in the console. 

### Deploy Mongo DB with Helm

### Deploy App with Mongo DB from Helm Chart

### Add Mongo DB to Helm Chart
