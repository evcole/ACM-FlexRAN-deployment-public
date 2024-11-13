# FlexRAN ACM Managed Cluster SNO Deployment using Zero Touch Provisioning
In this guide, a managed cluster of Single Node OpenShift 4.12 is deployed using Red Hat Zero Touch Provisioning (ZTP). It will be provisioned as a Telco DU node, ready for the Intel FlexRAN L1 application. Zero Touch Provisioning will deploy and deliver OpenShift 4 in an automated fashion, where a single hub cluster manages many managed clusters. The hub cluster will manage, deploy, and control the lifecycle of the managed clusters using Red Hat Advanced Cluster Management (RHACM).

## Prerequisites  
Requirements:  
- A pre-existing ACM hub cluster
- A Git repository for the managed cluster
- ArgoCD

## Creating the ArgoCD application
To begin, login to the ACM hub cluster on an account with cluster admin.  

From the command line, create the secret containing the credentials for your Git repository.

```
oc create -n openshift-gitops secret generic intel-hub-repo --from-literal=username=<USERNAME> --from-literal=password=<TOKEN> --from-literal=type=git --from-literal=url=<REPOSITORY_URL> --from-literal=insecure=true
```

After creating the secret, label it with *argocd.argoproj.io/secret-type=repository* so that ArgoCD can access the repository.  

```
oc label -n openshift-gitops secret intel-hub-repo argocd.argoproj.io/secret-type=repository
```

Edit deployment/clusters-app.yaml with your application and project names, and the URL to your Git repository.

```
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: flexran-ztp-clusters
  namespace: openshift-gitops
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: clusters-sub
  project: ztp-app-project
  source:
    path: siteconfig
    repoURL: https://github.com/evcole/ACM-FlexRAN-deployment-public
    targetRevision: main
    # uncomment the below plugin if you will be adding the plugin binaries in the same repo->dir where
    # the sitconfig.yaml exist AND use the ../../hack/patch-argocd-dev.sh script to re-patch the deployment-repo-server
#    plugin:
#      name: kustomize-with-local-plugins
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

Now, create the ArgoCD app from the command line, when logged in to the ACM hub cluster:
```
oc apply -f deployment/clusters-app.yaml
```

In your ArgoCD interface, verify that the project/application has been created and has access to your Git repository.  

## Configuring the Siteconfig
The majority of the edits needed to be made to deploy your managed cluster are within *siteconfig/siteconfig.yaml*. Edit the siteconfig to contain the correct information for your cluster deployment, including the cluster image set, cluster labels, and node networking information. For more information, see Step 4 of [Generating ZTP installation and configuration CRs manually](https://docs.openshift.com/container-platform/4.12/scalability_and_performance/ztp_far_edge/ztp-manual-install.html#ztp-generating-install-and-config-crs-manually_ztp-manual-install).  


## Pushing changes to the repository  
After making changes to files tracked in the Git repository, it is necessary to push the changes to the repository and sync on ArgoCD. To do this, follow the procedure:  

1. Edit, change, or add files in the ArgoCD-tracked Git repository
2. If these changes included any changes to the siteconfig, edit *siteconfig/kustomization.yaml* and comment out each YAML file in the Kustomization. This will ensure that ArgoCD properly updates the resources.
3. Add, commit, and push the changes to the Git repository
4. In the ArgoCD application, manually Refresh/Sync with Prune enabled. Wait for ArgoCD's status to update to Sync OK with no errors.
5. If you commented out lines in *siteconfig/kustomization.yaml*, you must now uncomment those lines. Add, commit, and push the *kustomization.yaml* changes to the repository. Once done, return to the ArgoCD application and sync again, waiting for Sync OK with no errors.
6. Now, the managed cluster name should appear in the ACM hub cluster's Host Inventory. Verify that it exists.
7. Download your pull secret into a file named *config.json*.
8. Create the secret for your pull secret, using the --from-file flag to specify config.json as the source. Make sure to create the secret in the same namespace used for the rest of your cluster's resources. Once the secret is created, you should see the "Pull Secret" checkmark come up in the Hosts Inventory page, and the status for the host should switch to "Provisioning."
```
oc create secret generic pull-secret --from-file=.dockerconfigjson=config.json --type=kubernetes.io/dockerconfigjson -n ocp4clustername
```
9. Check the hub cluster's Host Inventory, and wait for the cluster's status to say "Provisioned."
10. To monitor the cluster's installation status, you can view the machine's virtual console to verify no errors were received. Once the machine is done booting, you can view the new managed cluster from the ACM hub cluster to verify it is properly configured and running.

## Troubleshooting deployment
Check the status of the managed cluster:  
```
oc get managedcluster
```

Check the agentclusterinstall status:  
```
oc get clusterdeployment -n ocp4clustername
```

If it failed, review the status of the AgentClusterInstall resource to try to resolve the errors:
```
oc describe agentclusterinstall -n ocp4clustername ocp4clustername
```

If the cluster or ArgoCD application is stuck on deleting or terminating, check which resources or projects in the cluster are failing to delete. To learn more about what may be impeding proper termination, get the output of the project in question:  

```
oc get project openshift-storage -o yaml
```

The following example output can give information on which resources still need to be removed.
```
status:
  conditions:
  - lastTransitionTime: "2020-07-26T12:32:56Z"
    message: All resources successfully discovered
    reason: ResourcesDiscovered
    status: "False"
    type: NamespaceDeletionDiscoveryFailure
  - lastTransitionTime: "2020-07-26T12:32:56Z"
    message: All legacy kube types successfully parsed
    reason: ParsedGroupVersions
    status: "False"
    type: NamespaceDeletionGroupVersionParsingFailure
  - lastTransitionTime: "2020-07-26T12:32:56Z"
    message: All content successfully deleted, may be waiting on finalization
    reason: ContentDeleted
    status: "False"
    type: NamespaceDeletionContentFailure
  - lastTransitionTime: "2020-07-26T12:32:56Z"
    message: 'Some resources are remaining: cephobjectstoreusers.ceph.rook.io has
      1 resource instances'
    reason: SomeResourcesRemain
    status: "True"
    type: NamespaceContentRemaining
  - lastTransitionTime: "2020-07-26T12:32:56Z"
    message: 'Some content in the namespace has finalizers remaining: cephobjectstoreuser.ceph.rook.io
      in 1 resource instances'
    reason: SomeFinalizersRemain
    status: "True"
    type: NamespaceFinalizersRemaining
```

In the previous step, we can see that the object *cephobjectstoreuser.ceph.rook.io* has remaining resources to be deleted. Get the Object name so that we can patch it to remove the finalizers:  
```
oc get cephobjectstoreusers.ceph.rook.io -n openshift-storage

===EXAMPLE OUTPUT===
NAME                           AGE
noobaa-ceph-objectstore-user   26h
===EXAMPLE OUTPUT===
```

Now, patch the resources:  
```
oc patch -n openshift-storage cephobjectstoreusers.ceph.rook.io/noobaa-ceph-objectstore-user --type=merge -p '{"metadata": {"finalizers":null}}'
```

Verify the project has been deleted:  
```
oc get project openshift-storage
```
