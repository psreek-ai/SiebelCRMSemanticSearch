# Deployment Guide: AI-Powered Semantic Search Solution

**Version:** 3.0 (Oracle 23ai + Azure AI Foundry Implementation)
**Date:** 2025-10-17
**Audience:** Deployment Team, Database Administrators, System Administrators, Siebel Administrators

## 1. Introduction
This guide provides a comprehensive, step-by-step process for deploying the AI-Powered Semantic Search solution into a target environment (DEV, UAT, or Production). This deployment uses **Oracle Database 23ai on Azure VM** (self-managed installation), **Azure AI Foundry** with OpenAI Service for embeddings, and Siebel CRM. The self-managed deployment provides full control over database configuration, ORDS installation, and resource allocation. **Follow these steps in the exact sequence specified.**

## 2. Pre-Deployment Checklist

### 2.1. Infrastructure Requirements

**Oracle Database 23ai on Azure VM:**
- [ ] Azure subscription with appropriate permissions to provision VMs and networking resources
- [ ] Azure VM provisioned (Standard_D8s_v3 or larger recommended - 8 vCPUs, 32 GB RAM)
- [ ] Premium SSD storage (minimum 512 GB for database files, additional for backups)
- [ ] Oracle Database 23ai Enterprise Edition installation media downloaded
- [ ] ORDS 23.x installation files downloaded from Oracle
- [ ] Network connectivity configured within Azure VNet
- [ ] Database links configured to Oracle 19c Siebel database
- [ ] Manual ORDS installation and configuration completed

**Azure AI Foundry:**
- [ ] Azure AI Foundry workspace created in target region
- [ ] OpenAI Service deployment created (text-embedding-3-small or text-embedding-3-large)
- [ ] Private endpoint configured for secure connectivity from Oracle 23ai VM
- [ ] API keys generated and securely stored in Azure Key Vault
- [ ] Content filtering and responsible AI policies configured

**Oracle Database 19c (Siebel):**
- [ ] Read-only user account created with access to Siebel schema
- [ ] Listener configured and accessible from Oracle 23ai VM (via VNet connectivity)
- [ ] Network firewall rules allow connections from Oracle 23ai VM subnet

**Siebel CRM:**
- [ ] Siebel Tools access with repository check-out privileges
- [ ] Siebel Web Server access for file deployment
- [ ] Siebel Application Server access for SRF deployment
- [ ] Administrative access to Siebel client

### 2.2. Configuration Values Document

Create a deployment configuration document with these values:

```
# Oracle Database 23ai on Azure VM
ORACLE23AI_VM_NAME=<vm_name>
ORACLE23AI_VM_IP=<private_ip_address>
ORACLE23AI_ADMIN_PASSWORD=<secure_sys_password>
ORACLE23AI_SERVICE_NAME=<oracle_sid>
ORACLE23AI_ORDS_URL=http://<vm_ip>:8080/ords/
ORACLE23AI_LISTENER_PORT=1521
ORACLE23AI_SSH_USER=<admin_username>

# ORDS Configuration
ORDS_VERSION=23.3.0
ORDS_CONFIG_DIR=/opt/oracle/ords/config
ORDS_JETTY_PORT=8080
ORDS_PUBLIC_URL=http://<vm_ip>:8080/ords/

# Azure Networking
AZURE_VNET_NAME=<vnet_name>
AZURE_SUBNET_DATABASE=<database_subnet_name>
AZURE_SUBNET_SIEBEL=<siebel_subnet_name>
AZURE_NSG_DATABASE=<database_nsg_name>

# Azure AI Foundry
AI_FOUNDRY_WORKSPACE=<workspace_name>
AI_FOUNDRY_ENDPOINT=https://<workspace-name>.<region>.api.azureml.ms
AI_FOUNDRY_API_KEY=<stored_in_azure_key_vault>
AI_FOUNDRY_DEPLOYMENT_NAME=<openai_deployment_name>
AI_FOUNDRY_MODEL=text-embedding-3-small
AI_FOUNDRY_DIMENSIONS=1536

# Oracle 19c Siebel Database
DB_19C_HOST=<hostname>
DB_19C_PORT=1521
DB_19C_SERVICE=<service_name>
DB_19C_READONLY_USER=siebel_readonly
DB_19C_READONLY_PWD=<password>

# Siebel Configuration
SIEBEL_WEB_ROOT=<path_to_siebsrvr/public/enu/files>
SIEBEL_SRF_PATH=<path_to_siebsrvr/objects>
SIEBEL_SERVER_NAME=<siebel_server_name>

# Security
API_KEY=<generate_secure_api_key>
```

---

## 3. Deployment Phases

## Phase 1: Provision Azure VM for Oracle Database 23ai

**Duration:** 2-3 hours  
**Executor:** Cloud Administrator / Infrastructure Team  
**Rollback Time:** < 30 minutes (delete VM)

### Step 1.1: Create Azure Virtual Machine

**Navigate to Azure Portal and create VM:**

1. In Azure Portal, navigate to **Virtual Machines** → **+ Create**
2. Configure the VM with the following settings:

**Basic Configuration:**
```
Subscription: <your_azure_subscription>
Resource Group: <create_new_or_select_existing>
VM Name: oracle23ai-vector-db
Region: <select_azure_region>
Availability Options: Availability Zone (for production) or No infrastructure redundancy (for dev/test)
Image: Oracle Linux 8.x or Red Hat Enterprise Linux 8.x
Size: Standard_D8s_v3 (8 vCPUs, 32 GB RAM) - minimum for Oracle 23ai
```

**Disks:**
```
OS Disk Type: Premium SSD (P10 - 128 GB)
Data Disks: 
  - Disk 1 (Database): Premium SSD P30 (1024 GB) - /u01
  - Disk 2 (Backup): Premium SSD P20 (512 GB) - /u02
Encryption: Platform-managed keys or customer-managed keys
```

**Networking:**
```
Virtual Network: <select_or_create_vnet>
Subnet: database-subnet (10.0.3.0/24)
Public IP: None (use Bastion or VPN for management access)
NIC Network Security Group: Advanced
Configure Network Security Group:
  - Allow SSH (22) from management subnet only
  - Allow Oracle Listener (1521) from Siebel subnet
  - Allow ORDS (8080) from application subnets
  - Allow HTTPS (443) for outbound to Azure AI Foundry
Accelerated Networking: Enabled
```

**Management:**
```
Boot Diagnostics: Enable with managed storage account
Auto-shutdown: Configure schedule (optional for dev/test)
Backup: Enable Azure Backup (recommended for production)
```

3. Click **Review + Create**, then **Create**
4. Wait for deployment to complete (typically 5-10 minutes)
5. Note the **Private IP Address** for later configuration

### Step 1.1.1: Configure Network Security Groups (NSGs)

**Duration:** 30 minutes  
**Executor:** Cloud Administrator / Network Administrator

Network Security Groups control inbound and outbound traffic to Azure resources. Proper NSG configuration is critical for security and connectivity.

#### NSG Architecture

```
Azure VNet (10.0.0.0/16)
├── Management Subnet (10.0.1.0/24) - Azure Bastion, Jump Box
├── Siebel Subnet (10.0.2.0/24) - Siebel CRM VMs
├── Database Subnet (10.0.3.0/24) - Oracle 23ai VM
├── Application Subnet (10.0.4.0/24) - Test apps, Container Apps
└── AI Foundry Subnet (10.0.5.0/24) - Private Endpoints
```

#### Create Network Security Groups

```bash
# Set variables
RESOURCE_GROUP="<your-resource-group>"
LOCATION="<azure-region>"
VNET_NAME="semantic-search-vnet"

# Create NSG for Database Subnet (Oracle 23ai VM)
az network nsg create \
  --resource-group $RESOURCE_GROUP \
  --name database-nsg \
  --location $LOCATION

# Create NSG for Siebel Subnet
az network nsg create \
  --resource-group $RESOURCE_GROUP \
  --name siebel-nsg \
  --location $LOCATION

# Create NSG for Application Subnet
az network nsg create \
  --resource-group $RESOURCE_GROUP \
  --name application-nsg \
  --location $LOCATION
```

#### Configure Database NSG (Oracle 23ai VM)

**Inbound Security Rules:**

| Priority | Name | Source | Source Port | Destination | Dest Port | Protocol | Action | Description |
|----------|------|--------|-------------|-------------|-----------|----------|--------|-------------|
| 100 | Allow-SSH-Bastion | Management-subnet (10.0.1.0/24) | * | VirtualNetwork | 22 | TCP | Allow | SSH access from Bastion |
| 110 | Allow-Oracle-Listener-Siebel | Siebel-subnet (10.0.2.0/24) | * | VirtualNetwork | 1521 | TCP | Allow | Oracle database connections from Siebel |
| 120 | Allow-ORDS-HTTP-Siebel | Siebel-subnet (10.0.2.0/24) | * | VirtualNetwork | 8080 | TCP | Allow | ORDS HTTP from Siebel |
| 130 | Allow-ORDS-HTTP-Apps | Application-subnet (10.0.4.0/24) | * | VirtualNetwork | 8080 | TCP | Allow | ORDS HTTP from test apps |
| 140 | Allow-ORDS-HTTPS-Siebel | Siebel-subnet (10.0.2.0/24) | * | VirtualNetwork | 443 | TCP | Allow | ORDS HTTPS from Siebel (production) |
| 150 | Allow-ORDS-HTTPS-Apps | Application-subnet (10.0.4.0/24) | * | VirtualNetwork | 443 | TCP | Allow | ORDS HTTPS from test apps (production) |
| 65000 | Allow-VNet-Inbound | VirtualNetwork | * | VirtualNetwork | * | Any | Allow | Default Azure rule |
| 65001 | Allow-AzureLoadBalancer | AzureLoadBalancer | * | * | * | Any | Allow | Default Azure rule |
| 65500 | Deny-All-Inbound | * | * | * | * | Any | Deny | Explicit deny all |

**Outbound Security Rules:**

| Priority | Name | Source | Source Port | Destination | Dest Port | Protocol | Action | Description |
|----------|------|--------|-------------|-------------|-----------|----------|--------|-------------|
| 100 | Allow-Oracle-To-Siebel | VirtualNetwork | * | Siebel-subnet (10.0.2.0/24) | 1521 | TCP | Allow | Database link to Siebel Oracle 19c |
| 110 | Allow-Azure-AI-Foundry | VirtualNetwork | * | AzureCloud.EastUS | 443 | TCP | Allow | Azure AI Foundry API (use your region) |
| 120 | Allow-Internet-HTTPS | VirtualNetwork | * | Internet | 443 | TCP | Allow | Software updates, Azure services |
| 130 | Allow-Internet-HTTP | VirtualNetwork | * | Internet | 80 | TCP | Allow | Package downloads (yum, wget) |
| 140 | Allow-DNS | VirtualNetwork | * | VirtualNetwork | 53 | TCP,UDP | Allow | DNS resolution |
| 150 | Allow-NTP | VirtualNetwork | * | Internet | 123 | UDP | Allow | Time synchronization |
| 65000 | Allow-VNet-Outbound | VirtualNetwork | * | VirtualNetwork | * | Any | Allow | Default Azure rule |
| 65001 | Allow-Internet-Outbound | * | * | Internet | * | Any | Allow | Default Azure rule |
| 65500 | Deny-All-Outbound | * | * | * | * | Any | Deny | Explicit deny all (optional) |

**Create NSG Rules via Azure CLI:**

