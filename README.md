# ARO DevOps Labs

## Lab 9: Azure Integrations with ARO

### Azure Key Vault Integration

> Info: [Repo](https://github.com/Azure/secrets-store-csi-driver-provider-azure/tree/master/charts/csi-secrets-store-provider-azure) for the helm chart.

Reference List:
- https://www.openshift.com/blog/managing-sccs-in-openshift
- https://github.com/aramase/secrets-store-csi-driver-provider-azure/blob/openshift-docs/website/content/en/configurations/deploy-in-openshift.md
- https://azure.github.io/secrets-store-csi-driver-provider-azure/demos/standard-walkthrough/
- https://secrets-store-csi-driver.sigs.k8s.io/topics/best-practices.html

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
expor SERVICE_PRINCIPAL_CLIENT_SECRET=<password from output of prior command>
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

