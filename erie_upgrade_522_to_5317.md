# IBM Software Hub Upgrade from 5.2.2 to 5.3.1.7
# Author: Alex Kuan (alex.kuan@ibm.com)

## Upgrade Context
- **OCP:** 4.17
- **SWH:** 5.2.2 → 5.3.1.7
- **FileStorageClass:** ontap-nas
- **BlockStorageClass:** ontap-nas
- **Components:** cpd_platform, datastage_ent_plus, ws_pipelines
- **PrivateImageRegistry:** Yes

## Table of Contents
1. [Pre-upgrade Tasks](#pre-upgrade-tasks)
2. [Upgrade Execution](#upgrade-execution)
3. [Post-upgrade Tasks](#post-upgrade-tasks)

## Pre-requisites

### Backup of the cluster is done

Backup your Cloud Pak for Data cluster before the upgrade

**Note:**
Make sure there are no scheduled backups conflicting with the scheduled upgrade

### The image mirroring completed successfully

Since you are using a private container registry, you must mirror the updated images from the IBM® Entitled Registry to your private container registry at `<YOUR_PRIVATE_REGISTRY>`

### The CASE files and cluster resource files downloaded successfully

Before upgrading IBM Scheduling, the IBM Software Hub platform, or any services, you must download the required cluster‑scoped resources—such as ClusterRoles and ClusterRoleBindings—for the components you plan to upgrade. Ensure that these files are available on the bastion node for use during the upgrade

For more information, see [Downloading CASE packages](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pruirn-downloading-case-packages-1)


### The permissions required for the upgrade is ready

- **OpenShift cluster permissions**
  
  An OpenShift cluster administrator can complete all of the installation tasks
  
  However, if you want to enable users with fewer permissions to complete some of the installation tasks, refer to the IBM Documentation about `Reauthorizing an instance administrator with the minimum RBAC to upgrade components`

- **IBM Software Hub permissions**
  
  The Cloud Pak for Data administrator role or permissions is required for upgrading the service instances

- **Registry permissions**
  - Permission to access the private image registry at `<YOUR_PRIVATE_REGISTRY>` for pushing or pulling images

- **Bastion node access**
  - Access to the bastion node for executing the upgrade commands

### A pre-upgrade health check is made to ensure the cluster's readiness for upgrade

- The OpenShift cluster, persistent storage, IBM Software Hub platform and services are in healthy status

### Migrating to Red Hat OpenShift certificate manager

The IBM Certificate manager is deprecated

If the IBM Certificate manager (ibm-cert-manager) is installed on your cluster, refer to IBM Documentation to migrate your certificates from the IBM Certificate manager to the Red Hat OpenShift certificate manager (cert-manager Operator)

[Migrating from the IBM Certificate manager to the Red Hat OpenShift certificate manager](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading-migrating-red-hat-openshift-certificate-manager)

### Backing up your existing certificates

Before you uninstall the IBM Certificate manager, back up the Issuer and Certificate custom resources

Create a temporary project where you can validate that the IBM Certificate manager is working correctly
```bash
oc new-project cert-mgr-test
```

Back up the Issuer custom resources to a file named issuer_list.yaml
```bash
oc get issuers.cert-manager.io -A -o yaml > issuer_list.yaml
```

Back up the Certificate custom resources to a file named certificate_list.yaml
```bash
oc get certificates.cert-manager.io -A -o yaml > certificate_list.yaml
```

Verify that IBM Certificate manager is working correctly

Create an Issuer custom resource called verify-issuer
```bash
cat <<EOF |oc apply -f -
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: verify-issuer
spec:
  selfSigned: {}
EOF
```

Apply the contents of the issuer_list.yaml file
```bash
oc apply -f issuer_list.yaml
```

Create a Certificate custom resource called verify-certificate
```bash
cat <<EOF |oc apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: verify-certificate
spec:
  commonName: verify-certificate
  secretName: verify-secret
  issuerRef:
    name: verify-issuer
    kind: Issuer
    group: cert-manager.io
EOF
```

Apply the contents of the certificate_list.yaml file
```bash
oc apply -f certificate_list.yaml
```

Verify that the verify-certificate custom resource is ready
```bash
oc get issuers.cert-manager.io
```

### Uninstall IBM Certificate manager 

Before you can install the Red Hat® OpenShift® certificate manager (cert-manager Operator), you must uninstall the IBM Certificate manager

Get the list of the IBM Certificate manager configuration instances
```bash
oc get certmanagerconfig -n ${PROJECT_CERT_MANAGER}
```

Delete each configuration instance returned by the preceding command
```bash
oc delete certmanagerconfig <name> -n ${PROJECT_CERT_MANAGER}
```

Uninstall the IBM Certificate manager operator

Delete the operator subscription:
```bash
oc delete sub ibm-cert-manager-operator -n ${PROJECT_CERT_MANAGER}
```

Find any ibm-cert-manager cluster service versions (CSVs)
```bash
oc get csv -n ${PROJECT_CERT_MANAGER} | grep ibm-cert-manager
```

Delete the CSVs returned by the previous command
```bash
oc delete csv <name> -n ${PROJECT_CERT_MANAGER}
```

Verify that the following IBM Certificate manager resources are deleted

Check for any deployments with the app.kubernetes.io/component=cert-manager label
```bash
oc get deployments -n ${PROJECT_CERT_MANAGER} -l app.kubernetes.io/component=cert-manager
```

If any deployments are returned by the preceding command, delete them
```bash
oc delete deployments <name> -n ${PROJECT_CERT_MANAGER}
```

Check for any services with the following app=ibm-cert-manager-webhook label
```bash
oc get service -n ${PROJECT_CERT_MANAGER} -l app=ibm-cert-manager-webhook
```

If any services are returned by the preceding command, delete them
```bash
oc delete service <name> -n ${PROJECT_CERT_MANAGER}
```

Check for any cert-manager-webhook mutating web hook configurations
```bash
oc get mutatingwebhookconfiguration -n ${PROJECT_CERT_MANAGER} | grep cert-manager-webhook
```

If any mutating web hook configurations are returned by the preceding command, delete them
```bash
oc delete mutatingwebhookconfiguration <name> -n ${PROJECT_CERT_MANAGER}
```

Check for any cert-manager-webhook validating web hook configurations
```bash
oc get validatingwebhookconfiguration -n ${PROJECT_CERT_MANAGER} | grep cert-manager-webhook
```

If any validating web hook configurations are returned by the preceding command, delete them
```bash
oc delete validatingwebhookconfiguration <name> -n ${PROJECT_CERT_MANAGER}
```

### Mirroring Red Hat OpenShift certificate manager images to a private container registry

[If your cluster pulls images from a private container registry, you must mirror the Red Hat OpenShift certificate manager images to your private container registry before you install the certificate manager.](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=manager-mirroring-red-hat-openshift-certificate-images)

### Installing the Red Hat OpenShift Container Platform cert-manager Operator

[You must install Red Hat OpenShift Container Platform cert-manager Operator before you upgrade to IBM Software Hub Version 5.3](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=manager-installing-cert-operator)

Verify that the OLM subscription is created by running the following command
```bash
oc get subscription -n cert-manager-operator
```

Example output
```bash
NAME                              PACKAGE                           SOURCE             CHANNEL
openshift-cert-manager-operator   openshift-cert-manager-operator   redhat-operators   stable-v1
```

Verify whether the Operator is successfully installed by running the following command
```bash
oc get csv -n cert-manager-operator
```

Example output
```bash
NAME                            DISPLAY                                       VERSION   REPLACES                        PHASE
cert-manager-operator.v1.13.0   cert-manager Operator for Red Hat OpenShift   1.13.0    cert-manager-operator.v1.12.1   Succeeded
```

Verify that the status cert-manager Operator for Red Hat OpenShift is Running by running the following command
```bash
oc get pods -n cert-manager-operator
```

Example output
```bash
NAME                                                        READY   STATUS    RESTARTS   AGE
cert-manager-operator-controller-manager-695b4d46cb-r4hld   2/2     Running   0          7m4s
```

Verify that the status of cert-manager pods is Running by running the following command
```bash
oc get pods -n cert-manager
```

Example output
```bash
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-58b7f649c4-dp6l4              1/1     Running   0          7m1s
cert-manager-cainjector-5565b8f897-gx25h   1/1     Running   0          7m37s
cert-manager-webhook-9bc98cbdd-f972x       1/1     Running   0          7m40s
```


# Pre-upgrade

**Note:**
Sourcing the latest environment variables used by this environment before proceeding with the following procedures. For more information, see [Updating your environment variables script](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=cri-updating-your-environment-variables-script-1)
```bash
source ./cpd_vars.sh
```

## Pre-upgrade check

### Checking the health of your cluster

Check nodes, cluster operators, machine config pools
```bash
oc get nodes,co,mcp
```

Check CR status
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Check for pods not running correctly
```bash
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed'
```

### Checking the known issues before the upgrade

- [Known issues and limitations for IBM Software Hub](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=overview-known-issues-limitations)

## Updating the IBM Software Hub command-line interface

### Updating the IBM Software Hub command-line interface

Update the cpd-cli utility to the latest version of 5.3.x (v14.3.1.7). For detailed documentation and instructions, see [Update the cpd-cli utility](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=workstations-updating-software-hub-cli)

### Obtaining the olm-utils-v4 image

All IBM Software Hub customers are entitled to use the olm-utils-v4 image

The cpd-cli uses podman to pull and manage the olm-utils-v4 container image

When the workstation is connected to the internet, run the following command to update the olm-utils-v4 image on the workstation
```bash
cpd-cli manage restart-container
```

Wait for the cpd-cli to return the following messages
```bash
[SUCCESS] ... Successfully pulled the container image icr.io/cpopen/cpd/olm-utils-v4:${VERSION}
[SUCCESS] ... Successfully started the container olm-utils-play-v4
[SUCCESS] ... Container olm-utils-play-v4 has been re-created
```

The version of the olm-utils-v4 image should be the same as the version of IBM Software Hub that you plan to install

For more information, see [Obtaining the olm-utils-v4 image](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pruirn-obtaining-olm-utils-v4-image-1)


### Installing Helm CLI

[Installing Helm](https://www.ibm.com/links?url=https%3A%2F%2Fhelm.sh%2Fdocs%2Fintro%2Finstall%2F)

# Upgrade

## Updating your environment variables script

**Important:** Ensure that your environment variables script includes the correct information for the instance of IBM Software Hub that you want to upgrade

[Updating your environment variables script](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=cri-updating-your-environment-variables-script)

### Editing your environment variables file

Open your existing environment variable shell script in a text editor

Locate the `VERSION` entry and specify the version of IBM Software Hub that you want to upgrade to
```bash
export VERSION=5.3.1
```

Set or update the `PATCH_ID` environment variable based on the patch that you want to install
```bash
export PATCH_ID=7
```


# Upgrade

## Upgrading the License Service

### Get the project of the License service

If you're not sure which project the License Service is in, run the following command
```bash
oc get deployment -A | grep ibm-licensing-operator
```

### Log in to the Red Hat OpenShift Container Platform cluster
```bash
${CPDM_OC_LOGIN}
```

### Upgrading the License Service
```bash
cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--license_acceptance=true \
--licensing_ns=${PROJECT_LICENSE_SERVICE}
```

Confirm that the License Service pods are Running or Completed
```bash
oc get pods --namespace=${PROJECT_LICENSE_SERVICE}
```

## Preparing to upgrade IBM Software Hub

### Updating the cluster-scoped resources for the platform and services

Generate cluster-scoped resources for platform and services
```bash
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cluster_resources=true
```

Run the 'oc apply -f' command returned in the terminal, for example
```bash
oc apply -f /cpd-cli-workspace/olm-utils-workspace/work/cluster_scoped_resources.yaml --server-side --force-conflicts
```

### Creating image pull secrets for an instance of IBM Software Hub

Log in to Red Hat® OpenShift® Container Platform as a user with sufficient permissions to complete the task
```bash
${OC_LOGIN}
```

Create a file named dockerconfig.json based on where your cluster pulls images from

For Private container registry
```bash
cat <<EOF > dockerconfig.json 
{
  "auths": {
    "${PRIVATE_REGISTRY_LOCATION}": {
      "auth": "${IMAGE_PULL_CREDENTIALS}"
    }
  }
}
EOF
```


Create the image pull secret in the operators project for the instance
```bash
oc create secret docker-registry ${IMAGE_PULL_SECRET} --from-file ".dockerconfigjson=dockerconfig.json" --namespace=${PROJECT_CPD_INST_OPERATORS}
```

Create the image pull secret in the operands project for the instance
```bash
oc create secret docker-registry ${IMAGE_PULL_SECRET} --from-file ".dockerconfigjson=dockerconfig.json" --namespace=${PROJECT_CPD_INST_OPERANDS}
```

## Upgrading IBM Software Hub


### Run the cpd-cli manage login-to-ocp command to log in to the cluster
```bash
${CPDM_OC_LOGIN}
```

### Upgrading the required operators and custom resources for the instance
```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=cpd_platform \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--run_storage_tests=false \
--upgrade=true
```

Once the above command `cpd-cli manage install-components` is completed, make sure the status of the IBM Software Hub is in 'Completed' status
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=cpd_platform
```

### Applying the RSI patches

Run the following command to re-apply your existing custom patches
```bash
cpd-cli manage apply-rsi-patches --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Check the RSI patches status again:
```bash
cpd-cli manage get-rsi-patch-info --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --all
```

## Upgrading DataStage Enterprise Plus

Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
${CPDM_OC_LOGIN}
```

Remove hotfix image_digests from CCS prior to starting the DataStage upgrade
```bash
oc patch ccs ccs-cr -n cpd-instance --type=json -p='[{"op": "remove", "path": "/spec/image_digests"}]'
```

Upgrading the operator and custom resource for the service
```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=datastage_ent_plus \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```

Once the above command `cpd-cli manage install-components` completed successfully, you can run the `cpd-cli manage get-cr-status` command for the validation
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=datastage_ent_plus
```

## Upgrading Orchestration Pipelines

Run the cpd-cli manage login-to-ocp command to log in to the cluster
```bash
${CPDM_OC_LOGIN}
```

Upgrading the operator and custom resource for the service
```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=ws_pipelines \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```

Once the above command `cpd-cli manage install-components` completed successfully, you can run the `cpd-cli manage get-cr-status` command for the validation
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=ws_pipelines
```

### Upgrade the cpdbr service

Export the OADP_OPERATOR_NS environment variable
```bash
export OADP_OPERATOR_NS=<oadp-operator-project>
```

Upgrade the cpdbr-tenant component for the instance for NetApp Trident Protect without the scheduling service
```bash
cpd-cli oadp install \
--component=cpdbr-tenant \
--cpdbr-hooks-image-prefix=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd \
--cpfs-image-prefix=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpfs \
--namespace=${OADP_OPERATOR_NS} \
--tenant-operator-namespace=${PROJECT_CPD_INST_OPERATORS} \
--skip-recipes=true \
--upgrade=true \
--log-level=debug \
--verbose
```

# Post-upgrade Validation

Check CR status
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Check for pods not running correctly
```bash
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed'
```

**End of runbook**
