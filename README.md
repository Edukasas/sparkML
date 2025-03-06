# SparkSQL [databricks]

## Prerequisites

Before proceeding, ensure you have the following tools installed:

- Azure CLI (az) – Used to interact with Azure services and manage resources.
- Terraform – Infrastructure as Code (IaC) tool for provisioning Azure resources.

## 1. Create a Storage Account in Azure for Terraform State

Terraform requires a remote backend to store its state file. Follow these steps:

### **Option 1: Using Azure CLI [Recommended]**

1. **Log in to Azure:**

```bash
az login
```

This will open a browser for authentication.

2. **Select Your Azure Subscription (if multiple exist):**

```bash
az account set --subscription "<YOUR_SUBSCRIPTION_ID>"
```

3. **Create a Resource Group:**

This is only used for storing Terraform state files. Terraform will create the necessary resource groups later.
```bash
az group create --name <RESOURCE_GROUP_NAME> --location <AZURE_REGION>
```

4. **Create a Storage Account:**

```bash
az storage account create --name <STORAGE_ACCOUNT_NAME> --resource-group <RESOURCE_GROUP_NAME> --location <AZURE_REGION> --sku Standard_LRS
```

5. **Create a Storage Container:**

```bash
az storage container create --name <CONTAINER_NAME> --account-name <STORAGE_ACCOUNT_NAME>
```

### **Option 2: Using Azure Portal (Web UI)**
1. **Log in to [Azure Portal](https://portal.azure.com/)**
2. Navigate to **Resource Groups** and click **Create**.
3. Enter a **Resource Group Name**, select a **Region**, and click **Review + Create**.
4. Once the Resource Group is created, go to **Storage Accounts** and click **Create**.
5. Fill in the required details:
   - **Storage Account Name**
   - **Resource Group** (Select the one you just created)
   - **Region** (Choose your preferred region)
   - **Performance**: Standard
   - **Redundancy**: Locally Redundant Storage (LRS)
6. Click **Review + Create** and then **Create**.
7. Once created, go to the **Storage Account** → **Data Storage** → **Containers** → Click **+ Container**.
8. Name it `tfstate`  (as example) and set **Access Level** to Private.
9. To get `<STORAGE_ACCOUNT_KEY>`: Navigate to your **Storage Account** →**Security & Networking** → **Access Keys**. Press `show` button on `key1`

## 2. Get Your Azure Subscription ID

### **Option 1: Using Azure Portal (Web UI)**

1. **Go to [Azure Portal](https://portal.azure.com/)**
2. Click on **Subscriptions** in the left-hand menu.
3. You will see a list of your subscriptions.
4. Choose the subscription you want to use and copy the **Subscription ID**.

### **Option 2: Using Azure CLI**

Retrieve it using Azure CLI:

```bash
az account show --query id --output tsv
```

## 3. Update Terraform Configuration

Modify `main.tf` and replace placeholders with your actual values.

- Get a Storage Account Key (`<STORAGE_ACCOUNT_KEY>`):

```bash
az storage account keys list --resource-group <RESOURCE_GROUP_NAME> --account-name <STORAGE_ACCOUNT_NAME> --query "[0].value"
```

- **Edit the backend block in `main.tf`:**

```hcl
  terraform {
    backend "azurerm" {
      resource_group_name  = "<RESOURCE_GROUP_NAME>"
      storage_account_name = "<STORAGE_ACCOUNT_NAME>"
      container_name       = "<CONTAINER_NAME>"
      key                  = "<STORAGE_ACCOUNT_KEY>"
    }
  }
  provider "azurerm" {
    features {}
    subscription_id = "<SUBSCRIPTION_ID>"
  }
```

## 4. Deploy Infrastructure with Terraform

Run the following Terraform commands:

```bash
terraform init
```  

```bash
terraform plan -out terraform.plan
```  

```bash
terraform apply terraform.plan
```  

To see the `<RESOURCE_GROUP_NAME_CREATED_BY_TERRAFORM>` (resource group that was created by terraform), run the command:

```bash
terraform output resource_group_name
```

## 5. Verify Resource Deployment in Azure

After Terraform completes, verify that resources were created:

1. **Go to the [Azure Portal](https://portal.azure.com/)**
2. Navigate to **Resource Groups** → **Find `<RESOURCE_GROUP_NAME_CREATED_BY_TERRAFORM>`**
3. Check that the resources (Storage Account, Databricks, etc.) are created.

Alternatively, check via CLI:

```bash
az resource list --resource-group <RESOURCE_GROUP_NAME_CREATED_BY_TERRAFORM> --output table
```

## 6. Launch notebooks on Databricks cluster

Follow the steps below to locate and access your Databricks workspace in the Azure Portal.

1. **Retrieve the Resource Group Name**

After running Terraform, you can find the resource group name by checking the Terraform output:

```bash
terraform output resource_group_name
```

Take note of this name for the next steps.

2. **Sign in to Azure Portal**

- Open your web browser and navigate to the Azure Portal: [https://portal.azure.com](https://portal.azure.com)
- Log in with your Azure credentials.

3. **Locate the Resource Group**

- In the Azure Portal, search for “Resource groups” in the top search bar.
- Click on Resource groups in the search results.
- Find the resource group from the Terraform output (e.g., `rg-<ENV>-<LOCATION>-<random_suffix>`).
- Click on the resource group name to open its details page.

4. **Find the Databricks Workspace**

- Inside the Resource Group, look for a resource with the Type: `Azure Databricks Service`.
- The Name of the Databricks workspace should be similar to: `dbw-<ENV>-<LOCATION>-<random_suffix>`
- Click on the Databricks workspace name.

5. **Launch Databricks**

- On the Databricks workspace overview page, locate the Launch Workspace button.
- Click Launch Workspace – this will redirect you to the Databricks portal.
- Sign in if prompted using your Azure credentials.

6. **Start Using Databricks**

Once inside the Databricks UI, you can create clusters, notebooks, and start working with your data.

- Open compute tab.
- Press `Create compute` button.
- Setting up the cluster settings: choose `Single Node`, in `Databricks Runtime Version` **use ML preset** (!), unset the `Use Photon Accseleration` then press the button `Create compute` (it will take some time 8-10 min).
- Create notebooks, write code and launch them on created Databricks cluster

7. **Upload data into DBFS**

- From the Databricks UI navigate into `Catalog`
- Choose `Add data` and press (you will be redirected in a new place)
- Look for `Upload files to DBFS` and press
- Now you're able to upload any `data` here

## 7. Destroy Infrastructure (Optional)

To remove all deployed resources, run:

```bash
terraform destroy
```
