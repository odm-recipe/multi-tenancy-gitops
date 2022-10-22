# Deploy [Operational Desision Manager](https://www.ibm.com/products/operational-decision-manager)

This recipe is for deploying the Operational Desision Manager in a single namespace (i.e. `tools`): 

### Adding a global pull secret using your [IBM Entitlement Key](https://myibm.ibm.com/products-services/containerlibrary) Cluster wide.
    
```bash
export IBM_ENTITLEMENT_KEY=<IBM.ENTITELMENT.KEY>
```
```bash
oc create secret docker-registry cpregistrysecret -n kube-system \
 --docker-server=cp.icr.io/cp/cpd \
 --docker-username=cp \
 --docker-password=${IBM_ENTITLEMENT_KEY} 
```

> If you want all the pods in your cluster to be able to pull images from the entitled registry, update the pull secret that is in the openshift-config namespace with the Docker account and entitlement key information.
>> Complete the following steps to update the pull secret:


1. Extract the current pull secret that is in the openshift-config namespace:
    ```bash
    oc extract secret/pull-secret -n openshift-config --keys=.dockerconfigjson --to=. --confirm
    ```
1. Convert the extracted pull secret by using jq command-line JSON processor. To install jq, see jq command-line JSON processor Opens in a new tab.
    ```bash
    cat .dockerconfigjson | jq . >  .dockerconfigjson.orig
    mv .dockerconfigjson.orig .dockerconfigjson
    ```
1. Convert your entitlement key to base64. Replace entitlement_key with the value of your entitlement key that you copied earlier from Container Software Library Opens in a new tab. Copy the base64 value to a safe location.
    ```bash
    echo "cp:entitlement_key" | base64
    ```
1. Make the following edits to the .dockerconfigjson file: In the auths section, add cp.icr.io to the existing list of objects. Replace auth_string with the base64 value that you got in the previous step. Important: You must enter the value of auth_string as a single, long string. If there are any line returns, you get an error.
    ```json
    {
    "auths": {
        "cp.icr.io" : {
            "auth": "auth_string"
                     }
        }
    }
    ```
1. Upload the updated pull secret to the openshift-config namespace:
    ```bash
    oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson
    ```
> After the pull secret successfully uploads, you see the following message:
    ```
    secret/pull-secret data updated
    ```
> The update restarts all the nodes in your cluster. Use the following command to monitor the status of the nodes. Wait until all nodes show the status as UPDATED: `True`.
    
```bash
watch -n 3 oc get machineconfigpool
```

### Infrastructure - Kustomization.yaml
1. Edit the Infrastructure layer `${GITOPS_PROFILE}/1-infra/kustomization.yaml`, un-comment the following lines, commit and push the changes and synchronize the `infra` Application in the ArgoCD console.

    ```bash        
    cd multi-tenancy-gitops/0-bootstrap/single-cluster/1-infra
    ```

    ```yaml
    - argocd/consolenotification.yaml
    - argocd/namespace-ibm-common-services.yaml
    - argocd/namespace-sealed-secrets.yaml
    - argocd/namespace-tools.yaml
    - argocd/serviceaccounts-ibm-common-services.yaml
    - argocd/serviceaccounts-tools.yaml
    ```
    >  💡 **NOTE**  
    > Commit and Push the changes for `multi-tenancy-gitops` & go to ArgoCD, open `infra` application and click refresh.
    > Wait until everything gets deployed before moving to the next steps.

### Services - Kustomization.yaml

1. This recipe is can be implemented using a combination of storage classes. Not all combination will work, the following table lists the storage classes that we have tested to work:

    | Component | Access Mode | IBM Cloud | OCS/ODF |
    | --- | --- | --- | --- |
    | DB2 | RWX | ibmc-file-gold-gid | ocs-storagecluster-cephfs |

1. Edit the Services layer `${GITOPS_PROFILE}/2-services/kustomization.yaml` and install Sealed Secrets by uncommenting the following line, **commit** and **push** the changes and refresh the `services` Application in the ArgoCD console.
   
    ```yaml
    ## IBM Foundational Services / Common Services
    - argocd/operators/ibm-foundations.yaml
    - argocd/instances/ibm-foundational-services-instance.yaml
    - argocd/operators/ibm-automation-foundation-core-operator.yaml

    ## IBM Catalogs
    - argocd/operators/ibm-catalogs.yaml

    # Sealed Secrets
    - argocd/instances/sealed-secrets.yaml
    ```

    >  💡 **NOTE**  
    > Commit and Push the changes for `multi-tenancy-gitops` & go to ArgoCD, open `services` application and click refresh.
    > Wait until everything gets deployed before moving to the next steps.

1. Edit the Services layer `${GITOPS_PROFILE}/2-services/kustomization.yaml` and install Sealed Secrets by uncommenting the following line, **commit** and **push** the changes and refresh the `services` Application in the ArgoCD console.
 
> **⚠️** Warning:
>> Make sure that `${GITOPS_PROFILE}/2-services/argocd/instances/ibm-platform-navigator-instance.yaml`
   
```yaml
    storage:
      class: managed-nfs-storage
```  
Then enable Platform Navigator Operator & Instance.  
```yaml
    ## Cloud Pak for Integration
    - argocd/operators/ibm-platform-navigator.yaml
    - argocd/instances/ibm-platform-navigator-instance.yaml
``` 

>  💡 **NOTE**  
> Commit and Push the changes for `multi-tenancy-gitops` & go to ArgoCD, open `services` application and click refresh.
> Wait until everything gets deployed before moving to the next steps.

1. Edit the Services layer `${GITOPS_PROFILE}/2-services/kustomization.yaml` by uncommenting the following line to install Sterling File Gateway, **commit** and **push** the changes and refresh the `services` Application in the ArgoCD console:

    ```yaml
    ## Cloud Pak for Integration
    - argocd/operators/ibm-datapower-operator.yaml
    ```

    >  💡 **NOTE**  
    > Commit and Push the changes for `multi-tenancy-gitops` and
    > sync ArgoCD application `services` layer.

---

### Validation
1.  Check the status of the `CommonService`,`PlatformNavigator` & `Datapower` custom resource.
    ```bash
    # Verify the Common Services instance has been deployed successfully
    oc get commonservice common-service -n ibm-common-services -o=jsonpath='{.status.phase}'
    # Expected output = Succeeded

    # [Optional] If selected, verify the Platform Navigator instance has been deployed successfully
    oc get platformnavigator -n tools -o=jsonpath='{ .items[*].status.conditions[].status }'
    # Expected output = True
    ```
1.  Log in to the Platform Navigator console
    ```bash
    # Retrieve Platform Navigator Console URL
    oc get route -n tools integration-navigator-pn -o template --template='https://{{.spec.host}}'
    # Retrieve admin password
    oc extract -n ibm-common-services secrets/platform-auth-idp-credentials --keys=admin_username,admin_password --to=-
    ```

1. Validate Datapower Operator
    ```bash
    oc get operators datapower-operator.openshift-operators -o=jsonpath='{ .status.components.refs[12].conditions[*].type}'
    # Expected output = Succeeded
    ```  
    or from the console by doing the following steps
    ```bash
    oc console
    ``` 
    Click on `Operators Tab`->`Installed operators` from the menu on the left and validate the status of datapower operator `Succeeded`
    ![DataPower Operator](images/datapower-operator.png)