```bash
# Database NSG - Inbound Rules
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name database-nsg \
  --name Allow-SSH-Bastion \
  --priority 100 \
  --source-address-prefixes 10.0.1.0/24 \
  --destination-port-ranges 22 \
  --protocol Tcp \
  --access Allow \
  --direction Inbound \
  --description "SSH access from Azure Bastion"

az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name database-nsg \
  --name Allow-Oracle-Listener-Siebel \
  --priority 110 \
  --source-address-prefixes 10.0.2.0/24 \
  --destination-port-ranges 1521 \
  --protocol Tcp \
  --access Allow \
  --direction Inbound \
  --description "Oracle Listener from Siebel subnet"

az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name database-nsg \
  --name Allow-ORDS-HTTP-Siebel \
  --priority 120 \
  --source-address-prefixes 10.0.2.0/24 \
  --destination-port-ranges 8080 \
  --protocol Tcp \
  --access Allow \
  --direction Inbound \
  --description "ORDS HTTP from Siebel subnet"

az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name database-nsg \
  --name Allow-ORDS-HTTP-Apps \
  --priority 130 \
  --source-address-prefixes 10.0.4.0/24 \
  --destination-port-ranges 8080 \
  --protocol Tcp \
  --access Allow \
  --direction Inbound \
  --description "ORDS HTTP from application subnet"

az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name database-nsg \
  --name Allow-ORDS-HTTPS-Siebel \
  --priority 140 \
  --source-address-prefixes 10.0.2.0/24 \
  --destination-port-ranges 443 \
  --protocol Tcp \
  --access Allow \
  --direction Inbound \
  --description "ORDS HTTPS from Siebel (production)"

az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name database-nsg \
  --name Allow-ORDS-HTTPS-Apps \
  --priority 150 \
  --source-address-prefixes 10.0.4.0/24 \
  --destination-port-ranges 443 \
  --protocol Tcp \
  --access Allow \
  --direction Inbound \
  --description "ORDS HTTPS from applications (production)"

# Database NSG - Outbound Rules
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name database-nsg \
  --name Allow-Oracle-To-Siebel \
  --priority 100 \
  --source-address-prefixes VirtualNetwork \
  --destination-address-prefixes 10.0.2.0/24 \
  --destination-port-ranges 1521 \
  --protocol Tcp \
  --access Allow \
  --direction Outbound \
  --description "Database link to Siebel Oracle 19c"

az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name database-nsg \
  --name Allow-Azure-AI-Foundry \
  --priority 110 \
  --source-address-prefixes VirtualNetwork \
  --destination-address-prefixes AzureCloud.EastUS \
  --destination-port-ranges 443 \
  --protocol Tcp \
  --access Allow \
  --direction Outbound \
  --description "Azure AI Foundry API access"

az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name database-nsg \
  --name Allow-Internet-HTTPS \
  --priority 120 \
  --source-address-prefixes VirtualNetwork \
  --destination-address-prefixes Internet \
  --destination-port-ranges 443 \
  --protocol Tcp \
  --access Allow \
  --direction Outbound \
  --description "HTTPS for Azure services and updates"

az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name database-nsg \
  --name Allow-Internet-HTTP \
  --priority 130 \
  --source-address-prefixes VirtualNetwork \
  --destination-address-prefixes Internet \
  --destination-port-ranges 80 \
  --protocol Tcp \
  --access Allow \
  --direction Outbound \
  --description "HTTP for package downloads"
```

#### Associate NSG with Subnet

```bash
# Associate database NSG with database subnet
az network vnet subnet update \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name database-subnet \
  --network-security-group database-nsg

# Verify NSG association
az network vnet subnet show \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name database-subnet \
  --query "networkSecurityGroup.id"
```

#### Configure Siebel NSG (Optional - for reference)

**Inbound Rules:**

| Priority | Name | Source | Dest Port | Description |
|----------|------|--------|-----------|-------------|
| 100 | Allow-HTTPS-Users | Internet or Corporate Network | 443 | HTTPS for Siebel web users |
| 110 | Allow-SSH-Mgmt | Management-subnet | 22 | SSH for management |
| 120 | Allow-Siebel-Ports | VirtualNetwork | 2321, 8080 | Siebel application ports |

**Outbound Rules:**

| Priority | Name | Destination | Dest Port | Description |
|----------|------|-------------|-----------|-------------|
| 100 | Allow-Oracle-23ai | Database-subnet | 1521, 8080, 443 | Oracle 23ai database and ORDS |
| 110 | Allow-Internet | Internet | 443, 80 | Internet access |

#### NSG Security Best Practices

**1. Principle of Least Privilege:**
- Only allow ports that are absolutely necessary
- Use specific source IP ranges instead of * (any)
- Remove default allow rules if possible

**2. Service Tags vs IP Ranges:**
```bash
# Prefer Service Tags for Azure services
--destination-address-prefixes AzureCloud.EastUS  # Good
--destination-address-prefixes Internet            # Use with caution

# For specific services
--destination-address-prefixes Storage.EastUS      # Azure Storage
--destination-address-prefixes Sql.EastUS          # Azure SQL
```

**3. NSG Flow Logs (Recommended for Production):**
```bash
# Create storage account for NSG flow logs
az storage account create \
  --name nsgflowlogs$RANDOM \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS

# Enable NSG flow logs
az network watcher flow-log create \
  --resource-group $RESOURCE_GROUP \
  --nsg database-nsg \
  --name database-nsg-flow-log \
  --location $LOCATION \
  --storage-account nsgflowlogs$RANDOM \
  --enabled true \
  --retention 30 \
  --format JSON \
  --log-version 2
```

**4. NSG Diagnostic Settings:**
```bash
# Create Log Analytics workspace
az monitor log-analytics workspace create \
  --resource-group $RESOURCE_GROUP \
  --workspace-name semantic-search-logs \
  --location $LOCATION

# Get workspace ID
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group $RESOURCE_GROUP \
  --workspace-name semantic-search-logs \
  --query id -o tsv)

# Enable diagnostic logs for NSG
az monitor diagnostic-settings create \
  --name nsg-diagnostics \
  --resource $(az network nsg show --resource-group $RESOURCE_GROUP --name database-nsg --query id -o tsv) \
  --workspace $WORKSPACE_ID \
  --logs '[{"category": "NetworkSecurityGroupEvent", "enabled": true}, {"category": "NetworkSecurityGroupRuleCounter", "enabled": true}]'
```

#### Verify NSG Configuration

```bash
# List all NSG rules
az network nsg show \
  --resource-group $RESOURCE_GROUP \
  --name database-nsg \
  --query "securityRules[].{Name:name, Priority:priority, Direction:direction, Access:access, Protocol:protocol, SourcePort:sourcePortRange, DestPort:destinationPortRange}" \
  --output table

# Test connectivity (from Siebel VM to Oracle 23ai VM)
# SSH to Siebel VM, then:
telnet <oracle-23ai-vm-private-ip> 8080
# Expected: Connected

# Test from Oracle 23ai VM to Siebel Oracle 19c
# SSH to Oracle 23ai VM, then:
telnet <siebel-vm-private-ip> 1521
# Expected: Connected
```

#### NSG Troubleshooting

**Connection Issues:**

```bash
# Check effective NSG rules on network interface
az network nic list-effective-nsg \
  --resource-group $RESOURCE_GROUP \
  --name <vm-nic-name>

# Verify IP flow (from Siebel to Oracle 23ai)
az network watcher test-ip-flow \
  --resource-group $RESOURCE_GROUP \
  --vm <siebel-vm-name> \
  --direction Outbound \
  --protocol TCP \
  --local <siebel-vm-ip>:* \
  --remote <oracle-23ai-vm-ip>:8080
# Expected: Access: Allow

# Check connection monitor
az network watcher connection-monitor create \
  --resource-group $RESOURCE_GROUP \
  --name siebel-to-oracle23ai \
  --source-resource <siebel-vm-id> \
  --dest-resource <oracle-23ai-vm-id> \
  --dest-port 8080 \
  --protocol Tcp
```

---

### Step 1.2: Configure OS and Storage for Oracle Database

**Connect to VM via SSH or Azure Bastion:**
```bash
ssh <admin_user>@<vm_private_ip>
```

**Configure Storage:**
```bash
# Become root
sudo su -

# List attached disks
lsblk

# Format and mount data disk for database (/dev/sdc)
mkfs.xfs /dev/sdc
mkdir -p /u01
echo "/dev/sdc /u01 xfs defaults,nofail 0 2" >> /etc/fstab
mount -a

# Format and mount backup disk (/dev/sdd)
mkfs.xfs /dev/sdd
mkdir -p /u02
echo "/dev/sdd /u02 xfs defaults,nofail 0 2" >> /etc/fstab
mount -a

# Create Oracle directory structure
mkdir -p /u01/app/oracle/product/23.0.0/dbhome_1
mkdir -p /u01/app/oracle/oradata
mkdir -p /u01/app/oracle/fast_recovery_area
mkdir -p /u02/backup

# Set permissions
chown -R oracle:oinstall /u01 /u02
chmod -R 775 /u01 /u02
```

**Install Required Packages:**
```bash
# Update system
yum update -y

# Install Oracle preinstall package (handles dependencies and kernel parameters)
yum install -y oracle-database-preinstall-23ai

# Install additional required packages
yum install -y \
  bc \
  binutils \
  compat-libcap1 \
  compat-libstdc++-33 \
  elfutils-libelf \
  elfutils-libelf-devel \
  fontconfig-devel \
  glibc \
  glibc-devel \
  ksh \
  libaio \
  libaio-devel \
  libX11 \
  libXau \
  libXi \
  libXtst \
  libXrender \
  libXrender-devel \
  libgcc \
  libstdc++ \
  libstdc++-devel \
  libxcb \
  make \
  smartmontools \
  sysstat \
  unixODBC \
  unixODBC-devel \
  wget \
  java-11-openjdk

# Verify oracle user was created
id oracle
```

**Configure Kernel Parameters (if not set by preinstall):**
```bash
# Edit /etc/sysctl.conf
cat >> /etc/sysctl.conf << EOF
fs.file-max = 6815744
kernel.sem = 250 32000 100 128
kernel.shmmni = 4096
kernel.shmall = 1073741824
kernel.shmmax = 4398046511104
kernel.panic_on_oops = 1
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2
fs.aio-max-nr = 1048576
net.ipv4.ip_local_port_range = 9000 65500
EOF

# Apply settings
sysctl -p
```

**Configure Limits:**
```bash
cat >> /etc/security/limits.conf << EOF
oracle   soft   nofile    1024
oracle   hard   nofile    65536
oracle   soft   nproc    16384
oracle   hard   nproc    16384
oracle   soft   stack    10240
oracle   hard   stack    32768
oracle   hard   memlock    134217728
oracle   soft   memlock    134217728
EOF
```

**Set Environment Variables for oracle user:**
```bash
# Switch to oracle user
su - oracle

# Edit .bash_profile
cat >> ~/.bash_profile << 'EOF'
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/23.0.0/dbhome_1
export ORACLE_SID=VECSRCH
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
EOF

# Apply environment
source ~/.bash_profile
```

**Verification:**
```bash
# Check disk space
df -h
# Expect: /u01 with ~1TB, /u02 with ~500GB

# Check oracle user environment
echo $ORACLE_HOME
echo $ORACLE_SID

# Check system resources
free -h
nproc
```

---

## Phase 2: Install Oracle Database 23ai

**Duration:** 2-4 hours  
**Executor:** Database Administrator  
**Rollback Time:** 1 hour (delete software)

### Step 2.1: Upload Oracle 23ai Installation Media

**Download Oracle Database 23ai from Oracle Technology Network:**
- Visit: https://www.oracle.com/database/technologies/oracle-database-software-downloads.html
- Download: Oracle Database 23ai Enterprise Edition for Linux x86-64
- File: LINUX.X64_233000_db_home.zip (~3.5 GB)

