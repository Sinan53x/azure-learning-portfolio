# Project 1: File Upload Processing with Functions and Blob Storage

## 📋 Project Overview

**Goal**: Build a serverless file processing system that automatically triggers when files are uploaded to Azure Blob Storage.

**What I Learned**: 
- How Azure Functions work with blob triggers
- Event-driven architecture basics
- Serverless computing concepts
- Azure CLI resource management

**Architecture**: Serverless file processing pipeline using Azure Functions triggered by blob uploads.

---

## 🏗️ Architecture

```
User uploads file → Blob Storage → Blob Trigger → Azure Function processes file
     ↑                                                              ↓
     └──────────────────────────────────────────────────────────────┘
                    (processed file stored back or logged)
```

**Components**:
1. **Azure Storage Account** - Stores uploaded files
2. **Blob Container** - Organizes files by type/purpose
3. **Azure Function App** - Serverless compute environment
4. **Blob Trigger Function** - Executes when new files arrive

---

## 🚀 Implementation Steps

### Prerequisites Checklist
- [ ] Azure account with active subscription
- [ ] Azure CLI installed (check with `az --version`)
- [ ] Basic understanding of serverless concepts

### Step 1: Environment Setup

```bash
# Set environment variables
export RESOURCE_GROUP="rg-file-processing-${RANDOM_SUFFIX}"
export LOCATION="switzerlandnorth"  # Swiss region
export SUBSCRIPTION_ID=$(az account show --query id --output tsv)

# Generate random suffix (automatic!)
RANDOM_SUFFIX=$(openssl rand -hex 3)

# Set resource names
export STORAGE_ACCOUNT="stfileproc${RANDOM_SUFFIX}"
export FUNCTION_APP="func-file-processing-${RANDOM_SUFFIX}"
export CONTAINER_NAME="uploads"

# Create resource group
az group create \
    --name ${RESOURCE_GROUP} \
    --location ${LOCATION} \
    --tags purpose=recipe environment=demo
```

**What this does**: Creates a container for all project resources in the Switzerland North region.

---

### Step 2: Create Storage Account

```bash
# Create storage account
az storage account create \
    --name ${STORAGE_ACCOUNT} \
    --resource-group ${RESOURCE_GROUP} \
    --location ${LOCATION} \
    --sku Standard_LRS \
    --kind StorageV2 \
    --access-tier Hot

# Get connection string
STORAGE_CONNECTION=$(az storage account show-connection-string \
    --name ${STORAGE_ACCOUNT} \
    --resource-group ${RESOURCE_GROUP} \
    --query connectionString \
    --output tsv)
```

**Key Concepts**:
- `Standard_LRS` = Locally redundant storage (3 copies in one datacenter)
- `StorageV2` = Latest storage account type with all features
- `Hot` tier = Optimized for frequently accessed data

---

### Step 3: Create Blob Container

```bash
az storage container create \
    --name ${CONTAINER_NAME} \
    --connection-string "${STORAGE_CONNECTION}" \
    --public-access off
```

**What this does**: Creates a private container where uploaded files will be stored.

---

### Step 4: Create Function App

```bash
az functionapp create \
    --name ${FUNCTION_APP} \
    --resource-group ${RESOURCE_GROUP} \
    --storage-account ${STORAGE_ACCOUNT} \
    --consumption-plan-location ${LOCATION} \
    --runtime node \
    --runtime-version 18 \
    --functions-version 4

# Wait for provisioning
sleep 30
```

**Key Concepts**:
- `consumption-plan` = Pay only when function runs (serverless)
- `runtime node` = Uses Node.js 18
- `functions-version 4` = Latest Functions runtime

---

### Step 5: Configure Storage Connection

```bash
az functionapp config appsettings set \
    --name ${FUNCTION_APP} \
    --resource-group ${RESOURCE_GROUP} \
    --settings "STORAGE_CONNECTION_STRING=${STORAGE_CONNECTION}"
```

