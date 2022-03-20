# Data-Virtualization (DV) Installation
Companies often try to break down silos by copying disparate data for analysis into central data stores, such as data marts, data warehouses and data lakes. This is costly and prone to error when most manage an average of 400 unique data sources for business intelligence. With data virtualization, you can access data at the source without moving data, accelerating time to value with faster and more accurate queries. In this article, I have documented step-by-step instructions to deploy IBM Data Virtualization on IBM Cloud Pak for Data running on Red Hat OpenShift in Azure. 

# Assumptions
 - Red Hat OpenShift cluster has access to a high-speed internet connection and can pull images directly from IBM Entitled Registry. If not set up yet, please follow the instructions provided here.
 - IBM Cloud Pak for data Control Plane, Foundational Services are installed and running. If not, please follow the instructions provided here.
 - IBM Cloud Pak for data operator is installed in the “ibm-common-services” namespace, and foundational services are installed in the “cpd-instance” namespace.
 - DV operator will be installed in the “ibm-common-services” namespace, and the DV service will be installed in the “cpd-instance” namespace.
 - Installing for demo purposes and so, the latest version of the software will automatically install on the Red Hat OpenShift cluster.
User has knowledge and experience managing Red Hat OpenShift cluster

# Pre-Requisite
 - Red Hat OpenShift cluster version 4.6 or later with min 64 vCPU and 256 GB RAM
 - Bastion host with two vCPU and 4GB RAM with Linux OS
 - Internet access for Bastion host and Red Hat OpenShift cluster
 - OpenShift Container Storage (OCS) attached to Red Hat OpenShift cluster. This link will help you determine supported storage. In this demo, I have used OCS Storage.
 - A User with OpenShift Cluster and Project Administrator access

# Step 1: Download files from the GitHub repo using the following command.

        git clone https://github.com/kapilrajyaguru/Data-Virtualization.git

  After downloading files, switch to the Watson-Knowledge-Catalog-Installation directory.
        
        cd Data-Virtualization/

# Step 2 - Creating an operator subscription for services

 - Create the Db2U operator subscription by running the following command

        oc apply -f db2u-operator.yaml
 
 - Validate that the operator was successfully created.
   
   Run the following command to confirm that the subscription was triggered:

        oc get sub -n ibm-common-services ibm-db2u-operator -o jsonpath='{.status.installedCSV} {"\n"}'

   Verify that the command returns db2u-operator.v1.1.11.
	
 - Run the following command to confirm that the cluster service version (CSV) is ready:
        
        oc get csv -n ibm-common-services db2u-operator.v1.1.11 -o jsonpath='{ .status.phase } : { .status.message} {"\n"}'

  Verify that the command returns Succeeded : install strategy completed with no errors.

 - Run the following command to confirm that the operator is ready:

        oc get deployments -n ibm-common-services -l olm.owner="db2u-operator.v1.1.11" -o jsonpath="{.items[0].status.availableReplicas} {'\n'}"

  Verify that the command returns an integer greater than or equal to 1. If the command returns 0, wait for the deployment to become available.

# Step 3 - Create the Data Virtualization operator subscription.
  
        oc apply -f dv-operator-sub.yaml
 
 - Validate that the operator was successfully created.
   For each command, ensure that you specify the appropriate Red Hat OpenShift project (either ibm-common-services or cpd-operators) for the --namespace (-n) argument.
  Run the following command to confirm that the subscription was triggered:

        oc get sub -n ibm-common-services ibm-dv-operator-catalog-subscription -o jsonpath='{.status.installedCSV} {"\n"}'

   Verify that the command returns ibm-dv-operator.v1.7.6.

 - Run the following command to confirm that the cluster service version (CSV) is ready:

        oc get csv -n ibm-common-services ibm-dv-operator.v1.7.6 -o jsonpath='{ .status.phase } : { .status.message} {"\n"}'

   Verify that the command returns Succeeded : install strategy completed with no errors.

 - Run the following command to confirm that the operator is ready:

        oc get deployments -n ibm-common-services -l olm.owner="ibm-dv-operator.v1.7.6" -o jsonpath="{.items[0].status.availableReplicas} {'\n'}"

   Verify that the command returns an integer greater than or equal to 1. If the command returns 0, wait for the deployment to become available.