**Upload to Azure VM:**
```bash
# From local machine, upload to VM
scp LINUX.X64_233000_db_home.zip <admin_user>@<vm_ip>:/tmp/

# On the VM, move to oracle user's staging area
sudo su - oracle
mkdir -p /u01/stage
mv /tmp/LINUX.X64_233000_db_home.zip /u01/stage/
```

### Step 2.2: Extract and Install Oracle Database Software

**Extract installation files:**
```bash
# As oracle user
cd $ORACLE_HOME
unzip -q /u01/stage/LINUX.X64_233000_db_home.zip
```

**Run Oracle Installer (Silent Mode):**
```bash
# Create response file for silent installation
cat > /tmp/db_install.rsp << EOF
oracle.install.option=INSTALL_DB_SWONLY
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/u01/app/oraInventory
ORACLE_HOME=/u01/app/oracle/product/23.0.0/dbhome_1
ORACLE_BASE=/u01/app/oracle
oracle.install.db.InstallEdition=EE
oracle.install.db.OSDBA_GROUP=dba
oracle.install.db.OSBACKUPDBA_GROUP=dba
oracle.install.db.OSDGDBA_GROUP=dba
oracle.install.db.OSKMDBA_GROUP=dba
oracle.install.db.OSRACDBA_GROUP=dba
SECURITY_UPDATES_VIA_MYORACLE=false
DECLINE_SECURITY_UPDATES=true
EOF

# Run installer
cd $ORACLE_HOME
./runInstaller -silent -responseFile /tmp/db_install.rsp -waitforcompletion

# Run root scripts as root
sudo /u01/app/oraInventory/orainstRoot.sh
sudo /u01/app/oracle/product/23.0.0/dbhome_1/root.sh
```

**Verification:**
```bash
# Check Oracle Home
ls -la $ORACLE_HOME/bin/sqlplus

# Check version
$ORACLE_HOME/bin/sqlplus -version
# Expected: SQL*Plus: Release 23.0.0.0.0
```

### Step 2.3: Create Oracle Database Instance

**Create database using DBCA (silent mode):**
```bash
# As oracle user
dbca -silent -createDatabase \
  -templateName General_Purpose.dbc \
  -gdbname VECSRCH \
  -sid VECSRCH \
  -responseFile NO_VALUE \
  -characterSet AL32UTF8 \
  -sysPassword <SysPassword123!> \
  -systemPassword <SystemPassword123!> \
  -createAsContainerDatabase false \
  -databaseType MULTIPURPOSE \
  -automaticMemoryManagement false \
  -totalMemory 20480 \
  -storageType FS \
  -datafileDestination /u01/app/oracle/oradata \
  -recoveryAreaDestination /u01/app/oracle/fast_recovery_area \
  -recoveryAreaSize 50000 \
  -redoLogFileSize 1024 \
  -emConfiguration NONE \
  -sampleSchema false

# Wait for database creation (15-30 minutes)
```

**Configure Listener:**
```bash
# Create listener.ora
cat > $ORACLE_HOME/network/admin/listener.ora << EOF
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = $(hostname))(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = VECSRCH)
      (ORACLE_HOME = /u01/app/oracle/product/23.0.0/dbhome_1)
      (SID_NAME = VECSRCH)
    )
  )

ADR_BASE_LISTENER = /u01/app/oracle
EOF

# Start listener
lsnrctl start

# Configure listener to start automatically
echo "lsnrctl start" >> ~/.bash_profile
```

**Configure tnsnames.ora:**
```bash
cat > $ORACLE_HOME/network/admin/tnsnames.ora << EOF
VECSRCH =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = $(hostname))(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = VECSRCH)
    )
  )
EOF
```

**Configure automatic startup:**
```bash
# Edit /etc/oratab
sudo sed -i 's/VECSRCH:\/u01\/app\/oracle\/product\/23.0.0\/dbhome_1:N/VECSRCH:\/u01\/app\/oracle\/product\/23.0.0\/dbhome_1:Y/' /etc/oratab

# Create systemd service
sudo cat > /etc/systemd/system/oracle-database.service << 'EOF'
[Unit]
Description=Oracle Database Service
After=network.target

[Service]
Type=forking
User=oracle
Group=oinstall
ExecStart=/u01/app/oracle/product/23.0.0/dbhome_1/bin/dbstart /u01/app/oracle/product/23.0.0/dbhome_1
ExecStop=/u01/app/oracle/product/23.0.0/dbhome_1/bin/dbshut /u01/app/oracle/product/23.0.0/dbhome_1
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable oracle-database
sudo systemctl start oracle-database
```

**Verification:**
```bash
# Connect to database
sqlplus / as sysdba

# Check database status
SELECT instance_name, status, version FROM v$instance;
-- Expected: VECSRCH, OPEN, 23.0.0.0.0

# Check AI Vector Search feature
SELECT * FROM v$option WHERE parameter = 'AI Vector Search';
-- Expected: TRUE

EXIT;
```

### Step 2.4: Create Application Schema

**Connect and create schema:**
```sql
sqlplus / as sysdba

-- Create tablespace
CREATE TABLESPACE SEMANTIC_SEARCH_DATA
  DATAFILE '/u01/app/oracle/oradata/VECSRCH/semantic_search01.dbf'
  SIZE 10G AUTOEXTEND ON NEXT 1G MAXSIZE 100G;

-- Create application user
CREATE USER SEMANTIC_SEARCH IDENTIFIED BY "<SecurePassword123!>"
  DEFAULT TABLESPACE SEMANTIC_SEARCH_DATA
  TEMPORARY TABLESPACE TEMP
  QUOTA UNLIMITED ON SEMANTIC_SEARCH_DATA;

-- Grant basic privileges
GRANT CREATE SESSION TO SEMANTIC_SEARCH;
GRANT CREATE TABLE TO SEMANTIC_SEARCH;
GRANT CREATE VIEW TO SEMANTIC_SEARCH;
GRANT CREATE PROCEDURE TO SEMANTIC_SEARCH;
GRANT CREATE SEQUENCE TO SEMANTIC_SEARCH;
GRANT CREATE JOB TO SEMANTIC_SEARCH;
GRANT CREATE DATABASE LINK TO SEMANTIC_SEARCH;

-- Grant DBMS_CLOUD privileges for Azure AI Foundry API calls
GRANT EXECUTE ON DBMS_CLOUD TO SEMANTIC_SEARCH;
GRANT EXECUTE ON DBMS_CLOUD_ADMIN TO SEMANTIC_SEARCH;

-- Grant network privileges (will configure ACLs separately)
GRANT EXECUTE ON DBMS_NETWORK_ACL_ADMIN TO SEMANTIC_SEARCH;

COMMIT;

-- Verify user creation
SELECT username, account_status, default_tablespace
FROM dba_users
WHERE username = 'SEMANTIC_SEARCH';

EXIT;
```

**Configure Network ACLs for Azure AI Foundry:**
```sql
sqlplus / as sysdba

-- Create ACL for Azure AI Foundry access
BEGIN
  DBMS_NETWORK_ACL_ADMIN.CREATE_ACL (
    acl          => 'azure_ai_foundry_acl.xml',
    description  => 'ACL for Azure AI Foundry API access',
    principal    => 'SEMANTIC_SEARCH',
    is_grant     => TRUE,
    privilege    => 'connect',
    start_date   => SYSTIMESTAMP,
    end_date     => NULL
  );
  
  DBMS_NETWORK_ACL_ADMIN.ADD_PRIVILEGE (
    acl          => 'azure_ai_foundry_acl.xml',
    principal    => 'SEMANTIC_SEARCH',
    is_grant     => TRUE,
    privilege    => 'resolve'
  );
  
  -- Assign ACL to Azure AI Foundry endpoints
  DBMS_NETWORK_ACL_ADMIN.ASSIGN_ACL (
    acl          => 'azure_ai_foundry_acl.xml',
    host         => '*.api.azureml.ms',
    lower_port   => 443,
    upper_port   => 443
  );
  
  DBMS_NETWORK_ACL_ADMIN.ASSIGN_ACL (
    acl          => 'azure_ai_foundry_acl.xml',
    host         => '*.openai.azure.com',
    lower_port   => 443,
    upper_port   => 443
  );
  
  COMMIT;
END;
/

-- Verify ACL configuration
SELECT host, lower_port, upper_port, acl
FROM dba_network_acls;

EXIT;
```

### Step 2.11: Configure Azure AI Foundry Private Endpoint (Recommended for Production)

**Duration:** 45 minutes  
**Executor:** Cloud Administrator / Network Administrator

Azure Private Endpoints provide secure, private connectivity to Azure AI Foundry services over your Azure VNet, eliminating exposure to the public internet.

#### Benefits of Private Endpoint

| Benefit | Public Endpoint | Private Endpoint |
|---------|----------------|------------------|
| **Security** | Traffic traverses public internet | Traffic stays within Azure backbone |
| **Compliance** | May not meet data residency requirements | Meets strict compliance requirements |
| **Latency** | Variable (depends on internet routing) | Predictable, lower latency |
| **Data Exfiltration** | Risk of unauthorized data transfer | Network-level controls prevent exfiltration |
| **IP Whitelisting** | Required on Azure AI Foundry | Not required (private IP access) |

#### Prerequisites

```bash
# Ensure Azure CLI is logged in
az login

# Set variables
RESOURCE_GROUP="<your-resource-group>"
LOCATION="<azure-region>"
VNET_NAME="semantic-search-vnet"
SUBNET_NAME="ai-foundry-subnet"
AI_FOUNDRY_WORKSPACE_NAME="<your-ai-foundry-workspace>"
AI_FOUNDRY_RESOURCE_ID=$(az ml workspace show \
  --name $AI_FOUNDRY_WORKSPACE_NAME \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)
```

#### Step 1: Create Subnet for Private Endpoints

```bash
# Create dedicated subnet for private endpoints
# Note: Disable private endpoint network policies on this subnet
az network vnet subnet create \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $SUBNET_NAME \
  --address-prefixes 10.0.5.0/24 \
  --disable-private-endpoint-network-policies true

# Verify subnet creation
az network vnet subnet show \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $SUBNET_NAME \
  --query "{Name:name, AddressPrefix:addressPrefix, PrivateEndpointNetworkPolicies:privateEndpointNetworkPolicies}"
```

#### Step 2: Create Private Endpoint

```bash
# Create private endpoint for Azure AI Foundry workspace
az network private-endpoint create \
  --resource-group $RESOURCE_GROUP \
  --name ai-foundry-private-endpoint \
  --location $LOCATION \
  --vnet-name $VNET_NAME \
  --subnet $SUBNET_NAME \
  --private-connection-resource-id $AI_FOUNDRY_RESOURCE_ID \
  --group-id amlworkspace \
  --connection-name ai-foundry-connection

# Get private endpoint details
PRIVATE_ENDPOINT_IP=$(az network private-endpoint show \
  --resource-group $RESOURCE_GROUP \
  --name ai-foundry-private-endpoint \
  --query "customDnsConfigs[0].ipAddresses[0]" -o tsv)

echo "Private Endpoint IP: $PRIVATE_ENDPOINT_IP"
```

#### Step 3: Create Private DNS Zone

