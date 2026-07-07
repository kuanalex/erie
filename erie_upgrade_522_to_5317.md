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

#### Backup of the cluster is done

Backup your Cloud Pak for Data cluster before the upgrade

**Note:**
Make sure there are no scheduled backups conflicting with the scheduled upgrade

#### The image mirroring completed successfully

Since you are using a private container registry, you must mirror the updated images from the IBM® Entitled Registry to your private container registry at `<YOUR_PRIVATE_REGISTRY>`

#### The CASE files and cluster resource files downloaded successfully

Before upgrading IBM Scheduling, the IBM Software Hub platform, or any services, you must download the required cluster‑scoped resources—such as ClusterRoles and ClusterRoleBindings—for the components you plan to upgrade. Ensure that these files are available on the bastion node for use during the upgrade

For more information, see [Downloading CASE packages](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pruirn-downloading-case-packages-1)


#### The permissions required for the upgrade is ready

- **OpenShift cluster permissions**
  
  An OpenShift cluster administrator can complete all of the installation tasks
  
  However, if you want to enable users with fewer permissions to complete some of the installation tasks, refer to the IBM Documentation about `Reauthorizing an instance administrator with the minimum RBAC to upgrade components`

- **IBM Software Hub permissions**
  
  The Cloud Pak for Data administrator role or permissions is required for upgrading the service instances

- **Registry permissions**
  - Permission to access the private image registry at `<YOUR_PRIVATE_REGISTRY>` for pushing or pulling images

- **Bastion node access**
  - Access to the bastion node for executing the upgrade commands

#### A pre-upgrade health check is made to ensure the cluster's readiness for upgrade

- The OpenShift cluster, persistent storage, IBM Software Hub platform and services are in healthy status

#### Migrating to Red Hat OpenShift certificate manager

The IBM Certificate manager is deprecated

If the IBM Certificate manager (ibm-cert-manager) is installed on your cluster, refer to IBM Documentation to migrate your certificates from the IBM Certificate manager to the Red Hat OpenShift certificate manager (cert-manager Operator)

[Migrating from the IBM Certificate manager to the Red Hat OpenShift certificate manager](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading-migrating-red-hat-openshift-certificate-manager)


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

### Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
${CPDM_OC_LOGIN}
```

### Upgrading the operator and custom resource for the service
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

### Run the cpd-cli manage login-to-ocp command to log in to the cluster
```bash
${CPDM_OC_LOGIN}
```

### Upgrading the operator and custom resource for the service
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