# Step 4 - CRI-O Container Settings

  If you have already installed Watson Knowledge Catalog and then you have step 4 & 5 performed. So, no need to perform it again and can directly jump to Step 6.

 - Copy crio.conf to /tmp directory

        cp crio.conf /tmp/

 - Log in to Red Hat open shift in the command line. Use cloned machineconfig object YAML file, as follows, and apply it.
   Note: If you are using Cloud Pak for Data on OpenShift Container Platform version 4.6, the ignition version is 3.1.0. If you are using Cloud Pak for Data on OpenShift Container Platform version 4.8, change the ignition version to 3.2.0 in the machineconfig.yaml

        oc apply -f machineconfig.yaml

  The above action will reboot your cluster nodes one by one. Monitor all of the nodes to ensure that the changes are applied by using the following command:

        watch oc get nodes

  You can also use the following command to confirm that the MachineConfig sync is complete:

        watch oc get mcp

# Step 5: Kernel Parameter Settings
  The following step will enable unsafe sysctls by configuring kubelet to allow Db2U to make unsafe sysctls calls for db2 to manage required memory settings.
Update all of the nodes to use a custom KubletConfig:

        oc apply -f kubeletconfig.yaml
  
  Update the label on the machineconfigpool:

        oc label machineconfigpool worker db2u-kubelet=sysctl
  
  Wait for the cluster to restart and then run the following command to verify that the machineconfigpool is updated:

        oc get machineconfigpool

  Next, wait until all of the worker nodes are updated and ready.

# Step 6 - Create a DvService custom resource to install Data Virtualization.

  Important: By creating a DvService custom resource with spec.license.accept: true, you are accepting the license terms for Data Virtualization. You can find links to the relevant licenses in IBM Cloud Pak for Data License Information.
Create a custom resource with the following format.

        oc apply -f dv-service.yaml

  When you create the custom resource, the Data Virtualization operator installs Data Virtualization.
  
 - Get the status of Data Virtualization (dv-service):
   Run the following command:

		oc get dvservice dv-service

   The result is similar to the following example, where the READY field indicates whether the DvService is installed.

		NAME         READY
		dv-service   True

 - To check whether the DvService has finished installing Data Virtualization service pods, run the following command:

		oc get DvService dv-service -o jsonpath="{.status.reconcileStatus}"

   Data Virtualization is installed when the command returns Completed. You must now provision a Data Virtualization instance to use Data Virtualization.

# Step 7 - Provision the Data Virtualization service:

 - On the navigation menu, click Services > Instances.
 - From the list of instances, locate the Data Virtualization service, click the action menu, and select Provision instance.
 - To configure the service, specify the resources that you want to allocate to the Data Virtualization worker nodes in the Nodes step.
 - Specify the number of Data Virtualization worker nodes to allocate to the service.
    Recommended: One worker node is sufficient for many workloads.
 - Specify the number of cores to allocate per node.
    You are constrained by the total number of available cores on the OpenShift® compute nodes.
 - Specify the amount of memory in GB to allocate per node.
    You are constrained by the total amount of memory on the OpenShift compute nodes.
 - You can scale the Data Virtualization service up and down at any time after you provision it. For more information, see Scaling Data Virtualization.
 - In the Storage step, specify the storage classes and persistent volume sizes that you want to use for the service nodes and caching storage. For more information, see Storage requirements.
 - In the Node storage section, select the storage class and specify the size to allocate to your nodes. The default size shown in the Node storage section is 50Gi.
   The term worker pod in Data Virtualization refers to a c-db2u-dv-db2u-x pod that runs one Data Virtualization worker component, where x starts at 1. You can allocate multiple worker components, which are effectively multiple c-db2u-dv-db2u-x pods, to the Data Virtualization service instance.
 - In the Caching storage section, select the storage class and specify the amount of storage to allocate to your data caches
   Note: Part of the total cache storage space is used for refreshing active caches that have a periodic refresh schedule. This refresh schedule impacts the storage space that is available for creating new cache entries.
 - Click Next.
 - Ensure that the summary is correct and click Configure.
   Wait for the service to be provisioned. This might take some time because of the number of components that must start up.
 - Optional: If you want to use Cloud Pak for Data while you wait for the Data Virtualization provisioning process to complete, click Home.