```bash
# Create private DNS zone for Azure Machine Learning (AI Foundry uses ML infrastructure)
az network private-dns zone create \
  --resource-group $RESOURCE_GROUP \
  --name privatelink.api.azureml.ms

# Link DNS zone to VNet
az network private-dns link vnet create \
  --resource-group $RESOURCE_GROUP \
  --zone-name privatelink.api.azureml.ms \
  --name ai-foundry-dns-link \
  --virtual-network $VNET_NAME \
  --registration-enabled false

# Verify DNS zone and link
az network private-dns zone show \
  --resource-group $RESOURCE_GROUP \
  --name privatelink.api.azureml.ms

az network private-dns link vnet list \
  --resource-group $RESOURCE_GROUP \
  --zone-name privatelink.api.azureml.ms \
  --output table
```

#### Step 4: Configure DNS Records

```bash
# Create DNS record group for private endpoint
az network private-endpoint dns-zone-group create \
  --resource-group $RESOURCE_GROUP \
  --endpoint-name ai-foundry-private-endpoint \
  --name ai-foundry-dns-zone-group \
  --private-dns-zone privatelink.api.azureml.ms \
  --zone-name privatelink.api.azureml.ms

# Verify DNS zone group
az network private-endpoint dns-zone-group show \
  --resource-group $RESOURCE_GROUP \
  --endpoint-name ai-foundry-private-endpoint \
  --name ai-foundry-dns-zone-group

# List DNS records (should show A record for workspace)
az network private-dns record-set a list \
  --resource-group $RESOURCE_GROUP \
  --zone-name privatelink.api.azureml.ms \
  --output table
```

#### Step 5: Update Network ACL for Private Endpoint

If you completed Step 2.10 (Azure AI Foundry ACL), update the ACL to include the private endpoint subnet:

```sql
sqlplus / as sysdba

-- Add private endpoint subnet to ACL
BEGIN
  -- Allow connections to private endpoint IP range
  DBMS_NETWORK_ACL_ADMIN.ASSIGN_ACL (
    acl          => 'azure_ai_foundry_acl.xml',
    host         => '10.0.5.*',  -- Private endpoint subnet
    lower_port   => 443,
    upper_port   => 443
  );
  
  COMMIT;
END;
/

-- Verify updated ACL
SELECT host, lower_port, upper_port, acl
FROM dba_network_acls
ORDER BY host;

EXIT;
```

#### Step 6: Update NSG Rules (Already completed in Step 1.1.1)

The NSG rules created in Step 1.1.1 already allow traffic to `AzureCloud.EastUS` service tag, which includes both public and private endpoints. No additional NSG rules are required.

#### Step 7: Verify Private Endpoint Connectivity

**Test DNS Resolution:**
```bash
# SSH to Oracle 23ai VM
ssh <admin_user>@<vm_private_ip>

# Test DNS resolution (should resolve to private IP 10.0.5.x)
nslookup <workspace-name>.<region>.api.azureml.ms

# Example:
# nslookup my-ai-workspace.eastus.api.azureml.ms
# Expected output:
# Name: my-ai-workspace.privatelink.api.azureml.ms
# Address: 10.0.5.4  (private IP, not public)
```

**Test Connectivity:**
```bash
# Test HTTPS connection to private endpoint
curl -v -k https://<workspace-name>.<region>.api.azureml.ms/health

# Example:
# curl -v -k https://my-ai-workspace.eastus.api.azureml.ms/health
# Expected: Connection should succeed via private IP
```

**Test from PL/SQL:**
```sql
sqlplus SEMANTIC_SEARCH/<password>@VECDB

-- Test network connectivity via UTL_HTTP
SET SERVEROUTPUT ON;
DECLARE
  l_req    UTL_HTTP.REQ;
  l_resp   UTL_HTTP.RESP;
  l_url    VARCHAR2(4000);
  l_buffer VARCHAR2(32767);
BEGIN
  -- Construct private endpoint URL
  l_url := 'https://<workspace-name>.<region>.api.azureml.ms/health';
  
  -- Set HTTP version and wallet
  UTL_HTTP.SET_WALLET('');
  
  -- Send request
  l_req := UTL_HTTP.BEGIN_REQUEST(l_url, 'GET', 'HTTP/1.1');
  UTL_HTTP.SET_HEADER(l_req, 'User-Agent', 'Oracle Database');
  
  -- Get response
  l_resp := UTL_HTTP.GET_RESPONSE(l_req);
  
  DBMS_OUTPUT.PUT_LINE('HTTP Status: ' || l_resp.status_code);
  DBMS_OUTPUT.PUT_LINE('Private Endpoint connectivity: SUCCESS');
  
  UTL_HTTP.END_RESPONSE(l_resp);
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
    DBMS_OUTPUT.PUT_LINE('Private Endpoint connectivity: FAILED');
END;
/

EXIT;
```

#### Step 8: Update Stored Procedures for Private Endpoint

The existing procedures (`GENERATE_EMBEDDINGS_BATCH`, `GET_SEMANTIC_RECOMMENDATIONS`) should work without modification. The private endpoint URL is identical to the public URL:

```
Public:  https://<workspace-name>.<region>.api.azureml.ms/...
Private: https://<workspace-name>.<region>.api.azureml.ms/...
         (resolves to 10.0.5.x instead of public IP)
```

No code changes required—DNS automatically routes to private endpoint.

#### Step 9: Disable Public Access (Production Only)

For maximum security, disable public network access to Azure AI Foundry workspace:

```bash
# Disable public network access
az ml workspace update \
  --name $AI_FOUNDRY_WORKSPACE_NAME \
  --resource-group $RESOURCE_GROUP \
  --public-network-access Disabled

# Verify public access is disabled
az ml workspace show \
  --name $AI_FOUNDRY_WORKSPACE_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "{Name:name, PublicNetworkAccess:publicNetworkAccess}"
```

**Important:** After disabling public access:
- All API calls must originate from VNet or peered networks
- Azure Portal access may be limited (use Azure Bastion)
- Testing from local machine will fail unless connected via VPN/ExpressRoute

#### Step 10: Monitor Private Endpoint

**Enable Private Endpoint Diagnostics:**
```bash
# Create Log Analytics workspace (if not exists from NSG setup)
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group $RESOURCE_GROUP \
  --workspace-name semantic-search-logs \
  --query id -o tsv)

# Enable diagnostic logs for private endpoint
PRIVATE_ENDPOINT_ID=$(az network private-endpoint show \
  --resource-group $RESOURCE_GROUP \
  --name ai-foundry-private-endpoint \
  --query id -o tsv)

az monitor diagnostic-settings create \
  --name private-endpoint-diagnostics \
  --resource $PRIVATE_ENDPOINT_ID \
  --workspace $WORKSPACE_ID \
  --logs '[{"category": "PrivateEndpointNetworkActivity", "enabled": true}]'
```

**Query Private Endpoint Logs (Azure Monitor):**
```kusto
// Count requests by result code
AzureDiagnostics
| where Category == "PrivateEndpointNetworkActivity"
| summarize Count=count() by ResultCode, bin(TimeGenerated, 1h)
| render timechart

// Find failed connections
AzureDiagnostics
| where Category == "PrivateEndpointNetworkActivity"
| where ResultCode != 200
| project TimeGenerated, ResourceId, ResultCode, Message
```

#### Performance Comparison: Public vs Private Endpoint

**Benchmark Procedure:**
```sql
sqlplus SEMANTIC_SEARCH/<password>@VECDB

-- Create benchmark table
CREATE TABLE endpoint_latency_test (
  test_id NUMBER PRIMARY KEY,
  endpoint_type VARCHAR2(20),  -- 'PUBLIC' or 'PRIVATE'
  request_time TIMESTAMP,
  response_time TIMESTAMP,
  latency_ms NUMBER,
  status_code NUMBER
);

-- Benchmark procedure
CREATE OR REPLACE PROCEDURE BENCHMARK_ENDPOINT_LATENCY(
  p_endpoint_type IN VARCHAR2,
  p_iterations IN NUMBER DEFAULT 100
) AS
  l_start_time TIMESTAMP;
  l_end_time TIMESTAMP;
  l_latency_ms NUMBER;
  l_status_code NUMBER;
BEGIN
  FOR i IN 1..p_iterations LOOP
    l_start_time := SYSTIMESTAMP;
    
    -- Call embedding generation (small test)
    BEGIN
      GENERATE_EMBEDDINGS_BATCH(
        p_sr_num_start => 'SR-2024-00001',
        p_sr_num_end   => 'SR-2024-00001',
        p_batch_size   => 1
      );
      l_status_code := 200;
    EXCEPTION
      WHEN OTHERS THEN
        l_status_code := 500;
    END;
    
    l_end_time := SYSTIMESTAMP;
    l_latency_ms := EXTRACT(SECOND FROM (l_end_time - l_start_time)) * 1000;
    
    INSERT INTO endpoint_latency_test VALUES (
      i, p_endpoint_type, l_start_time, l_end_time, l_latency_ms, l_status_code
    );
    
    COMMIT;
    
    -- Small delay between requests
    DBMS_LOCK.SLEEP(0.5);
  END LOOP;
END;
/

-- Run benchmark (100 requests each)
EXEC BENCHMARK_ENDPOINT_LATENCY('PUBLIC', 100);
-- Now switch to private endpoint (update DNS or firewall)
EXEC BENCHMARK_ENDPOINT_LATENCY('PRIVATE', 100);

-- Compare results
SELECT 
  endpoint_type,
  COUNT(*) as total_requests,
  ROUND(AVG(latency_ms), 2) as avg_latency_ms,
  ROUND(MIN(latency_ms), 2) as min_latency_ms,
  ROUND(MAX(latency_ms), 2) as max_latency_ms,
  ROUND(PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latency_ms), 2) as p95_latency_ms,
  SUM(CASE WHEN status_code = 200 THEN 1 ELSE 0 END) as successful_requests,
  ROUND(SUM(CASE WHEN status_code = 200 THEN 1 ELSE 0 END) / COUNT(*) * 100, 2) as success_rate_pct
FROM endpoint_latency_test
GROUP BY endpoint_type
ORDER BY endpoint_type;

-- Expected results:
-- PUBLIC:  avg_latency_ms = 80-150ms, p95 = 200ms
-- PRIVATE: avg_latency_ms = 30-60ms,  p95 = 100ms
-- Private endpoint typically shows 40-60% latency improvement
```

#### Private Endpoint Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ Azure VNet (10.0.0.0/16)                                    │
│                                                             │
│  ┌──────────────────────┐         ┌────────────────────┐   │
│  │ Database Subnet      │         │ AI Foundry Subnet  │   │
│  │ (10.0.3.0/24)        │         │ (10.0.5.0/24)      │   │
│  │                      │         │                    │   │
│  │ ┌─────────────────┐  │         │ ┌───────────────┐  │   │
│  │ │ Oracle 23ai VM  │  │         │ │ Private       │  │   │
│  │ │ - PL/SQL        │──┼─────────┼▶│ Endpoint      │  │   │
│  │ │ - ORDS          │  │ Private │ │ (10.0.5.4)    │  │   │
│  │ └─────────────────┘  │ Traffic │ └───────┬───────┘  │   │
│  └──────────────────────┘         │         │          │   │
└────────────────────────────────────┼─────────┼──────────┘   
                                     │         │              
                        Azure Backbone (Private)             
                                     │         │              
                           ┌─────────▼─────────▼──────┐       
                           │ Azure AI Foundry         │       
                           │ (Private Link Service)   │       
                           │ - OpenAI Embeddings      │       
                           │ - No Public IP           │       
                           └──────────────────────────┘       
```

#### Troubleshooting Private Endpoint

**DNS Not Resolving to Private IP:**
```bash
# Check DNS zone link
az network private-dns link vnet show \
  --resource-group $RESOURCE_GROUP \
  --zone-name privatelink.api.azureml.ms \
  --name ai-foundry-dns-link

