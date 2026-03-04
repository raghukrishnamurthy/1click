# Cisco UCS 1Click Deployment

This repository contains Ansible playbooks for automating Cisco UCS C-Series standalone server management tasks, including password reset, DNS configuration, claiming servers to Cisco Intersight Private Virtual Appliance (PVA), onboarding to Nutanix Foundation Central, deploying AHV on the nodes and creeating AOS cluster.

## 📋 Table of Contents

- [Overview](#overview)
- [What's New in v2.0](#whats-new-in-v20)
- [Prerequisites](#prerequisites)
- [Inventory Configuration](#inventory-configuration)
- [Playbooks](#playbooks)
  - [1. CIMC Password Reset](#1-cimc-password-reset)
  - [2. DNS Configuration via Device Connector](#2-dns-configuration-via-device-connector)
  - [3. Claim Server to Intersight PVA](#3-claim-server-to-intersight-pva)
  - [4. Onboard Servers to Foundation Central](#4-onboard-servers-to-foundation-central)
  - [5. Deploy Nodes via Foundation Central](#5-deploy-nodes-via-foundation-central)
- [Complete Workflow: End-to-End Automation](#complete-workflow-end-to-end-automation)
- [Playbook Execution Order](#playbook-execution-order)
- [Inventory Files Reference](#inventory-files-reference)
- [Troubleshooting](#troubleshooting)
- [Version History](#version-history)

---

## 🎯 Overview

These playbooks automate common operational tasks for Cisco UCS C-Series standalone servers and integration with management platforms:

### CIMC Management Playbooks:
1. **cimc_password_reset.yml** - Resets CIMC admin password using Redfish API
2. **add_dns_is_device_connector.yml** - Configures DNS settings via Device Connector API with automatic session cookie authentication

### Integration Playbooks:
3. **claim_server_intersight.yml** - Claims servers to Cisco Intersight PVA with workflow validation
4. **onboard_server_foundation_central.yml** - Bulk onboards multiple nodes to Nutanix Foundation Central from Intersight
5. **node_deployment_foundation_central.yml** - Deploys onboarded nodes as Nutanix clusters via Foundation Central

### Key Features:
- ✅ **Parallel execution support** for multi-server operations
- ✅ **Automatic cookie-based authentication** for CIMC APIs
- ✅ **Workflow validation** for Intersight device registration
- ✅ **Bulk node onboarding** with loop support
- ✅ **Filtered node listing** by state and hardware manager
- ✅ **Cluster creation by 1 click** 



---

## 🆕 What's New in v2.0

#### 1. Automatic CIMC Authentication
- **Old:** Manually provide session cookies in inventory
- **New:** Playbook automatically authenticates via `/nuova` XML API and extracts cookies
- **Benefit:** Eliminates manual cookie management

#### 2. Bulk Node Onboarding
- **Old:** Onboard one node at a time with manual configuration
- **New:** Loop through all nodes automatically from hardware manager
- **Benefit:** Onboard 20+ nodes with a single playbook run

#### 3. Workflow Validation for Intersight Claims
- **Old:** No feedback on claim success/failure
- **New:** Polls `/api/v1/workflow/WorkflowInfos` until completion
- **Benefit:** Ensures device is fully registered before proceeding

#### 4. Filtered Node Listing
- **Old:** List all nodes regardless of state
- **New:** Filter by `STATE_ONBOARDED`, `INTERSIGHT` type, and hardware manager UUID
- **Benefit:** Only see nodes ready for deployment

#### 5. Parallel Multi-Server Support
- **New:** Added `strategy: free` support documentation
- **Benefit:** 10 servers complete in same time as 1 server

### API Improvements

| API Endpoint | Enhancement | Impact |
|-------------|-------------|---------|
| `/nuova` | Added XML-based login | Auto cookie extraction |
| `/connector/CommConfigs` | Changed header to `ucsmcookie` | Proper CIMC authentication |
| `/workflow/WorkflowInfos` | Added polling with filters | Claim validation |
| `/hardware_managers/<uuid>/nodes/list` | Added node discovery | Bulk onboarding |
| `/imaged_nodes/list` | Added state/type filters | Targeted deployment |
| `/imaged_nodes/onboard` | Added loop support | Multiple nodes at once |

### Data Export Features
- **Node data** → `fc-api-respones/hardware_managers_nodes_<ip>.txt` (JSON)
- **Hardware managers** → `fc-api-respones/hardware_managers_<ip>.txt`
- **Onboarded nodes** → `fc-api-respones/imaged_nodes_list_<ip>.txt` (JSON)

---

## ⚙️ Prerequisites

### Required Software

- **Ansible** 2.9 or higher
- **Python** 3.6 or higher
- **Cisco Intersight Collection** (for claim_server_intersight.yml)
  ```bash
  ansible-galaxy collection install cisco.intersight
  ```

### Network Requirements

- Network connectivity to CIMC management interfaces
- Network connectivity to Intersight PVA (for claiming)
- Network connectivity to Nutanix Foundation Central (for onboarding)
- HTTPS access to target servers (ports 443, 9440 for Foundation Central)

### Credentials Required

- CIMC admin credentials (username/password)
- Intersight API key and private key (for PVA claiming)
- Foundation Central credentials (username/password for node onboarding)
- Session cookie (optional, for DNS configuration)

---

## 📝 Inventory Configuration

The `inventory` file defines your servers and credentials. Update it with your environment details.

### Inventory File Structure

```ini
[Cisco_CSeries_servers]
server1 server_ip=10.193.129.9 user=admin default_password=password new_password=YourNewPassword NameServers=171.70.168.183 DomainName="" sessionCookie=your_session_cookie

[Cisco_CSeries_servers:vars]
# Intersight PVA Configuration
intersight_pva_ip=intersight-appliance-aa.cisco.com
api_key=YOUR_API_KEY_ID
api_private_key_path=/path/to/your/private_key.pem

# CIMC credentials mapping for claiming
server_cimc_ip={{ server_ip }}
server_cimc_username={{ user }}
server_cimc_password={{ new_password }}
```

### Inventory Variables

#### Per-Host Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `server_ip` | CIMC IP address | `10.193.129.9` |
| `user` | CIMC username | `admin` |
| `default_password` | Current/default CIMC password | `password` |
| `new_password` | Desired new password | `Cisco123!` |
| `NameServers` | DNS server IP(s), comma-separated | `171.70.168.183` |
| `DomainName` | DNS domain name | `example.com` |
| `sessionCookie` | Session cookie for API authentication | `bd0b7eb07099129d007990070e07d2c0` |

#### Group Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `intersight_pva_ip` | Intersight PVA hostname/IP | `intersight-appliance-aa.cisco.com` |
| `api_key` | Intersight API key ID | `65e8b8ca756461301fedd26e/...` |
| `api_private_key_path` | Path to Intersight private key file | `/path/to/private_key.pem` |

---

## 📚 Playbooks

### 1. CIMC Password Reset

**File:** `cimc_password_reset.yml`

#### Purpose
Resets the CIMC admin password using the Redfish API. The playbook checks if a password change is required and only performs the reset if necessary.

#### How It Works
1. Queries the Redfish API to check `PasswordChangeRequired` status
2. If password change is required, performs a PATCH request to update the password
3. Provides detailed debug output showing the operation results

#### Usage
```bash
ansible-playbook -i inventory cimc_password_reset.yml
```

#### Variables Used
- `server_ip` - Target CIMC IP address
- `user` - CIMC username
- `default_password` - Current password
- `new_password` - New password to set

#### Output
- Shows current password status
- Confirms successful password change
- Displays error messages if the operation fails

---

### 2. DNS Configuration via Device Connector

**File:** `add_dns_is_device_connector.yml`

#### Purpose
Configures DNS name servers and domain name on the CIMC using the Device Connector API with automatic session cookie retrieval.

#### How It Works
1. **Authenticates to CIMC** via `/nuova` endpoint with XML login request
2. **Extracts session cookie** (`outCookie`) from XML response
3. **Retrieves current CommConfigs** to verify authentication
4. **Sends PUT request** with DNS configuration (Name Servers and Domain Name)
5. Uses **ucsm-cookie** header format for authentication

#### New Features
- ✅ **Automatic cookie authentication** - No need to manually provide session cookie
- ✅ **XML-based login** via CIMC Nuova API
- ✅ **Regex-based cookie extraction** from login response
- ✅ **Multi-server support** with parallel execution capability

#### Usage
```bash
ansible-playbook -i inventory add_dns_is_device_connector.yml
```

#### Variables Used
- `server_ip` - Target CIMC IP address
- `user` - CIMC username
- `new_password` - CIMC password
- `NameServers` - DNS server IP(s)
- `DomainName` - DNS domain name

#### Authentication Flow
1. **POST to /nuova** - XML login: `<aaaLogin inPassword="..." inName="..." />`
2. **Extract outCookie** - Parse XML response for session token
3. **Use cookie** - Send as `ucsmcookie: ucsm-cookie=<token>` header

#### API Endpoints
- **Login:** `POST https://<server_ip>/nuova`
- **DNS Config:** `PUT https://<server_ip>/connector/CommConfigs`

#### Output
- Login response with session cookie
- Extracted cookie value
- Current DNS configuration
- Configuration status
- Success/failure messages

---

### 3. Claim Server to Intersight PVA

**File:** `claim_server_intersight.yml`

#### Purpose
Claims standalone Cisco UCS servers to Cisco Intersight Private Virtual Appliance (PVA) for centralized management and monitoring with automatic workflow validation.

#### How It Works
1. **Displays server information** before claiming
2. **Sends claim request** to Intersight PVA API endpoint
3. **Provides CIMC credentials** to establish connection
4. **Validates workflow completion** by polling WorkflowInfos API
5. **Reports final status** with workflow details

#### New Features
- ✅ **Workflow validation** - Automatically checks if device registration completed
- ✅ **Retry mechanism** - Polls workflow status with configurable retries (120 attempts, 15-second intervals)
- ✅ **Status filtering** - Only returns completed workflows
- ✅ **Multi-server parallel claiming** - Use `strategy: free` for independent execution

#### Usage
```bash
# Single or multiple servers
ansible-playbook -i inventory_cimc_intersight claim_server_intersight.yml
```

#### Variables Used
- `server_cimc_ip` - Target CIMC IP address
- `server_cimc_username` - CIMC username
- `server_cimc_password` - CIMC password
- `intersight_pva_ip` - Intersight PVA hostname/IP
- `api_key` - Intersight API key ID
- `api_private_key_path` - Path to Intersight private key

#### API Details

**Device Claim:**
- **Endpoint:** `/api/v1/appliance/DeviceClaims`
- **Method:** POST
- **Platform Type:** IMC (Integrated Management Controller)

**Workflow Validation:**
- **Endpoint:** `/api/v1/workflow/WorkflowInfos`
- **Method:** GET
- **Query Filters:**
  - `Name eq 'Device registration request'`
  - `WorkflowCtx/InitiatorCtx/InitiatorName eq '<server_cimc_ip>'`
  - `Status eq 'COMPLETED'`
- **Max Retries:** 120 (30 minutes total)
- **Retry Delay:** 15 seconds

#### Inventory File
Uses: `inventory_cimc_intersight`

```ini
[Cisco_CSeries_servers]
server1 server_ip=10.193.129.9 user=admin new_password=Nbv12345
server2 server_ip=10.193.129.65 user=admin new_password=Nbv12345

[Cisco_CSeries_servers:vars]
intersight_pva_ip=intersight-appliance-aa.cisco.com
api_key=YOUR_API_KEY_ID/PART2/PART3
api_private_key_path=/path/to/japikey-SecretKey.txt
```

#### Output
- Server information being claimed
- Claim operation status
- Workflow validation results
- Complete workflow response (status, name, message)
- Success or error messages

---

### 4. Onboard Servers to Foundation Central

**File:** `onboard_server_foundation_central.yml`

#### Purpose
Automates **bulk onboarding** of multiple Cisco UCS servers from Intersight to Nutanix Foundation Central. The playbook retrieves hardware manager information, lists all available nodes, and onboards them in parallel with a single execution.

#### How It Works
1. **Connects to Foundation Central API** using basic authentication
2. **Lists hardware managers** from `/api/fc/v1/hardware_managers/list`
3. **Extracts hardware manager UUIDs** and registers as Ansible facts
4. **Lists all nodes** from the hardware manager via `/nodes/list` endpoint
5. **Saves node data** to local JSON file for reference
6. **Bulk onboards nodes** using loop over all nodes in `hardware_managers_nodes_list.json.nodes`
7. **Waits 30 seconds** for onboarding to process
8. **Lists onboarded nodes** with filters (STATE_ONBOARDED, INTERSIGHT type)
9. **Provides summary** of successful and failed onboardings

#### New Features
- ✅ **Bulk node onboarding** - Loops through all nodes automatically
- ✅ **Automatic node discovery** - Retrieves all nodes from hardware manager
- ✅ **JSON file export** - Saves node data for reference
- ✅ **Progress tracking** - Shows onboarding summary (success/failed counts)
- ✅ **30-second wait** - Allows onboarding to complete before listing
- ✅ **Filtered listing** - Only shows onboarded Intersight nodes
- ✅ **Loop labels** - Displays node serial during execution

#### Usage
```bash
ansible-playbook -i inventory_foundation_central_onboard onboard_server_foundation_central.yml
```

#### Variables Used

**Foundation Central Connection:**
- `foundation_central_ip` - Foundation Central server IP address
- `fc_port` - Foundation Central API port (default: 9440)
- `fc_username` - Foundation Central username
- `fc_password` - Foundation Central password
- `fc_api_base_path` - Base API path (default: `/api/fc/v1`)

**Automatically Retrieved from Nodes:**
Each node contains complete configuration from Intersight:
- `node_serial` - Server serial number
- `block_serial` - Block serial number
- `memory_gb` - Memory in GB
- `cpu_core_count` - Number of CPU cores
- `cpu_vendor` - CPU manufacturer
- `model` - Server model (e.g., UCSC-C220-M7S)
- `hardware_manager_uuid` - Hardware manager UUID
- `hardware_manager_type` - Type (INTERSIGHT)
- `hardware_manager_data.intersight_data`:
  - `node_name` - Node name from Intersight
  - `organizations` - List of Intersight organizations
  - `domain` - Domain name (if applicable)
  - `classification` - Classification (NUTANIX, UNKNOWN)
  - `management_mode` - Management mode (ISM, IMM)
  - `tags` - List of tags

#### API Details

**1. Hardware Manager Listing:**
- **Endpoint:** `/api/fc/v1/hardware_managers/list`
- **Method:** POST
- **Body:** `{"length": 0, "offset": 0}`

**2. List Nodes from Hardware Manager:**
- **Endpoint:** `/api/fc/v1/hardware_managers/<uuid>/nodes/list`
- **Method:** POST
- **Body:** `{"length": 0, "offset": 0}`

**3. Bulk Node Onboarding (Loop):**
- **Endpoint:** `/api/fc/v1/imaged_nodes/onboard`
- **Method:** POST
- **Body:** Each complete node object from nodes list
- **Loop:** Iterates over `hardware_managers_nodes_list.json.nodes[]`

**4. Wait for Processing:**
- **Module:** `pause`
- **Duration:** 30 seconds

**5. List Onboarded Nodes (with filters):**
- **Endpoint:** `/api/fc/v1/imaged_nodes/list`
- **Method:** POST
- **Body:**
  ```json
  {
    "length": 0,
    "filters": {
      "node_state": "STATE_ONBOARDED",
      "hardware_manager_type": "INTERSIGHT",
      "hardware_manager_uuid": "<hardware_manager_uuid>"
    },
    "offset": 0
  }
  ```

#### Registered Variables
- `hardware_manager_uuids` - List of all hardware manager UUIDs
- `hardware_managers` - Complete hardware manager entities
- `hardware_managers_nodes_list` - All nodes from hardware manager
- `onboard_results` - Results from bulk onboarding loop

#### Output Files
- `./fc-api-respones/hardware_managers_<ip>.txt` - Hardware manager info
- `./fc-api-respones/hardware_managers_nodes_<ip>.txt` - Complete node list (JSON)
- `./fc-api-respones/imaged_nodes_list_<ip>.txt` - Onboarded nodes (JSON)
- **`./imaged_nodes_list`** - **Deployment-ready node details** (formatted for easy reference)

#### 📌 Important: Using Output for Deployment

The playbook generates `imaged_nodes_list` file with critical information needed for cluster deployment:

```yaml
Nodes UUIDs:
- UUID: 8e54f28b-c106-4512-627c-e807d93e958b
  block_serial: WZP27120LEL
  node_serial: WZP27120LEL
  model: UCSC-C220-M6S
  cpu_core_count: 52
  memory_gb: 256
- UUID: a2fc21e8-996f-4106-561e-e8d4b8c10135
  block_serial: WZP27300FAQ
  node_serial: WZP27300FAQ
  model: UCSC-C220-M7S
  cpu_core_count: 40
  memory_gb: 128
```

**Workflow for Deployment:**
1. ✅ Run `onboard_server_foundation_central.yml`
2. 📄 Open generated `./imaged_nodes_list` file
3. 📝 Copy `imaged_node_uuid` values from this file
4. ✏️ Update `inventory_node_deployment_fc`:
   - Set `node1_imaged_node_uuid=<UUID_from_file>`
   - Set `node2_imaged_node_uuid=<UUID_from_file>`
   - Update in the `imaged_cluster_body` JSON payload
5. ▶️ Run `node_deployment_foundation_central.yml`

**Example:**
```ini
# From imaged_nodes_list file:
# - UUID: 8e54f28b-c106-4512-627c-e807d93e958b (WZP27120LEL)
# - UUID: a2fc21e8-996f-4106-561e-e8d4b8c10135 (WZP27300FAQ)

# Update in inventory_node_deployment_fc:
node1_imaged_node_uuid=8e54f28b-c106-4512-627c-e807d93e958b
node2_imaged_node_uuid=a2fc21e8-996f-4106-561e-e8d4b8c10135
```

#### Output
- Total hardware managers found
- Hardware manager UUIDs and details
- Total nodes discovered
- Onboarding summary (success/failed counts)
- Node serial numbers being onboarded
- Final list of onboarded nodes with STATE_ONBOARDED status

#### Workflow Example

For a setup with 4 nodes in Intersight:
1. **Lists hardware managers** → Finds Intersight manager
2. **Lists nodes** → Discovers 4 nodes (WZP271200FU, WZP27300FAQ, WZP2848975L, FCH282271MZ)
3. **Bulk onboards** → Loops through all 4 nodes
4. **Waits** → 30 seconds for processing
5. **Validates** → Lists onboarded nodes with filters
6. **Reports** → "Onboarded 4 out of 4 nodes"

#### Inventory File
Uses: `inventory_foundation_central_onboard`

```ini
[foundation_central]
fc_server1 foundation_central_ip=10.193.210.172 fc_port=9440

[foundation_central:vars]
# Foundation Central Authentication
fc_username=admin
fc_password=Cisco@1357
fc_api_base_path=/api/fc/v1
```

**Note:** Node details are automatically retrieved from the hardware manager - no manual node configuration needed!

---

### 5. Deploy Nodes via Foundation Central

**File:** `node_deployment_foundation_central.yml`

#### Purpose
Deploys onboarded Cisco UCS servers as Nutanix clusters via Foundation Central. This playbook creates imaged clusters using nodes that have been previously onboarded.

#### How It Works
1. **Lists hardware managers** from Foundation Central
2. **Extracts hardware manager UUIDs**
3. **Lists onboarded nodes** with filters:
   - `node_state: STATE_ONBOARDED`
   - `hardware_manager_type: INTERSIGHT`
   - `hardware_manager_uuid: <uuid>`
4. **Creates imaged cluster** with specified configuration
5. **Reports deployment status**

#### New Features
- ✅ **Filtered node listing** - Only shows nodes ready for deployment
- ✅ **Hardware manager UUID filtering** - Targets specific Intersight instance
- ✅ **Complete cluster configuration** - Includes AOS, hypervisor, network settings
- ✅ **Multi-node cluster support** - Define multiple nodes in nodes_list

#### Usage

**Prerequisites:**
1. Nodes must be onboarded first using `onboard_server_foundation_central.yml`
2. Use the generated `imaged_nodes_list` file to get `imaged_node_uuid` values
3. Update `inventory_node_deployment_fc` with correct UUIDs

```bash
ansible-playbook -i inventory_node_deployment_fc node_deployment_foundation_central.yml
```

💡 **Tip:** The `imaged_node_uuid` values are automatically generated during onboarding. Check the `./imaged_nodes_list` file created by the onboarding playbook.

#### Variables Used

**Foundation Central Connection:**
- `foundation_central_ip` - Foundation Central IP
- `fc_port` - API port (9440)
- `fc_username` - Username
- `fc_password` - Password
- `fc_api_base_path` - Base path (`/api/fc/v1`)

**Cluster Configuration:**
- `aos_package_url` - URL to AOS installer package
- `hypervisor_iso_url` - URL to hypervisor ISO
- `cluster_name` - Name for the cluster
- `cvm_dns_servers` - DNS servers for CVMs
- `cvm_ntp_servers` - NTP servers for CVMs
- `hypervisor_dns_servers` - DNS servers for hypervisors
- `hypervisor_ntp_servers` - NTP servers for hypervisors
- `redundancy_factor` - Replication factor (2 or 3)
- `timezone` - Cluster timezone
- `api_key_uuid` - Foundation Central API key UUID

**Node Configuration (per node):**
- `cvm_gateway` - CVM gateway IP
- `cvm_ip` - CVM IP address
- `cvm_netmask` - CVM subnet mask
- `hypervisor_gateway` - Hypervisor gateway
- `hypervisor_hostname` - Hypervisor hostname
- `hypervisor_ip` - Hypervisor IP
- `hypervisor_netmask` - Hypervisor subnet mask
- `imaged_node_uuid` - UUID from onboarding

#### API Details

**List Onboarded Nodes:**
- **Endpoint:** `/api/fc/v1/imaged_nodes/list`
- **Method:** POST
- **Body with Filters:**
  ```json
  {
    "length": 0,
    "filters": {
      "node_state": "STATE_ONBOARDED",
      "hardware_manager_type": "INTERSIGHT",
      "hardware_manager_uuid": "<uuid>"
    },
    "offset": 0
  }
  ```

**Create Imaged Cluster:**
- **Endpoint:** `/api/fc/v1/imaged_clusters`
- **Method:** POST
- **Body:** Complete cluster configuration JSON (see inventory example below)

#### Inventory File
Uses: `inventory_node_deployment_fc`

```ini
[foundation_central_deployment]
fc_server1 foundation_central_ip=10.193.210.172 fc_port=9440

[foundation_central_deployment:vars]
fc_username=admin
fc_password=Cisco@1357
fc_api_base_path=/api/fc/v1

# Cluster Configuration
aos_package_url=http://10.193.229.229:8080/images/AOS/7.3.0.5/nutanix_installer_package-release-ganges-7.3.0.5-stable.tar.gz
hypervisor_iso_url=http://10.193.229.229:8080/images/AOS/7.3.0.5/AHV-DVD-x86_64-10.3.0.1-24.iso
cluster_name=1clicktest
cvm_dns_servers=171.70.168.183
cvm_ntp_servers=ntp.xyz.abc.com
redundancy_factor=2
timezone=UTC
api_key_uuid=ce7fed71-7f6e-4162-5c5c-e1485f7bfaed

# Node 1
node1_cvm_ip=10.1.1.21
node1_hypervisor_ip=10.1.1.11
node1_imaged_node_uuid=8e54f28b-c106-4512-627c-e807d93e958b

# Node 2
node2_cvm_ip=10.1.1.22
node2_hypervisor_ip=10.1.1.12
node2_imaged_node_uuid=a2fc21e8-996f-4106-561e-e8d4b8c10135

# Full JSON payload
imaged_cluster_body='<complete_json_configuration>'
```

#### Output
- Hardware manager information
- List of onboarded nodes available for deployment
- Cluster creation status
- Deployment task ID and status



### Complete Workflow: End-to-End Automation

**Phase 1: CIMC Configuration (Cisco UCS Servers)**
```bash
# Step 1: Reset CIMC password
ansible-playbook -i inventory cimc_password_reset.yml

# Step 2: Configure DNS settings (with automatic cookie auth)
ansible-playbook -i inventory add_dns_is_device_connector.yml
```

**Phase 2: Claim to Intersight PVA**
```bash
# Step 3: Claim servers to Intersight PVA (with workflow validation)
ansible-playbook -i inventory_cimc_intersight claim_server_intersight.yml
```

**Phase 3: Foundation Central Operations**
```bash
# Step 4: Bulk onboard all nodes from Intersight to Foundation Central
ansible-playbook -i inventory_foundation_central_onboard onboard_server_foundation_central.yml

# Step 4.5: Update inventory with imaged_node_uuid values
# Open the generated ./imaged_nodes_list file
# Copy UUID values and update inventory_node_deployment_fc

# Step 5: Deploy onboarded nodes as Nutanix cluster
ansible-playbook -i inventory_node_deployment_fc node_deployment_foundation_central.yml
```

**📌 Important Bridge Step:**
After onboarding (Step 4), the playbook creates `./imaged_nodes_list` containing `imaged_node_uuid` for each node. These UUIDs **must** be copied into `inventory_node_deployment_fc` before running the deployment (Step 5).

### Parallel Execution for Multiple Servers

All playbooks support multi-server operations. For maximum parallelism, add `strategy: free` to the playbook:

```yaml
---
- name: Your Playbook Name
  hosts: your_host_group
  connection: local
  gather_facts: false
  strategy: free  # Enable parallel execution
  
  tasks:
    # ... your tasks
```

**Benefits of `strategy: free`:**
- Each server runs through all tasks independently
- Faster execution for multi-server operations
- No waiting at task boundaries

### Run for Specific Hosts

```bash
# Target specific server
ansible-playbook -i inventory -l server1 cimc_password_reset.yml

# Target multiple servers
ansible-playbook -i inventory -l server1,server2 claim_server_intersight.yml
```

### Verbose Output for Debugging

```bash
# Use -v for verbose, -vv for more verbose, -vvv for maximum verbosity
ansible-playbook -i inventory -vvv add_dns_is_device_connector.yml
```

```

---



## 📊 Playbook Execution Order

### Complete End-to-End Workflow:

```
Phase 1: CIMC Setup
├── 1. cimc_password_reset.yml              → Secure the CIMC with new password
└── 2. add_dns_is_device_connector.yml      → Configure DNS (auto cookie auth)

Phase 2: Intersight Integration
└── 3. claim_server_intersight.yml          → Claim to Intersight PVA (with validation)

Phase 3: Foundation Central
├── 4. onboard_server_foundation_central.yml → Bulk onboard nodes (loop through all)
│    └── 📄 Generates: ./imaged_nodes_list
│    └── 📝 Action Required: Copy imaged_node_uuid to inventory_node_deployment_fc
└── 5. node_deployment_foundation_central.yml → Deploy as Nutanix cluster
```

**Critical Data Flow:**
```
onboard_server_foundation_central.yml
    ↓
./imaged_nodes_list (contains imaged_node_uuid values)
    ↓
[Manual Step] Copy UUIDs to inventory_node_deployment_fc
    ↓
node_deployment_foundation_central.yml (uses UUIDs for deployment)
```

### Execution Time Estimates

| Playbook | Single Server | Multiple Servers (parallel) |
|----------|--------------|----------------------------|
| cimc_password_reset.yml | ~10 seconds | ~10 seconds |
| add_dns_is_device_connector.yml | ~15 seconds | ~15 seconds |
| claim_server_intersight.yml | ~5-30 minutes* | ~5-30 minutes* |
| onboard_server_foundation_central.yml | ~2 minutes | ~2 minutes |
| node_deployment_foundation_central.yml | ~20-60 minutes** | ~20-60 minutes** |

*Includes workflow validation polling (up to 30 minutes max)  
**Depends on AOS installation and cluster creation time

---


---

## 📁 Inventory Files Reference

The repository uses multiple inventory files for different workflows:

| Inventory File | Used By | Purpose |
|---------------|---------|---------|
| `inventory` | cimc_password_reset.yml<br>add_dns_is_device_connector.yml | CIMC management operations |
| `inventory_cimc_intersight` | claim_server_intersight.yml | Claiming servers to Intersight PVA |
| `inventory_foundation_central_onboard` | onboard_server_foundation_central.yml | Onboarding nodes to Foundation Central |
| `inventory_node_deployment_fc` | node_deployment_foundation_central.yml | Deploying Nutanix clusters |

---

## 🔍 Troubleshooting

### Common Issues

**1. Session Cookie Authentication Fails**
- **Solution:** The playbook now automatically retrieves session cookies via `/nuova` endpoint
- Ensure CIMC credentials are correct in inventory

**2. Workflow Validation Times Out (claim_server_intersight.yml)**
- **Cause:** Device registration taking longer than 30 minutes
- **Solution:** Increase `retries` parameter in the workflow validation task
- Check Intersight PVA connectivity and device status

**3. Firmware Bundle Missing (node_deployment_foundation_central.yml)**
- **Error:** `firmware bundle(s) are missing in Intersight`
- **Solution:** 
  - Upload required firmware bundle to Intersight
  - OR set `"reuse_existing_software": true` in `imaged_cluster_body`

**4. Node Onboarding Returns Empty List**
- **Cause:** Nodes not yet discovered by hardware manager
- **Solution:** 
  - Wait for Intersight sync to complete
  - Verify hardware manager UUID is correct
  - Check node classification in Intersight

**5. Multi-Server Operations Execute Slowly**
- **Solution:** Add `strategy: free` to playbook for parallel execution
- Increase `forks` in ansible.cfg for more concurrent connections

**6. Deployment Fails with "Invalid imaged_node_uuid"**
- **Error:** `Node with UUID <uuid> not found or not in STATE_ONBOARDED`
- **Cause:** Using wrong or outdated UUIDs in deployment inventory
- **Solution:**
  1. Run `onboard_server_foundation_central.yml` first
  2. Open the generated `./imaged_nodes_list` file
  3. Copy the correct `imaged_node_uuid` values
  4. Update `inventory_node_deployment_fc` with these UUIDs
  5. Ensure UUIDs match in both individual node variables AND the `imaged_cluster_body` JSON

**7. Cannot Find Generated Files**
- **Files created:**
  - `./imaged_nodes_list` - Human-readable node details
  - `./fc-api-respones/imaged_nodes_list_<ip>.txt` - Complete JSON response
  - `./fc-api-respones/hardware_managers_nodes_<ip>.txt` - All node data
- **Location:** Files are created in the playbook execution directory
- **Solution:** Check current working directory when running playbook

---

## 🔄 Version History

### v2.0 (January 2026)
- ✅ Added automatic cookie authentication for CIMC APIs
- ✅ Implemented workflow validation for Intersight device claims
- ✅ Added bulk node onboarding with loop support
- ✅ Implemented filtered node listing by state and type
- ✅ Added node deployment playbook for cluster creation
- ✅ Added 30-second wait for onboarding processing
- ✅ Implemented parallel execution support for multi-server operations
- ✅ Added JSON file exports for node and hardware manager data

### v1.0 (December 2025)
- Initial release with basic CIMC management and Intersight claiming
- Foundation Central hardware manager listing 

---

**Last Updated:** January 23, 2026

