# ARO DevOps Labs

## Lab 8: Persistent Volumes

### Test Ephemeral Container

1. To create a DevOps setup that can promote an image across environments, first we will create the different projects in OpenShift and deploy the required resources that will be used to build and run the application.

2. Deploy an instance of the app to the lab-8 project:

    ```
    oc new-project lab-8
    helm install lab-8 charts/app-chart/. -f ./charts/app-chart/values-dev.yaml
    oc start-build node-express-mongo-app --from-dir=express-mongo-app/. --follow
    ```

3. Connect to the running container and create a file:

    ```
    oc get pods
    oc rsh <pod>
    ```

4. In the working directory, create a new file with `touch testfile.txt` and then run `exit` to leave the container.

5. Let's restart the deployment to create a new instance of the container and confirm that the created file does not exist. Run the following:

    ```
    oc rollout restart deployment/node-express-mongo-app

    oc get events -w

    oc rsh <pod>
    ```

### Attach PVC to App

1. Let's create a PVC using the default Azure Disk storage class. Go to the console and get a sample of the YAML syntax for the PVC (you can also use the file in `charts/app-chart/pvc.yaml`).

2. Notice how in the console, the PVC is created, but there is no persistent volume associated with it. View the resource events and you will find that the system is waiting for a consumer. Let's add the volume to our deployment. Run the following from the `charts/app-chart` directory:

    ```
    helm template test . -s templates/deployment.yaml -f values-dev.yaml | oc apply -f -
    ```

3. You should see the PVC get bound to the app in the console. Additionally a PV is created and a backing Azure Disk was created.

4. Run the following to test:

    ```
    oc get pods
    oc rsh <pod>
    cd /var/test && touch testfile.txt
    exit

    oc rollout restart deployment/node-express-mongo-app

    oc get pods
    oc rsh <pod>
    cd /var/test    # you should see testfile.txt
    ```

### Setup Azure Files with ARO

> Info: Sourced from [ARO Microsoft Docs](https://docs.microsoft.com/en-us/azure/openshift/howto-create-a-storageclass)

1. Run the following to create a storage account in Azure:

    ```
    AZURE_FILES_RESOURCE_GROUP=aro_azure_files
    LOCATION=eastus

    az group create -l $LOCATION -n $AZURE_FILES_RESOURCE_GROUP

    AZURE_STORAGE_ACCOUNT_NAME=aroazurefiles<UNIQUE>

    az storage account create \
        --name $AZURE_STORAGE_ACCOUNT_NAME \
        --resource-group $AZURE_FILES_RESOURCE_GROUP \
        --kind StorageV2 \
        --sku Standard_LRS
    ```

2. Run the following to update the service principle permissions in ARO:

```
ARO_RESOURCE_GROUP=aro-rg
CLUSTER=cluster
ARO_SERVICE_PRINCIPAL_ID=$(az aro show -g $ARO_RESOURCE_GROUP -n $CLUSTER --query servicePrincipalProfile.clientId -o tsv)

az role assignment create --role Contributor --assignee $ARO_SERVICE_PRINCIPAL_ID -g $AZURE_FILES_RESOURCE_GROUP
```

3. Run these set of commands to give the persistent volume binder service account the ability to read and create secrets:

```
ARO_API_SERVER=$(az aro list --query "[?contains(name,'$CLUSTER')].[apiserverProfile.url]" -o tsv)

oc login -u kubeadmin -p $(az aro list-credentials -g $ARO_RESOURCE_GROUP -n $CLUSTER --query=kubeadminPassword -o tsv) $ARO_API_SERVER

oc create clusterrole azure-secret-reader \
	--verb=create,get \
	--resource=secrets

oc adm policy add-cluster-role-to-user azure-secret-reader system:serviceaccount:kube-system:persistent-volume-binder
```

4. Finally, create the Storage Class for Azure Files by running this in the same shell as where the prior environment variables were set:

    ```
    cat << EOF >> azure-storageclass-azure-file.yaml
    kind: StorageClass
    apiVersion: storage.k8s.io/v1
    metadata:
    name: azure-file
    provisioner: kubernetes.io/azure-file
    parameters:
    location: $LOCATION
    secretNamespace: kube-system
    skuName: Standard_LRS
    storageAccount: $AZURE_STORAGE_ACCOUNT_NAME
    resourceGroup: $AZURE_FILES_RESOURCE_GROUP
    reclaimPolicy: Delete
    volumeBindingMode: Immediate
    EOF

    oc create -f azure-storageclass-azure-file.yaml
    ```

5. Update the PVC with the storage class name create above (azure-file) and then run through a similar test to confirm the azure file share is mounted.