# Verify DNS records exist
az network private-dns record-set a list \
  --resource-group $RESOURCE_GROUP \
  --zone-name privatelink.api.azureml.ms

# From Oracle VM, check /etc/resolv.conf
cat /etc/resolv.conf
# Should show Azure DNS: nameserver 168.63.129.16
```

**Connection Timeout:**
```bash
# Verify NSG allows outbound to AI Foundry subnet
az network nsg rule show \
  --resource-group $RESOURCE_GROUP \
  --nsg-name database-nsg \
  --name Allow-Azure-AI-Foundry

# Check route table (should not have route blocking private endpoint)
az network nic show-effective-route-table \
  --resource-group $RESOURCE_GROUP \
  --name <oracle-vm-nic-name>
```

**Certificate Validation Errors:**
```sql
-- Oracle Wallet issue - reload certificates
BEGIN
  UTL_HTTP.SET_WALLET('file:/u01/app/oracle/admin/VECDB/wallet', 'WalletPassword123');
END;
/
```

---

## Phase 3: Install and Configure ORDS (Oracle REST Data Services)

**Duration:** 2-3 hours  
**Executor:** Database Administrator  
**Rollback Time:** 1 hour (delete software and configuration)

**Important:** Unlike Autonomous Database, Oracle 23ai on Azure VM does not include pre-configured ORDS. Manual installation and configuration are required.

### Step 3.1: Download ORDS Software

Download Oracle REST Data Services 23.3.0 or later from Oracle Technology Network:

```bash
# Switch to oracle user
sudo su - oracle

# Create ORDS directory structure
mkdir -p /opt/oracle/ords
mkdir -p /opt/oracle/ords/config
mkdir -p /opt/oracle/ords/logs
cd /opt/oracle/ords

# Download ORDS from Oracle Technology Network (OTN)
# Option 1: Download via browser from https://www.oracle.com/database/technologies/appdev/rest.html
# Option 2: Use wget if you have Oracle SSO credentials
wget https://download.oracle.com/otn_software/java/ords/ords-23.3.0.270.1547.zip

# Or upload from your local machine using SCP
# scp ords-23.3.0.270.1547.zip oracle@<vm-ip>:/opt/oracle/ords/
```

**Prerequisites Check:**
```bash
# Verify Java 11 or later is installed
java -version
# Expected: java version "11.0" or later

# If Java is not installed:
sudo yum install -y java-11-openjdk java-11-openjdk-devel

# Set JAVA_HOME if not already set
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk
echo "export JAVA_HOME=/usr/lib/jvm/java-11-openjdk" >> ~/.bash_profile
```

### Step 3.2: Extract and Install ORDS

```bash
# As oracle user
cd /opt/oracle/ords

# Extract ORDS
unzip ords-23.3.0.270.1547.zip
# This creates bin/ and lib/ directories

# Verify extraction
ls -la
# Expected: bin/, lib/, ords-23.3.0.270.1547.zip

# Make ORDS executable
chmod +x bin/ords
```

### Step 3.3: Configure ORDS for Oracle Database

```bash
# As oracle user
cd /opt/oracle/ords

# Run ORDS installation wizard (interactive mode)
bin/ords --config /opt/oracle/ords/config install

# The wizard will prompt for:
# 1. Enter the database hostname [localhost]: localhost
# 2. Enter the database port [1521]: 1521
# 3. Enter the database service name: VECSRCH
# 4. Enter the database username with SYSDBA privileges [SYS]: SYS
# 5. Enter the database password: <SYS_PASSWORD>
# 6. Enter a number to select a feature to enable [1]: 1 (SQL Developer Web)
# 7. Enter a number to select a feature to enable [1]: 2 (REST Enabled SQL)
# 8. Enter a number to select a feature to enable [1]: 4 (Database API)
# 9. Enter a number to select a feature to enable [1]: <Enter> (done)
# 10. Enter the default tablespace for ORDS_METADATA [SYSAUX]: SYSAUX
# 11. Enter the temporary tablespace for ORDS_METADATA [TEMP]: TEMP
# 12. Enter the default tablespace for ORDS_PUBLIC_USER [SYSAUX]: SYSAUX
# 13. Enter the temporary tablespace for ORDS_PUBLIC_USER [TEMP]: TEMP

# ORDS will create schema objects and configuration files
```

**Alternative: Silent Installation with Configuration File**

Create a configuration file for automated installation:

```bash
# Create connection configuration
cat > /opt/oracle/ords/config/databases/default/pool.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<pool>
  <name>default</name>
  <jdbc.url>jdbc:oracle:thin:@localhost:1521/VECSRCH</jdbc.url>
  <jdbc.username>ORDS_PUBLIC_USER</jdbc.username>
  <jdbc.password>!vault!{HmacSHA256}<password_will_be_encrypted></jdbc.password>
  <db.connectionType>basic</db.connectionType>
  <db.hostname>localhost</db.hostname>
  <db.port>1521</db.port>
  <db.servicename>VECSRCH</db.servicename>
</pool>
EOF

# Run silent installation
bin/ords --config /opt/oracle/ords/config install \
  --admin-user SYS \
  --db-hostname localhost \
  --db-port 1521 \
  --db-servicename VECSRCH \
  --feature-db-api true \
  --feature-rest-enabled-sql true \
  --feature-sdw true \
  --log-folder /opt/oracle/ords/logs

# When prompted, enter SYS password
```

**Verify ORDS Installation:**
```sql
-- Connect as SYS
sqlplus / as sysdba

-- Check ORDS metadata schema
SELECT username, account_status FROM dba_users WHERE username IN ('ORDS_METADATA', 'ORDS_PUBLIC_USER');
-- Expected: Both users exist and are OPEN

-- Verify ORDS version
SELECT ords_metadata.ords_version FROM dual;
-- Expected: 23.3.0 or later

EXIT;
```

### Step 3.4: Configure ORDS Standalone Mode

**Configure Jetty (Standalone Web Server):**

```bash
# As oracle user
cd /opt/oracle/ords

# Edit standalone configuration
cat > /opt/oracle/ords/config/global/settings.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<settings>
  <setting id="standalone.http.port">8080</setting>
  <setting id="standalone.static.path">/opt/oracle/ords/config/ords/standalone/doc_root</setting>
  <setting id="standalone.access.log">/opt/oracle/ords/logs</setting>
  <setting id="standalone.context.path">/ords</setting>
  <setting id="jdbc.MaxLimit">50</setting>
  <setting id="jdbc.InitialLimit">10</setting>
  <setting id="debug.printDebugToScreen">false</setting>
</settings>
EOF

# Create document root directory
mkdir -p /opt/oracle/ords/config/ords/standalone/doc_root

# Set file permissions
chmod -R 755 /opt/oracle/ords/config
chmod -R 644 /opt/oracle/ords/config/**/*.xml
```

### Step 3.5: Create ORDS Systemd Service

Create a systemd service for automatic startup:

```bash
# As root user
sudo tee /etc/systemd/system/ords.service > /dev/null << 'EOF'
[Unit]
Description=Oracle REST Data Services
After=network.target oracle-database.service
Requires=oracle-database.service

[Service]
Type=simple
User=oracle
Group=oinstall
Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk"
Environment="ORACLE_HOME=/u01/app/oracle/product/23.0.0/dbhome_1"
ExecStart=/opt/oracle/ords/bin/ords --config /opt/oracle/ords/config serve
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd
sudo systemctl daemon-reload

# Enable ORDS service to start on boot
sudo systemctl enable ords.service

# Start ORDS service
sudo systemctl start ords.service

# Check status
sudo systemctl status ords.service
# Expected: Active: active (running)

# Monitor ORDS logs
sudo journalctl -u ords.service -f
# Expected: "ORDS is ready to accept requests"
```

### Step 3.6: Configure Firewall for ORDS

```bash
# Open port 8080 for ORDS
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload

# Verify port is listening
sudo netstat -tuln | grep 8080
# Expected: tcp 0.0.0.0:8080 LISTEN

# Test ORDS locally
curl http://localhost:8080/ords/
# Expected: HTML page or JSON response from ORDS
```

### Step 3.7: Enable SEMANTIC_SEARCH Schema for REST

```sql
-- Connect as SEMANTIC_SEARCH user
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH

-- Enable schema for ORDS REST services
BEGIN
    ORDS.ENABLE_SCHEMA(
        p_enabled             => TRUE,
        p_schema              => 'SEMANTIC_SEARCH',
        p_url_mapping_type    => 'BASE_PATH',
        p_url_mapping_pattern => 'semantic_search',
        p_auto_rest_auth      => FALSE
    );
    COMMIT;
END;
/

-- Verify schema is REST-enabled
SELECT name, enabled, url_mapping_type, url_mapping_pattern
FROM user_ords_schemas
WHERE name = 'SEMANTIC_SEARCH';

-- Expected output:
-- NAME               ENABLED  URL_MAPPING_TYPE  URL_MAPPING_PATTERN
-- SEMANTIC_SEARCH    TRUE     BASE_PATH         semantic_search

EXIT;
```

### Step 3.8: Test ORDS Installation

**Local Testing:**
```bash
# Test ORDS root endpoint
curl http://localhost:8080/ords/

# Test schema endpoint
curl http://localhost:8080/ords/semantic_search/

# Test from another Azure VM in the same VNet
curl http://<vm-private-ip>:8080/ords/
```

**Test from Azure Portal (Azure Bastion or Serial Console):**
```bash
# From Azure VM with network access
curl -v http://<vm-private-ip>:8080/ords/

# Expected response headers:
# HTTP/1.1 200 OK
# Server: Oracle REST Data Services
```

**Troubleshooting:**
- **Port 8080 not accessible**: Check firewall rules and NSG configuration
- **Connection refused**: Verify ORDS service is running (`systemctl status ords.service`)
- **ORA-12541**: Check listener is running and database is accessible
- **ORDS service fails to start**: Check logs with `journalctl -u ords.service -n 100`
- **Java errors**: Verify JAVA_HOME is set correctly and Java 11+ is installed
- **Database connection errors**: Verify database is running and connection details are correct

**Common Issues and Fixes:**
```bash
# If ORDS won't start, check configuration
cat /opt/oracle/ords/logs/ords.log

# Reset ORDS configuration (if needed)
cd /opt/oracle/ords
rm -rf config/databases/default
bin/ords --config /opt/oracle/ords/config install

# Check ORDS process
ps -ef | grep ords

# Restart ORDS service
sudo systemctl restart ords.service
```

---

### Step 3.9: Advanced ORDS Configuration (Production Recommendations)

**Duration:** 1-2 hours  
**Executor:** Database Administrator / System Administrator

#### 3.9.1. Connection Pool Tuning

Optimize ORDS connection pool for production workloads:

```bash
# Edit pool configuration
vi /opt/oracle/ords/config/databases/default/pool.xml
```

Add the following settings:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<pool>
  <name>default</name>
  
  <!-- Database Connection -->
  <jdbc.url>jdbc:oracle:thin:@localhost:1521/VECSRCH</jdbc.url>
  <jdbc.username>ORDS_PUBLIC_USER</jdbc.username>
  <jdbc.password>!vault!{HmacSHA256}...</jdbc.password>
  
  <!-- Connection Pool Settings (Tuned for Production) -->
  <jdbc.InitialLimit>10</jdbc.InitialLimit>
  <jdbc.MinLimit>10</jdbc.MinLimit>
  <jdbc.MaxLimit>50</jdbc.MaxLimit>
  
  <!-- Connection Timeout (milliseconds) -->
  <jdbc.InactivityTimeout>900</jdbc.InactivityTimeout>
  <jdbc.statementTimeout>900</jdbc.statementTimeout>
  
  <!-- Validation Query -->
  <jdbc.auth.enabled>true</jdbc.auth.enabled>
  
  <!-- Connection Properties -->
  <db.connectionType>basic</db.connectionType>
  <db.hostname>localhost</db.hostname>
  <db.port>1521</db.port>
  <db.servicename>VECSRCH</db.servicename>
</pool>
```