**Why**: The function needs to know which storage account to monitor for blob uploads.

---

### Step 6: Create the Function Code

Create `function-code/function.json`:
```json
{
  "bindings": [
    {
      "name": "myBlob",
      "type": "blobTrigger",
      "direction": "in",
      "path": "uploads/{name}",
      "connection": "STORAGE_CONNECTION_STRING"
    }
  ]
}
```

Create `function-code/index.js`:
```javascript
module.exports = async function (context, myBlob) {
    const fileName = context.bindingData.name;
    const fileSize = myBlob.length;
    const timestamp = new Date().toISOString();
    
    context.log(`Processing file: ${fileName}`);
    context.log(`File size: ${fileSize} bytes`);
    context.log(`Processing time: ${timestamp}`);
    
    // Check file type and log accordingly
    if (fileName.toLowerCase().includes('.jpg') || fileName.toLowerCase().includes('.png')) {
        context.log(`Image file detected: ${fileName}`);
    } else if (fileName.toLowerCase().includes('.pdf')) {
        context.log(`PDF document detected: ${fileName}`);
    } else {
        context.log(`Generic file detected: ${fileName}`);
    }
    
    context.log(`File processing completed for: ${fileName}`);
};
```

**How it works**: 
- The `blobTrigger` listens to the `uploads` container
- When a file is uploaded, the function automatically executes
- The function receives the file content and metadata

---

### Step 7: Deploy the Function

```bash
cd function-code
zip -r ../function-deploy.zip .
cd ..

az functionapp deployment source config-zip \
    --name ${FUNCTION_APP} \
    --resource-group ${RESOURCE_GROUP} \
    --src function-deploy.zip

sleep 15
```

---

## ✅ Testing

### Create Test Files
```bash
echo "This is a test file for Azure Functions processing" > test-file.txt
echo "Test image content simulation" > test-image.jpg
```

### Upload Files
```bash
az storage blob upload \
    --file test-file.txt \
    --name "test-document-$(date +%s).txt" \
    --container-name ${CONTAINER_NAME} \
    --connection-string "${STORAGE_CONNECTION}"

az storage blob upload \
    --file test-image.jpg \
    --name "test-image-$(date +%s).jpg" \
    --container-name ${CONTAINER_NAME} \
    --connection-string "${STORAGE_CONNECTION}"
```

### View Logs
```bash
az webapp log tail \
    --name ${FUNCTION_APP} \
    --resource-group ${RESOURCE_GROUP}
```

You should see log messages showing the files being processed!

---

## 🧹 Cleanup

```bash
# Delete all Azure resources
az group delete \
    --name ${RESOURCE_GROUP} \
    --yes \
    --no-wait

# Clean up local files
rm -rf function-code
rm -f function-deploy.zip test-file.txt test-image.jpg
```

---

## 💡 What I Learned

### Key Takeaways
1. **Serverless computing**: You write code, Azure manages the infrastructure
2. **Event-driven**: Functions trigger automatically when events occur (file upload)
3. **Cost-effective**: Consumption plan means pay-per-execution, not per-hour
4. **Scalability**: Automatically scales from 0 to thousands of instances

### Challenges Faced
- *Document any issues you encountered here*
- *How you solved them*

### Next Steps
- [ ] Add file type validation
- [ ] Implement different processing for different file types
- [ ] Add error handling and retry logic
- [ ] Set up monitoring with Application Insights

---

## 📸 Screenshots

*[Add screenshots of Azure Portal showing your resources here]*

---

## 🔗 Resources

- Original Project: [CloudProjects.dev](https://cloudprojects.dev)
- Azure Functions Docs: https://docs.microsoft.com/en-us/azure/azure-functions/
- Blob Storage Docs: https://docs.microsoft.com/en-us/azure/storage/blobs/

---

**Status**: 🚧 In Progress  
**Time Spent**: ~30 minutes  
**Cost**: ~$0.10 (mostly within free tier)
