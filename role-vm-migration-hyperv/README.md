# VM Migration via AWX – Step-by-Step Execution Guide

## 1. Purpose
This document provides a standardized process to execute the VM migration playbook from AWX for migrating VMs from Hyper-V to OpenShift. It ensures consistent, reliable, and auditable migrations.

## 2. Scope
- Applies to all migrations initiated through AWX using the VM migration Ansible playbook.  
- Covers Linux and Windows VM migration workflows.  

## 3. Prerequisites

### 3.1 Infrastructure
- The AWX instance is operational and accessible.  
- WinRM is installed and enabled on the Hyper-V host.  
- The source VM is powered off or in a migration-ready state.  
- The source VM does not have any checkpoints.  
- The source Hyper-V has sufficient space to accommodate a copy of the VM we are trying to migrate.  
- The destination OpenShift cluster is reachable and has sufficient PVC capacity.  
- Place oc, virtctl and qemu commands under `C:\Software\Openshift-Tool` on all other Hyper-V hosts in the cluster.

### 3.2 Credentials
- Hyper-V administrative credentials stored securely in the AWX Credential Store.  
- OpenShift API token/cluster-admin-level credential stored in the AWX Credential Store.


## 4. Execution Steps

### Step 1.1 – Convert Disks to Dynamic on Hyper-V
- **Note:** For Windows VMs, install virtio driver.  
- Put the VM in maintenance mode on the monitoring system.  
- Ensure a full VM backup is taken in backup tools for a safer side.  
- Shut down the VM on the Hyper-V cluster.  
- Ensure all disks are dynamically provisioned. If not, convert thick provisioned disks to thin on the source VM.

### Step 1.2 – Log in to AWX
1. Open a browser and navigate to the AWX web interface: 
2. Enter your AWX username and password.  
3. Click **Sign In**.

### Step 2 – Select the Migration Template
1. In the left navigation panel, click **Templates**.  
2. Search for and select **Openshift-HyperV-to-OC-Migration-from-Hyper-V**.  
3. Verify that:  
   - Playbook: `OpenShift/role-vm-migration-win/master.yml` is selected.  
   - Execution Environment is correct.  
   - Credentials (Hyper-V & OpenShift) are assigned.  
   - Select appropriate inventory with all Hyper-V hosts

### Step 3 – Launch the Migration
1. Click the **Launch** button (rocket icon).  
2. A prompt window appears for extra variables.  
3. Fill in the required variables:

{
  "export_dir": "C:\\ClusterStorage\\export",
  "biosmode": "UEFI",
  "vmname": "testvm",
  "ocp_namespace": "vm-migrations"
}

Enter Source VM Name, data store location with sufficient space, namespace based on environment, BIOS mode based on source VM.

Validate the job once and launch.
### Step 4 – Monitor Job Execution

The Job Details page opens automatically.
Monitor the playbook execution log in real time.
## Ensure the following tasks succeed:
VM existence check on Hyper-V host
PVC creation on OpenShift
Disk image conversion (qemu-img process)
Data transfer completion
VM manifest application in OpenShift

### Step 5 – Post-Migration Validation

Log in to the OpenShift Web Console.
Navigate to Virtualization → Virtual Machines.
Verify the migrated VM appears in the correct namespace.
Confirm PVCs are bound and the VM can be powered on.
Assign Network interface and then set IP address via console. 
Perform a basic functionality test (ping, application startup).