**Connection Pool Sizing Guidelines:**

| Workload | Concurrent Users | InitialLimit | MaxLimit | Recommended |
|----------|------------------|--------------|----------|-------------|
| Development | < 10 | 5 | 20 | Low resource usage |
| QA/UAT | 10-50 | 10 | 30 | Medium load testing |
| Production (Small) | 50-100 | 10 | 50 | Balance performance/resources |
| Production (Medium) | 100-500 | 20 | 100 | High throughput |
| Production (Large) | 500+ | 30 | 150 | Maximum performance |

**Formula for sizing:**
```
MaxLimit = (Expected Concurrent Users × 0.5) rounded up
InitialLimit = MaxLimit × 0.2
MinLimit = InitialLimit
```

#### 3.9.2. ORDS Global Settings Configuration

Edit global settings for optimal performance:

```bash
# Edit global settings
vi /opt/oracle/ords/config/global/settings.xml
```

**Complete settings.xml for Production:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings>
  <!-- HTTP Server Settings -->
  <setting id="standalone.http.port">8080</setting>
  <setting id="standalone.context.path">/ords</setting>
  
  <!-- HTTPS Settings (Production) -->
  <!-- Uncomment for SSL/TLS configuration -->
  <!-- <setting id="standalone.https.port">443</setting> -->
  <!-- <setting id="standalone.https.cert">/opt/oracle/ords/config/ssl/cert.pem</setting> -->
  <!-- <setting id="standalone.https.cert.key">/opt/oracle/ords/config/ssl/key.pem</setting> -->
  
  <!-- Static Files -->
  <setting id="standalone.static.path">/opt/oracle/ords/config/ords/standalone/doc_root</setting>
  
  <!-- Logging -->
  <setting id="standalone.access.log">/opt/oracle/ords/logs</setting>
  <setting id="log.logging">true</setting>
  <setting id="log.maxEntries">500</setting>
  
  <!-- Security -->
  <setting id="security.requestValidationFunction">ords_util.validate_request</setting>
  <setting id="security.disableDefaultExclusionList">false</setting>
  
  <!-- Performance Tuning -->
  <setting id="jdbc.MaxLimit">50</setting>
  <setting id="jdbc.InitialLimit">10</setting>
  <setting id="jdbc.MinLimit">10</setting>
  <setting id="jdbc.InactivityTimeout">900</setting>
  <setting id="jdbc.statementTimeout">900</setting>
  
  <!-- Request Handling -->
  <setting id="request.traceHeaderName">X-ORDS-Trace</setting>
  <setting id="standalone.max.request.header.size">8192</setting>
  
  <!-- Debug (disable in production) -->
  <setting id="debug.printDebugToScreen">false</setting>
  <setting id="debug.debugger">false</setting>
  
  <!-- Error Handling -->
  <setting id="error.responseFormat">JSON</setting>
  <setting id="error.keepErrorMessages">false</setting>
</settings>
```

#### 3.9.3. SSL/TLS Configuration for Production

**Important:** For production, ORDS should use HTTPS instead of HTTP.

**Option A: Using Self-Signed Certificate (Dev/Test)**

```bash
# As oracle user
mkdir -p /opt/oracle/ords/config/ssl
cd /opt/oracle/ords/config/ssl

# Generate self-signed certificate
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=<vm-hostname>"

# Set permissions
chmod 600 key.pem
chmod 644 cert.pem
```

**Option B: Using Azure Key Vault Certificate (Production)**

```bash
# Download certificate from Azure Key Vault
az keyvault secret download \
  --vault-name <keyvault-name> \
  --name ssl-cert \
  --file /opt/oracle/ords/config/ssl/cert.pem

az keyvault secret download \
  --vault-name <keyvault-name> \
  --name ssl-key \
  --file /opt/oracle/ords/config/ssl/key.pem

# Set permissions
chown oracle:oinstall /opt/oracle/ords/config/ssl/*.pem
chmod 600 /opt/oracle/ords/config/ssl/key.pem
chmod 644 /opt/oracle/ords/config/ssl/cert.pem
```

**Update settings.xml for HTTPS:**

```xml
<!-- Add to settings.xml -->
<setting id="standalone.https.port">443</setting>
<setting id="standalone.https.cert">/opt/oracle/ords/config/ssl/cert.pem</setting>
<setting id="standalone.https.cert.key">/opt/oracle/ords/config/ssl/key.pem</setting>
<setting id="standalone.https.host">0.0.0.0</setting>

<!-- Optional: Redirect HTTP to HTTPS -->
<setting id="standalone.http.port">8080</setting>
<setting id="standalone.redirect.http.to.https">true</setting>
```

**Update firewall rules:**

```bash
# Open HTTPS port
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload

# Update NSG to allow HTTPS traffic (Azure Portal or CLI)
az network nsg rule create \
  --resource-group <rg-name> \
  --nsg-name <nsg-name> \
  --name Allow-HTTPS-ORDS \
  --priority 120 \
  --source-address-prefixes '<siebel-subnet-cidr>' \
  --destination-port-ranges 443 \
  --protocol Tcp \
  --access Allow \
  --direction Inbound
```

**Restart ORDS:**

```bash
sudo systemctl restart ords.service

# Test HTTPS
curl -k https://localhost:443/ords/
curl -k https://<vm-private-ip>:443/ords/
```

#### 3.9.4. ORDS Monitoring and Health Checks

**Create health check endpoint:**

```sql
-- Connect as SEMANTIC_SEARCH user
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH

-- Create health check procedure
CREATE OR REPLACE PROCEDURE HEALTH_CHECK (
    p_status OUT VARCHAR2
) AS
    v_db_count NUMBER;
    v_vector_count NUMBER;
BEGIN
    -- Check database connectivity
    SELECT COUNT(*) INTO v_db_count FROM DUAL;
    
    -- Check vector table
    SELECT COUNT(*) INTO v_vector_count 
    FROM SIEBEL_KNOWLEDGE_VECTORS 
    WHERE ROWNUM = 1;
    
    p_status := JSON_OBJECT(
        'status' VALUE 'healthy',
        'timestamp' VALUE SYSTIMESTAMP,
        'database' VALUE 'connected',
        'vector_table' VALUE 'accessible'
    );
EXCEPTION
    WHEN OTHERS THEN
        p_status := JSON_OBJECT(
            'status' VALUE 'unhealthy',
            'error' VALUE SQLERRM
        );
END;
/

-- Create ORDS REST endpoint for health check
BEGIN
    ORDS.DEFINE_MODULE(
        p_module_name    => 'health',
        p_base_path      => 'health/',
        p_items_per_page => 0
    );
    
    ORDS.DEFINE_TEMPLATE(
        p_module_name    => 'health',
        p_pattern        => 'check'
    );
    
    ORDS.DEFINE_HANDLER(
        p_module_name    => 'health',
        p_pattern        => 'check',
        p_method         => 'GET',
        p_source_type    => ORDS.source_type_plsql,
        p_source         => 'BEGIN HEALTH_CHECK(:status); END;',
        p_items_per_page => 0
    );
    
    COMMIT;
END;
/

EXIT;
```

**Test health endpoint:**

```bash
# Test health check
curl http://localhost:8080/ords/semantic_search/health/check

# Expected response:
# {"status":"healthy","timestamp":"2025-10-17T10:30:00Z","database":"connected","vector_table":"accessible"}
```

**Create monitoring script:**

```bash
# Create monitoring script
sudo tee /opt/oracle/ords/monitor_ords.sh > /dev/null << 'EOF'
#!/bin/bash
# ORDS Health Monitoring Script

LOG_FILE="/opt/oracle/ords/logs/monitor.log"
HEALTH_URL="http://localhost:8080/ords/semantic_search/health/check"

# Test ORDS health
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" $HEALTH_URL)

if [ "$HTTP_STATUS" -eq 200 ]; then
    echo "$(date): ORDS is healthy (HTTP $HTTP_STATUS)" >> $LOG_FILE
    exit 0
else
    echo "$(date): ORDS is unhealthy (HTTP $HTTP_STATUS)" >> $LOG_FILE
    # Optional: Send alert or restart service
    # systemctl restart ords.service
    exit 1
fi
EOF

chmod +x /opt/oracle/ords/monitor_ords.sh

# Add to crontab for periodic checks (every 5 minutes)
(crontab -l 2>/dev/null; echo "*/5 * * * * /opt/oracle/ords/monitor_ords.sh") | crontab -
```

#### 3.9.5. ORDS Logging Configuration

**Configure detailed logging for troubleshooting:**

```bash
# Create log configuration file
cat > /opt/oracle/ords/config/global/logging.properties << 'EOF'
# ORDS Logging Configuration

# Root logger
.level=INFO
.handlers=java.util.logging.FileHandler,java.util.logging.ConsoleHandler

# File handler
java.util.logging.FileHandler.pattern=/opt/oracle/ords/logs/ords_%u_%g.log
java.util.logging.FileHandler.limit=10485760
java.util.logging.FileHandler.count=10
java.util.logging.FileHandler.formatter=java.util.logging.SimpleFormatter
java.util.logging.SimpleFormatter.format=%1$tF %1$tT %4$s %2$s %5$s%6$s%n

# Console handler (for systemd journal)
java.util.logging.ConsoleHandler.level=INFO
java.util.logging.ConsoleHandler.formatter=java.util.logging.SimpleFormatter

# ORDS specific loggers
oracle.dbtools.level=INFO
oracle.dbtools.rt.web.level=INFO
oracle.dbtools.rt.resource.level=INFO

# Set to FINE for detailed debugging (disable in production)
# oracle.dbtools.level=FINE
EOF

# Update systemd service to use logging configuration
sudo vi /etc/systemd/system/ords.service
```

**Add to ExecStart:**

```ini
ExecStart=/opt/oracle/ords/bin/ords --config /opt/oracle/ords/config serve --log /opt/oracle/ords/logs/ords.log
```

**Reload and restart:**

```bash
sudo systemctl daemon-reload
sudo systemctl restart ords.service
```

**Log rotation setup:**

```bash
# Create logrotate configuration
sudo tee /etc/logrotate.d/ords > /dev/null << 'EOF'
/opt/oracle/ords/logs/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0644 oracle oinstall
    sharedscripts
    postrotate
        systemctl reload ords.service > /dev/null 2>&1 || true
    endscript
}
EOF
```

#### 3.9.6. ORDS Performance Metrics

**Enable performance monitoring:**

```sql
-- Connect as SEMANTIC_SEARCH user
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH

-- Create performance metrics table
CREATE TABLE ORDS_METRICS (
    metric_id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    metric_timestamp TIMESTAMP DEFAULT SYSTIMESTAMP,
    endpoint VARCHAR2(500),
    response_time_ms NUMBER,
    status_code NUMBER,
    error_message VARCHAR2(4000)
);

-- Create index for performance
CREATE INDEX idx_metrics_timestamp ON ORDS_METRICS(metric_timestamp);

-- Create procedure to log metrics
CREATE OR REPLACE PROCEDURE LOG_ORDS_METRIC (
    p_endpoint IN VARCHAR2,
    p_response_time IN NUMBER,
    p_status_code IN NUMBER DEFAULT 200,
    p_error_message IN VARCHAR2 DEFAULT NULL
) AS
PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
    INSERT INTO ORDS_METRICS (endpoint, response_time_ms, status_code, error_message)
    VALUES (p_endpoint, p_response_time, p_status_code, p_error_message);
    COMMIT;
