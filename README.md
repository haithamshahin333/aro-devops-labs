# ARO DevOps Labs

## Lab 9: Azure Integrations with ARO

### Azure Key Vault Integration

> Info: [Repo](https://github.com/Azure/secrets-store-csi-driver-provider-azure/tree/master/charts/csi-secrets-store-provider-azure) for the helm chart.

Reference List:
- https://www.openshift.com/blog/managing-sccs-in-openshift
- https://github.com/aramase/secrets-store-csi-driver-provider-azure/blob/openshift-docs/website/content/en/configurations/deploy-in-openshift.md
- https://azure.github.io/secrets-store-csi-driver-provider-azure/demos/standard-walkthrough/
- https://secrets-store-csi-driver.sigs.k8s.io/topics/best-practices.html
- https://docs.openshift.com/container-platform/4.7/openshift_images/managing_images/using-image-pull-secrets.html
- https://docs.openshift.com/container-platform/4.7/registry/accessing-the-registry.html

1. Run the following:

    ```
    helm repo add csi-secrets-store-provider-azure https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/master/charts
    helm install csi-secrets-store-provider-azure/csi-secrets-store-provider-azure --generate-name
    ```
2. Then run the following:

```
oc adm policy add-scc-to-user privileged system:serviceaccount:$target_namespace:secrets-store-csi-driver
oc adm policy add-scc-to-user privileged system:serviceaccount:$target_namespace:csi-secrets-store-provider-azure
```

3. az cli setup commands:

```
export SUBSCRIPTION_ID="<SubscriptionID>"
export TENANT_ID="<tenant id>"

# login as a user and set the appropriate subscription ID
az login
az account set -s "${SUBSCRIPTION_ID}"

export KEYVAULT_RESOURCE_GROUP=<keyvault-resource-group>
export KEYVAULT_LOCATION=eastus
export KEYVAULT_NAME=secret-store-$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 10 | head -n 1)
```

4. Create a keyvault instance and secret:

```
 az group create -n ${KEYVAULT_RESOURCE_GROUP} --location ${KEYVAULT_LOCATION}
 az keyvault create -n ${KEYVAULT_NAME} -g ${KEYVAULT_RESOURCE_GROUP} --location ${KEYVAULT_LOCATION}
```

5. Add a secret to the vault instance:

```
az keyvault secret set --vault-name ${KEYVAULT_NAME} --name secret1 --value "Hello!"
```

> Info: Validate that the secret exists through the Azure portal.

6. Create an Azure Service Principal that the secret provider can use in ARO:

```
# Create a service principal to access keyvault
az ad sp create-for-rbac --skip-assignment --name http://secrets-store-test-aro
export SERVICE_PRINCIPAL_CLIENT_SECRET=<password from output of prior command>
export SERVICE_PRINCIPAL_CLIENT_ID=<appId from output of earlier command>

az keyvault set-policy -n ${KEYVAULT_NAME} --secret-permissions get --spn ${SERVICE_PRINCIPAL_CLIENT_ID} -g ${KEYVAULT_RESOURCE_GROUP}
```

6. Run `oc new-project lab-9` and create the following kubernetes secret:

```
kubectl create secret generic secrets-store-creds --from-literal clientid=${SERVICE_PRINCIPAL_CLIENT_ID} --from-literal clientsecret=${SERVICE_PRINCIPAL_CLIENT_SECRET}
```

7. Run the following to create a Secret Provider Class:

```
cat <<EOF | oc apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: azure-kvname
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "false"
    userAssignedIdentityID: ""
    keyvaultName: "${KEYVAULT_NAME}"
    objects: |
      array:
        - |
          objectName: secret1              
          objectType: secret
          objectVersion: ""
    tenantId: "${TENANT_ID}"
EOF
```

8. Run the following test:

```
cat <<EOF | oc apply -f -
kind: Pod
apiVersion: v1
metadata:
  name: busybox-secrets-store-inline
spec:
  containers:
  - name: busybox
    image: image-registry.openshift-image-registry.svc:5000/openshift/tools
    command:
      - "/bin/sleep"
      - "10000"
    volumeMounts:
    - name: secrets-store-inline
      mountPath: "/mnt/secrets-store"
      readOnly: true
  volumes:
    - name: secrets-store-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "azure-kvname"
        nodePublishSecretRef:                       # Only required when using service principal mode
          name: secrets-store-creds                 # Only required when using service principal mode
EOF
```

9. Go to the terminal for the container and run `cd /mnt/secrets-store` and `cat secret1` to view the secret value.

### ACR Integration

Reference List:
- https://docs.microsoft.com/en-us/azure/openshift/howto-use-acr-with-aro
- https://docs.microsoft.com/en-us/azure/container-registry/container-registry-image-tag-version
- https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-azure-cli
- https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-kubernetes

1. Let's create an ACR Instance in the ARO Resource Group per the following commands:

  ```
  # Create an ACR Instance in your resource group with a basic sku
  export RG_NAME=<resource group>
  export ACR_NAME=<acr name>

  az acr create --resource-group $RG_NAME --name $ACR_NAME --sku Basic
  ```

  > Info: Confirm the ACR Instance is in the resource group in the portal.

2. Create a service principal that can authenticate against the ACR Instance:

  ```
  #!/bin/bash

  # Modify for your environment.
  # ACR_NAME: The name of your Azure Container Registry (set in previous step)
  # SERVICE_PRINCIPAL_NAME: Must be unique within your AD tenant
  export SERVICE_PRINCIPAL_NAME=acr-aro-service-principal

  # Obtain the full registry ID for subsequent command args
  export ACR_REGISTRY_ID=$(az acr show --name $ACR_NAME --resource-group $RG_NAME --query id --output tsv)

  # Create the service principal with rights scoped to the registry.
  # Default permissions are for docker pull access. Modify the '--role'
  # argument value as desired:
  # acrpull:     pull only
  # acrpush:     push and pull
  # owner:       push, pull, and assign roles
  SP_PASSWD=$(az ad sp create-for-rbac --name http://$SERVICE_PRINCIPAL_NAME --scopes $ACR_REGISTRY_ID --role acrpull --query password --output tsv)
  SP_APP_ID=$(az ad sp show --id http://$SERVICE_PRINCIPAL_NAME --query appId --output tsv)

  # Output the service principal's credentials; use these in your services and
  # applications to authenticate to the container registry.
  echo "Service principal ID: $SP_APP_ID"
  echo "Service principal password: $SP_PASSWD"
  ```

3. Create an impage pull secret in a test project to get the proper docker config:

  ```
  # Create a test project
  oc new-project test-project-pull-secret

  # Create pull secret in the test project
  oc create secret docker-registry test-pull-secret-acr \
      --namespace test-project-pull-secret \
      --docker-server=$ACR_NAME.azurecr.io \
      --docker-username=$SP_APP_ID \
      --docker-password=$SP_PASSWD
  ```

  > Info: Once the secret is created, navigate to the console and view the .dockerconfigjson associated with the secret. We will add this to the global pull secret in the cluster so we can pull from ACR globally and not need to create this secret in each project we use.

4. Create a file locally so we can update the global secret. Run `touch pullsecret.json`.

  > Note: Ensure this file is not committed since it has your secrets. It should be included in the `.gitignore` file.

5. In the console, go to the `openshift-config` project and copy the pull-secret secret data to the file. It should be formatted as a JSON. From there, navigate to the secret you created in the test project and copy the acr section of that secret into the `pullsecret.json` file. This is how we will add this auth mechanism for the custom ACR instance.

6. Run `oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=pullsecret.json`. Confirm that the secret has been updated in the console.

7. To test and confirm the registry is integrated with the cluster, let's push the app image to our ACR instance. To do this, we will need access to the built-in registry in OpenShift.

8. Run the following commands:

  ```
  # Switch to project "openshift-image-registry"
  oc project openshift-image-registry

  # Expose the registry using "DefaultRoute"
  oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge

  # Note: the value of "Container Registry URL" in the output is the fully qualified registry name.
  HOST=$(oc get route default-route --template='{{ .spec.host }}')
  echo "Container Registry URL: $HOST"
  ```

9. Run `docker login -u kubeadmin -p $(oc whoami -t) $HOST` to login to the registry locally. From here, you can pull the image that was built in prior labs (i.e. run `docker pull $HOST/<PROJECT>/<NAME>:<TAG>`).

10. Run the following commands to tag the image and push it to your ACR instance:

  ```
  # Tag the image locally for the ACR Registry
  docker tag $HOST/<PROJECT>/<NAME>:<TAG> $ACR_NAME.azurecr.io/lab9/node-app:latest

  # Login to the ACR instance locally
  az acr login -n $ACR_NAME -g $RG_NAME

  # Push the image to the ACR Instance
  docker push $ACR_NAME.azurecr.io/lab9/node-app:latest
  ```

  > Info: Confirm in the Azure Portal that the image exists in the ACR Instance.

11. Deploy an instance of the app with this image in ACR by updating the config value for the `image` value in the `charts/app-chart/values-acr.yaml` file. Then run the following:

  ```
  # Create new project
  oc new-project lab-9-acr-deploy

  # Run helm install with custom values file for ACR Image
  helm install test charts/app-chart/. -f charts/app-chart/values-acr.yaml
  ```

  > Info: Confirm the app is up in the console.

### Container Insights with ARO

Reference:
- https://docs.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-azure-redhat4-setup

1. Run the following ARM Template to deploy a Logs Analytics Workspace in the same Resource Group as the ARO Cluster:

```
export RG_NAME=<resource group>

az deployment group create \
  --resource-group $RG_NAME \
  --name test-deployment-log-analytics \
  --template-file log-analytics-deployment.yaml
```

2. Run `az resource list --resource-type Microsoft.OperationalInsights/workspaces -o json -g $RG_NAME` to get the ID of the workspace.

3. Run `az resource list --resource-type Microsoft.RedHatOpenShift/OpenShiftClusters -o json -g $RG_NAME` to get the ID of the cluster.

4. Run the following with the IDs from above:

```
export azureAroV4ClusterResourceId="/subscriptions/<subscriptionId>/resourceGroups/<resourceGroupName>/providers/Microsoft.RedHatOpenShift/OpenShiftClusters/<clusterName>"

export logAnalyticsWorkspaceResourceId="/subscriptions/<subscriptionId>/resourceGroups/<resourceGroupName>/providers/microsoft.operationalinsights/workspaces/<workspaceName>"
```

5. Run `curl -o enable-monitoring.sh -L https://aka.ms/enable-monitoring-bash-script` to download the script.

6. Run `bash enable-monitoring.sh --resource-id $azureAroV4ClusterResourceId --workspace-id $logAnalyticsWorkspaceResourceId`