END;
/

EXIT;
```

**View performance metrics:**

```sql
-- Average response time by endpoint (last 24 hours)
SELECT 
    endpoint,
    COUNT(*) AS request_count,
    ROUND(AVG(response_time_ms), 2) AS avg_response_time_ms,
    ROUND(MIN(response_time_ms), 2) AS min_response_time_ms,
    ROUND(MAX(response_time_ms), 2) AS max_response_time_ms,
    ROUND(PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY response_time_ms), 2) AS p95_response_time_ms
FROM ORDS_METRICS
WHERE metric_timestamp >= SYSTIMESTAMP - INTERVAL '24' HOUR
GROUP BY endpoint
ORDER BY avg_response_time_ms DESC;

-- Error rate (last hour)
SELECT 
    status_code,
    COUNT(*) AS error_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS percentage
FROM ORDS_METRICS
WHERE metric_timestamp >= SYSTIMESTAMP - INTERVAL '1' HOUR
GROUP BY status_code
ORDER BY error_count DESC;
```

---

## Phase 4: Siebel CRM Integration

**Duration:** 2-3 hours  
**Executor:** Siebel Administrator / Developer  
**Rollback Time:** < 1 hour

**Prerequisites:**
- Phase 3 completed (ORDS installed and running on Oracle 23ai VM)
- ORDS endpoint tested and accessible: `http://<oracle-23ai-vm-ip>:8080/ords/`
- Network connectivity confirmed between Siebel CRM and Oracle 23ai VM
- API key configured in SEMANTIC_SEARCH schema

### Step 4.1: Configure Network Connectivity

**Configure Azure NSG Rules:**

```bash
# From Azure Portal or Azure CLI
# Allow Siebel VM to access Oracle 23ai VM on port 8080

az network nsg rule create \
  --resource-group <resource-group-name> \
  --nsg-name <oracle-23ai-vm-nsg> \
  --name Allow-Siebel-ORDS \
  --priority 200 \
  --source-address-prefixes <siebel-vm-private-ip> \
  --destination-address-prefixes <oracle-23ai-vm-private-ip> \
  --destination-port-ranges 8080 \
  --protocol Tcp \
  --access Allow \
  --direction Inbound
```

**Test Connectivity from Siebel VM:**
```bash
# SSH to Siebel VM
ssh siebel@<siebel-vm-ip>

# Test ORDS endpoint
curl -v http://<oracle-23ai-vm-private-ip>:8080/ords/
# Expected: HTTP/1.1 200 OK

# Test semantic search endpoint (should return 401 without API key)
curl http://<oracle-23ai-vm-private-ip>:8080/ords/semantic_search/
# Expected: HTTP/1.1 401 Unauthorized or similar response
```

### Step 4.2: Configure Named Subsystem in Siebel CRM

1. Log in to Siebel Client as Administrator
2. Navigate to: **Administration - Server Configuration > Profile Configuration**
3. Create Named Subsystem: **SemanticSearchORDS**
4. Add parameters:
   - **URL**: `http://<oracle-23ai-vm-private-ip>:8080/ords/semantic_search/siebel/search`
   - **APIKey**: `<API_KEY>` (generated in Phase 3, Step 3.7)
   - **Timeout**: `5000`
   - **MaxRetries**: `2`
   - **ContentType**: `text/plain`
   - **Method**: `POST`

**Verification SQL (run against Siebel Oracle 19c database):**
```sql
-- Connect to Siebel Oracle 19c database
sqlplus siebel/<password>@<siebel-db-host>:1521/<siebel-service-name>

SELECT 
    sp.NAME AS SUBSYSTEM,
    spp.NAME AS PARAMETER,
    spp.VAL AS VALUE
FROM SIEBEL.S_SYS_PROF sp
JOIN SIEBEL.S_SYS_PROF_PROP spp ON sp.ROW_ID = spp.PAR_ROW_ID
WHERE sp.NAME = 'SemanticSearchORDS'
ORDER BY spp.NAME;

-- Expected output:
-- SUBSYSTEM              PARAMETER     VALUE
-- SemanticSearchORDS     APIKey        <your-api-key>
-- SemanticSearchORDS     ContentType   text/plain
-- SemanticSearchORDS     MaxRetries    2
-- SemanticSearchORDS     Method        POST
-- SemanticSearchORDS     Timeout       5000
-- SemanticSearchORDS     URL           http://<oracle-23ai-vm-private-ip>:8080/ords/semantic_search/siebel/search

EXIT;
```

### Step 4.2: Deploy Repository Changes

Follow **TDD 4** completely:

**In Siebel Tools:**

1. Create Business Service:
   - Name: `SemanticSearchAPIService`
   - Import eScript from TDD 4, Step 2.4

2. Compile objects:
   ```
   Tools > Compile > Selected Objects
   ```

3. Check in objects:
   ```
   Tools > Check In
   Select: SemanticSearchAPIService
   Comments: "Semantic search integration - v2.2"
   ```

4. Generate SRF:
   ```bash
   cd <SIEBEL_TOOLS_HOME>/bin
   ./srvrmgr -g <gateway> -e <enterprise> -u SADMIN -p <password>
   
   srvrmgr> list server
   srvrmgr> compile srf for server <SIEBEL_SERVER_NAME>
   srvrmgr> exit
   ```

### Step 4.3: Deploy Web Files

```bash
# Copy custom JavaScript files
cp CatalogSearchPM.js <SIEBEL_WEB_ROOT>/custom/
cp manifest_search.js <SIEBEL_WEB_ROOT>/custom/

# Set permissions
chmod 644 <SIEBEL_WEB_ROOT>/custom/*.js
chown siebel:siebel <SIEBEL_WEB_ROOT>/custom/*.js
```

### Step 4.4: Deploy SRF File

```bash
# Stop Siebel servers
cd <SIEBEL_ROOT>/siebsrvr/bin
./stop_ns

# Backup old SRF
cp ../objects/siebel.srf ../objects/siebel.srf.backup_$(date +%Y%m%d)

# Copy new SRF
cp /tmp/siebel.srf ../objects/

# Start Siebel servers
./start_ns

# Verify servers are running
./srvrmgr -g <gateway> -e <enterprise> -u SADMIN -p <password>
srvrmgr> list comp
# Expected: All components running
```

### Step 4.5: Clear Caches

```bash
# Clear browser cache directories
rm -rf <SIEBEL_WEB_ROOT>/cache/*

# Restart Apache/IIS
systemctl restart httpd
# OR
iisreset
```

---

## 4. Post-Deployment Verification

### 4.1. Database Verification

```sql
-- Connect to Oracle 23ai VM
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH

-- Or use SSH and local connection
ssh oracle@<oracle-23ai-vm-ip>
sqlplus SEMANTIC_SEARCH/<password>

-- Verify data load
SELECT 
    'Staging Records' AS METRIC, COUNT(*) AS COUNT
FROM SIEBEL_NARRATIVES_STAGING
UNION ALL
SELECT 'Vector Records', COUNT(*)
FROM SIEBEL_KNOWLEDGE_VECTORS
UNION ALL
SELECT 'Vectors Generated', COUNT(*)
FROM SIEBEL_KNOWLEDGE_VECTORS
WHERE NARRATIVE_VECTOR IS NOT NULL;

-- Verify vector index
SELECT index_name, status, last_analyzed
FROM user_indexes
WHERE table_name = 'SIEBEL_KNOWLEDGE_VECTORS';

-- Verify scheduled jobs
SELECT job_name, enabled, state, next_run_date
FROM user_scheduler_jobs
WHERE job_name IN ('DAILY_STAGING_REFRESH', 'DAILY_EMBEDDING_GENERATION');

-- Verify database link to Siebel Oracle 19c
SELECT COUNT(*) FROM SIEBEL.S_SRV_REQ@SIEBEL_19C_LINK;
-- Expected: A positive number matching Siebel data count

EXIT;
```

### 4.2. ORDS API Verification

```bash
# Test ORDS health (from any machine with network access to Oracle 23ai VM)
curl http://<oracle-23ai-vm-private-ip>:8080/ords/
# Expected: HTTP/1.1 200 OK with ORDS information

# Test from Oracle 23ai VM itself
ssh oracle@<oracle-23ai-vm-ip>
curl http://localhost:8080/ords/
# Expected: HTML or JSON response from ORDS

# Test semantic search schema endpoint
curl http://<oracle-23ai-vm-private-ip>:8080/ords/semantic_search/
# Expected: HTTP/1.1 200 OK or 401 Unauthorized (both indicate ORDS is working)

# Test search endpoint with API key
curl -X POST \
  -H "Content-Type: text/plain" \
  -H "X-API-Key: <API_KEY>" \
  -H "Top-K: 3" \
  -d "printer not working" \
  http://<oracle-23ai-vm-private-ip>:8080/ords/semantic_search/siebel/search | jq .

# Verify response structure
# Expected fields: search_id, query, timestamp, recommendations[]
# Sample output:
# {
#   "search_id": "550e8400-e29b-41d4-a716-446655440000",
#   "query": "printer not working",
#   "timestamp": "2025-01-17T10:30:45Z",
#   "recommendations": [
#     {
#       "sr_num": "1-12345",
#       "abstract": "Printer driver installation",
#       "similarity_score": 0.92
#     }
#   ]
# }
```

### 4.3. Azure AI Foundry Integration Verification

```sql
-- Connect to Oracle 23ai
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH

-- Test Azure AI Foundry connectivity
SET SERVEROUTPUT ON SIZE UNLIMITED;
DECLARE
    l_test_text VARCHAR2(1000) := 'This is a test query';
    l_response CLOB;
BEGIN
    -- Test embedding generation (should call Azure AI Foundry)
    l_response := GENERATE_EMBEDDING(l_test_text);
    
    DBMS_OUTPUT.PUT_LINE('Response length: ' || DBMS_LOB.GETLENGTH(l_response));
    DBMS_OUTPUT.PUT_LINE('First 100 chars: ' || SUBSTR(l_response, 1, 100));
    
    -- Expected: JSON response with embedding vector (1536 dimensions)
END;
/

-- Verify network ACL allows Azure AI Foundry access
SELECT host, lower_port, upper_port, acl
FROM dba_network_acls
WHERE host LIKE '%api.azureml.ms' OR host LIKE '%openai.azure.com';

-- Expected: ACLs for *.api.azureml.ms and *.openai.azure.com on port 443

EXIT;
```

### 4.3. Siebel Verification

1. Log in to Siebel application
2. Navigate to Service Request screen
3. Enter test query: "laptop battery not charging"
4. Click Search
5. Verify:
   - Loading indicator appears
   - Results display in ranked order
   - No errors in browser console
   - Network tab shows successful API call

### 4.4. End-to-End Test

```bash
# Monitor ORDS logs during Siebel search (SSH to Oracle 23ai VM)
ssh oracle@<oracle-23ai-vm-ip>
tail -f /opt/oracle/ords/logs/ords.log
# Expected: Log entries showing incoming requests during searches

# Or monitor via systemd journal
sudo journalctl -u ords.service -f

# Monitor database API logs
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH

SELECT 
    log_id,
    search_id,
    SUBSTR(query_text, 1, 50) AS query,
    response_time_ms,
    result_count,
    request_timestamp
FROM API_SEARCH_LOG
ORDER BY request_timestamp DESC
FETCH FIRST 10 ROWS ONLY;

-- Expected: Recent search queries from Siebel with timestamps and performance metrics

EXIT;
```

**Performance Baseline:**
- API response time: < 500ms for 90th percentile
- Database query time: < 200ms for vector similarity search
- End-to-end Siebel UI response: < 1 second
- ORDS uptime: 99.9% (systemd automatic restart on failure)

---

## 5. Rollback Procedures

### 5.1. Rollback Siebel Changes

```bash
# Stop Siebel servers
cd <SIEBEL_ROOT>/siebsrvr/bin
./stop_ns

# Restore old SRF
cp ../objects/siebel.srf.backup_<date> ../objects/siebel.srf

# Remove custom files
rm <SIEBEL_WEB_ROOT>/custom/CatalogSearchPM.js
rm <SIEBEL_WEB_ROOT>/custom/manifest_search.js

# Start Siebel servers
./start_ns

# Clear caches
rm -rf <SIEBEL_WEB_ROOT>/cache/*
systemctl restart httpd
```

### 5.2. Rollback ORDS API

**Option 1: Delete ORDS Endpoints Only (Keep Schema Enabled)**

```sql
-- Connect to Oracle 23ai VM
ssh oracle@<oracle-23ai-vm-ip>
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH

-- Delete ORDS handler
BEGIN
    ORDS.DELETE_HANDLER(
        p_module_name => 'siebel_search',
        p_pattern => 'search',
        p_method => 'POST'
    );
    COMMIT;
END;
/

-- Delete template
BEGIN
    ORDS.DELETE_TEMPLATE(
        p_module_name => 'siebel_search',
        p_pattern => 'search'
    );
    COMMIT;
END;
/

-- Delete module
BEGIN
    ORDS.DELETE_MODULE(
        p_module_name => 'siebel_search'
    );
    COMMIT;
END;
/

EXIT;
```

**Option 2: Completely Uninstall ORDS**

```bash
# SSH to Oracle 23ai VM
ssh oracle@<oracle-23ai-vm-ip>

# Stop ORDS service
sudo systemctl stop ords.service
sudo systemctl disable ords.service

# Remove ORDS systemd service
sudo rm /etc/systemd/system/ords.service
sudo systemctl daemon-reload

# Remove ORDS software and configuration
sudo rm -rf /opt/oracle/ords

# Close firewall port
sudo firewall-cmd --permanent --remove-port=8080/tcp
sudo firewall-cmd --reload

# Drop ORDS metadata schemas from database
sqlplus / as sysdba

DROP USER ORDS_METADATA CASCADE;
DROP USER ORDS_PUBLIC_USER CASCADE;

EXIT;
```

### 5.3. Rollback Database Changes

```sql
-- Connect as SEMANTIC_SEARCH user
ssh oracle@<oracle-23ai-vm-ip>
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH

-- Drop scheduler jobs
BEGIN
    DBMS_SCHEDULER.DROP_JOB('DAILY_STAGING_REFRESH', force => TRUE);
    DBMS_SCHEDULER.DROP_JOB('DAILY_EMBEDDING_GENERATION', force => TRUE);
END;
/

-- Drop database link to Siebel Oracle 19c
DROP DATABASE LINK SIEBEL_19C_LINK;

-- Drop vector index
DROP INDEX SIBL_KNOW_VEC_HNSW_IDX;

-- Drop tables (optional - only if full rollback needed)
DROP TABLE SIEBEL_KNOWLEDGE_VECTORS PURGE;
DROP TABLE SIEBEL_NARRATIVES_STAGING PURGE;
DROP TABLE STG_S_SRV_REQ PURGE;
DROP TABLE STG_S_ORDER PURGE;
DROP TABLE STG_S_EVT_ACT PURGE;
DROP TABLE API_SEARCH_LOG PURGE;

EXIT;
```

### 5.4. Rollback Azure AI Foundry Configuration

```sql
-- Connect as SEMANTIC_SEARCH user
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH

-- Drop Azure AI Foundry credentials
BEGIN
    DBMS_CLOUD.DROP_CREDENTIAL('AZURE_AI_FOUNDRY_CRED');
END;
/

-- Remove network ACLs
BEGIN
    DBMS_NETWORK_ACL_ADMIN.DROP_ACL(acl => 'azure_ai_foundry_acl.xml');
    COMMIT;
END;
/

EXIT;
```

### 5.5. Rollback Azure Infrastructure (If Needed)

**WARNING:** Only perform this if you want to completely remove the Oracle 23ai VM.

```bash
# Delete Azure VM
az vm delete \
  --resource-group <resource-group-name> \
  --name <oracle-23ai-vm-name> \
  --yes

# Delete managed disks
az disk delete \
  --resource-group <resource-group-name> \
  --name <oracle-23ai-data-disk> \
  --yes

az disk delete \
  --resource-group <resource-group-name> \
  --name <oracle-23ai-backup-disk> \
  --yes

# Delete network interface
az network nic delete \
  --resource-group <resource-group-name> \
  --name <oracle-23ai-vm-nic>

# Delete NSG rules (optional)
az network nsg rule delete \
  --resource-group <resource-group-name> \
  --nsg-name <oracle-23ai-vm-nsg> \
  --name Allow-Siebel-ORDS
```

---
DROP TABLE SEMANTIC_SEARCH.STG_S_PROD_INT PURGE;

-- Drop user (only if complete removal needed)
DROP USER SEMANTIC_SEARCH CASCADE;
```

---

## 6. Troubleshooting Common Deployment Issues

| Issue | Symptom | Solution |
|-------|---------|----------|
| **VM Connection Issues** | Cannot SSH to Oracle 23ai VM | Verify Azure NSG rules allow SSH (port 22), check VM is running, verify SSH key permissions |
| **Database Link Fails** | ORA-12154 or ORA-12541 | Verify inline connection descriptor syntax, check Oracle 19c listener on Siebel VM, verify network connectivity between VMs |
| **Oracle 23ai Installation Fails** | runInstaller errors | Check Java version (11+), verify storage space, review installation logs in /tmp |
| **ORDS Won't Start** | `systemctl status ords` shows failed | Check Java installation, verify database is running, review `/opt/oracle/ords/logs/ords.log` |
| **ORDS Port Not Accessible** | Connection refused on port 8080 | Verify firewall rules (`firewall-cmd --list-ports`), check NSG allows port 8080, verify ORDS is listening (`netstat -tuln \| grep 8080`) |
| **Azure AI Foundry API Fails** | ORA-29532, HTTP 401/403 | Verify network ACLs configured, check API key validity, test connectivity from VM (`curl https://<workspace>.api.azureml.ms`), verify private endpoint |
| **Embedding Generation Slow** | Timeout errors | Check Azure AI Foundry quota limits, verify network latency to AI Foundry endpoint, consider batch size optimization |
| **API Returns 404** | Endpoint not found | Verify ORDS schema is enabled (`SELECT * FROM user_ords_schemas`), check module/template creation |
| **Siebel Search Hangs** | Timeout after 5 seconds | Check `API_SEARCH_LOG` for errors, verify ORDS is accessible from Siebel VM (`curl http://<oracle-23ai-ip>:8080/ords/`), check Named Subsystem URL |
| **No Search Results** | Empty recommendations array | Verify vector index exists and is valid, check data in `SIEBEL_KNOWLEDGE_VECTORS` table, verify embeddings are not NULL |
| **Vector Index Creation Fails** | ORA-00955 or performance issues | Ensure all embeddings are generated before creating index, check tablespace has sufficient space, verify vector dimensions match (1536) |
| **Database Link to Siebel Fails** | Cannot query Siebel tables | Verify Oracle 19c is accessible, check credentials for `siebel_readonly` user, test TNS connectivity |
| **DBMS_CLOUD Calls Fail** | ORA-20401 or network errors | Verify network ACLs are configured for `*.api.azureml.ms` and `*.openai.azure.com`, check credentials exist |

---

## 7. Post-Deployment Tasks

### 7.1. Enable Monitoring and Alerting

**Database Performance Monitoring:**

```sql
-- Connect to Oracle 23ai
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH

-- Create monitoring view
CREATE OR REPLACE VIEW V_SEARCH_METRICS AS
SELECT 
    TRUNC(request_timestamp, 'HH24') AS hour,
    COUNT(*) AS search_count,
    ROUND(AVG(response_time_ms), 2) AS avg_response_ms,
    ROUND(MAX(response_time_ms), 2) AS max_response_ms,
    ROUND(PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY response_time_ms), 2) AS p95_response_ms,
    COUNT(CASE WHEN error_message IS NOT NULL THEN 1 END) AS error_count,
    ROUND(COUNT(CASE WHEN error_message IS NOT NULL THEN 1 END) * 100.0 / COUNT(*), 2) AS error_rate_pct
FROM API_SEARCH_LOG
WHERE request_timestamp >= SYSDATE - 7
GROUP BY TRUNC(request_timestamp, 'HH24')
ORDER BY hour DESC;

-- Query hourly metrics
SELECT * FROM V_SEARCH_METRICS WHERE hour >= SYSDATE - 1;

EXIT;
```

**ORDS and System Monitoring:**

```bash
# SSH to Oracle 23ai VM
ssh oracle@<oracle-23ai-vm-ip>

# Monitor ORDS service status
sudo systemctl status ords.service

# Monitor ORDS logs for errors
sudo journalctl -u ords.service --since "1 hour ago" | grep -i error

# Monitor database listener
lsnrctl status

# Check disk space (should maintain 20%+ free)
df -h /u01
df -h /u02

# Monitor CPU and memory usage
top -b -n 1 | head -20

# Create monitoring script
cat > /home/oracle/monitor_health.sh << 'EOF'
#!/bin/bash
echo "=== Oracle 23ai Health Check ==="
echo "Date: $(date)"
echo ""
echo "--- ORDS Status ---"
systemctl is-active ords.service
echo ""
echo "--- Database Status ---"
ps -ef | grep pmon | grep -v grep
echo ""
echo "--- Disk Usage ---"
df -h /u01 /u02 | grep -v Filesystem
echo ""
echo "--- ORDS Port ---"
netstat -tuln | grep 8080
echo ""
echo "--- Recent API Errors (last hour) ---"
journalctl -u ords.service --since "1 hour ago" | grep -i error | tail -5
EOF

chmod +x /home/oracle/monitor_health.sh

# Run health check
./monitor_health.sh
```

**Azure Monitor Integration:**

```bash
# Install Azure Monitor Agent on Oracle 23ai VM
# From Azure Portal: Monitor > Virtual Machines > <oracle-23ai-vm> > Insights > Enable

# Configure custom metrics collection
# Monitor: 
# - CPU usage > 80%
# - Memory usage > 85%
# - Disk space < 20% free
# - ORDS service down
# - Database listener down
```

### 7.2. Schedule Maintenance

- Review **TDD 1, Step 6** for data refresh schedule
- Review **TDD 2, Step 4.3** for embedding generation schedule
- Set up alerting for failed jobs

### 7.3. Document Environment-Specific Information

Create a deployment record with:
- Deployment date/time
- Version numbers
- Configuration values used
- Test results
- Known issues
- Contact information

---

## 8. Success Criteria

Deployment is considered successful when:

- ✅ All database tables populated with data
- ✅ Vector index created successfully
- ✅ ORDS API responds with valid JSON
- ✅ Siebel search returns ranked results
- ✅ End-to-end search completes in < 3 seconds
- ✅ No errors in logs
- ✅ Scheduled jobs running successfully
- ✅ Rollback procedures tested and documented

---

## 9. Next Steps

After successful deployment:
1. Conduct User Acceptance Testing (see **Testing Guide**)
2. Train end users on new search functionality
3. Monitor performance and user feedback
4. Plan for production go-live
5. Schedule post-deployment review meeting
