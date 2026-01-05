# Azure Web App Deployment Guide

This guide walks you through deploying your AI Assistant application to Azure Web App.

## Prerequisites

- Azure subscription
- Azure CLI installed (https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
- Git installed
- Python 3.9+ installed locally

## Files Modified for Azure Deployment

### 1. **requirements.txt**
   - Generated from your virtual environment
   - Added `gunicorn==23.0.0` for production server

### 2. **main.py**
   - Updated root endpoint `/` to serve the UI
   - Added comments for CORS configuration
   - Uses relative URLs compatible with Azure

### 3. **static/index.html**
   - Changed from `http://localhost:8000/ask-rag` to `/ask-rag`
   - Changed from `http://localhost:8000/upload-document` to `/upload-document`
   - Now works with any domain

### 4. **startup.txt**
   - Gunicorn configuration for Azure App Service
   - 4 workers with Uvicorn worker class
   - 600-second timeout for long-running requests

### 5. **.gitignore**
   - Excludes virtual environment, .env files, and temporary files

## Deployment Steps

### Step 1: Prepare Your Azure Environment Variables

Your application requires the following environment variables. You'll configure these in Azure App Service:

```bash
AZURE_OPENAI_ENDPOINT=<your-azure-openai-endpoint>
AZURE_OPENAI_API_KEY=<your-api-key>
AZURE_OPENAI_API_VERSION=<api-version>
AZURE_OPENAI_DEPLOYMENT=<your-deployment-name>
AZURE_OPENAI_EMBEDDING_DEPLOYMENT=<your-embedding-deployment-name>
```

### Step 2: Create Azure Web App

#### Option A: Using Azure Portal

1. Go to Azure Portal (https://portal.azure.com)
2. Click "Create a resource"
3. Search for "Web App" and click "Create"
4. Fill in the details:
   - **Subscription**: Select your subscription
   - **Resource Group**: Create new or use existing
   - **Name**: Choose a unique name (e.g., `td-ai-assistant`)
   - **Publish**: Code
   - **Runtime stack**: Python 3.11 (or 3.10)
   - **Operating System**: Linux
   - **Region**: Choose closest to your users
   - **Pricing Plan**: Select appropriate plan (B1 or higher recommended)
5. Click "Review + Create" then "Create"

#### Option B: Using Azure CLI

```bash
# Login to Azure
az login

# Create a resource group (if you don't have one)
az group create --name myResourceGroup --location eastus

# Create an App Service plan
az appservice plan create \
  --name myAppServicePlan \
  --resource-group myResourceGroup \
  --sku B1 \
  --is-linux

# Create the web app
az webapp create \
  --resource-group myResourceGroup \
  --plan myAppServicePlan \
  --name td-ai-assistant \
  --runtime "PYTHON|3.11"
```

### Step 3: Configure Environment Variables

#### Using Azure Portal:

1. Navigate to your Web App in Azure Portal
2. Go to **Settings** > **Configuration**
3. Click **New application setting** for each variable:
   - Name: `AZURE_OPENAI_ENDPOINT`, Value: `<your-endpoint>`
   - Name: `AZURE_OPENAI_API_KEY`, Value: `<your-key>`
   - Name: `AZURE_OPENAI_API_VERSION`, Value: `<version>`
   - Name: `AZURE_OPENAI_DEPLOYMENT`, Value: `<deployment-name>`
   - Name: `AZURE_OPENAI_EMBEDDING_DEPLOYMENT`, Value: `<embedding-name>`
4. Click **Save**

#### Using Azure CLI:

```bash
az webapp config appsettings set \
  --resource-group myResourceGroup \
  --name td-ai-assistant \
  --settings \
  AZURE_OPENAI_ENDPOINT="<your-endpoint>" \
  AZURE_OPENAI_API_KEY="<your-key>" \
  AZURE_OPENAI_API_VERSION="<version>" \
  AZURE_OPENAI_DEPLOYMENT="<deployment>" \
  AZURE_OPENAI_EMBEDDING_DEPLOYMENT="<embedding>"
```

### Step 4: Configure Startup Command

#### Using Azure Portal:

1. Go to **Settings** > **Configuration**
2. Click **General settings** tab
3. In **Startup Command**, enter:
   ```
   gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app --bind 0.0.0.0:8000 --timeout 600
   ```
4. Click **Save**

#### Using Azure CLI:

```bash
az webapp config set \
  --resource-group myResourceGroup \
  --name td-ai-assistant \
  --startup-file "gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app --bind 0.0.0.0:8000 --timeout 600"
```

### Step 5: Deploy Your Application

#### Option A: Deploy from Local Git

```bash
# Initialize git repository (if not already done)
git init
git add .
git commit -m "Initial commit for Azure deployment"

# Configure Azure deployment user (one-time setup)
az webapp deployment user set \
  --user-name <username> \
  --password <password>

# Get the deployment URL
az webapp deployment source config-local-git \
  --name td-ai-assistant \
  --resource-group myResourceGroup

# Add Azure as a git remote (use the URL from previous command)
git remote add azure <deployment-git-url>

# Push to Azure
git push azure master
```

#### Option B: Deploy from GitHub

1. Push your code to a GitHub repository
2. In Azure Portal, go to your Web App
3. Go to **Deployment** > **Deployment Center**
4. Select **GitHub** as source
5. Authorize Azure to access your GitHub
6. Select your repository and branch
7. Click **Save**

Azure will automatically deploy whenever you push to the selected branch.

#### Option C: Deploy using VS Code

1. Install Azure App Service extension in VS Code
2. Sign in to Azure
3. Right-click on your project folder
4. Select **Deploy to Web App**
5. Choose your subscription and Web App
6. Confirm deployment

#### Option D: Deploy using ZIP file

```bash
# Create a ZIP file of your project (excluding .venv and .git)
# Make sure to include: main.py, static/, documents/, requirements.txt

# Deploy the ZIP
az webapp deployment source config-zip \
  --resource-group myResourceGroup \
  --name td-ai-assistant \
  --src <path-to-your-zip-file>
```

### Step 6: Verify Deployment

1. Wait 5-10 minutes for deployment to complete
2. Open your browser and navigate to:
   ```
   https://td-ai-assistant.azurewebsites.net
   ```
3. You should see your AI Assistant UI
4. Check the logs in Azure Portal: **Monitoring** > **Log stream**

## Important Considerations

### 1. Vector Store Persistence

**Current Setup**: The application uses Chroma in-memory mode, which means:
- Vector store is rebuilt on every restart
- Uploaded documents are lost on restart

**Production Recommendation**:
- Use a persistent vector database (Azure Cosmos DB, Azure AI Search, or Chroma with persistent storage)
- Store uploaded documents in Azure Blob Storage
- Implement a background job to rebuild the vector store

### 2. File Storage for Documents

**Current Setup**: Documents are loaded from the `documents/` folder

**Azure Options**:
- **Option A**: Include documents in deployment (current approach)
- **Option B**: Store documents in Azure Blob Storage and download on startup
- **Option C**: Use Azure File Share mounted to `/documents`

### 3. Scaling Considerations

- **App Service Plan**: B1 or higher recommended (Basic tier or above)
- **Workers**: Currently set to 4 in `startup.txt`
- **Timeout**: Set to 600 seconds for long embeddings operations
- **Auto-scaling**: Configure under **Settings** > **Scale out**

### 4. CORS Configuration

Update CORS in `main.py` for production:

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://td-ai-assistant.azurewebsites.net"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### 5. Monitoring and Logging

Enable Application Insights:

```bash
az webapp log config \
  --name td-ai-assistant \
  --resource-group myResourceGroup \
  --application-logging filesystem \
  --level information
```

View logs:
```bash
az webapp log tail \
  --name td-ai-assistant \
  --resource-group myResourceGroup
```

### 6. Performance Optimization

#### Enable Always On
```bash
az webapp config set \
  --name td-ai-assistant \
  --resource-group myResourceGroup \
  --always-on true
```

#### Adjust Worker Count
For high traffic, increase workers in startup command:
```
gunicorn -w 8 -k uvicorn.workers.UvicornWorker main:app --bind 0.0.0.0:8000 --timeout 600
```

## Troubleshooting

### Issue: Application not starting

**Check logs**:
```bash
az webapp log tail --name td-ai-assistant --resource-group myResourceGroup
```

**Common causes**:
- Missing environment variables
- Incorrect startup command
- Package installation failures

### Issue: 502 Bad Gateway

**Causes**:
- Application startup timeout
- Insufficient memory
- Blocking operations during startup

**Solutions**:
- Increase timeout in startup command
- Use a higher tier App Service Plan
- Move document loading to background task

### Issue: Uploaded files not persisting

**Explanation**: Azure App Service uses ephemeral file systems

**Solution**: Implement Azure Blob Storage for file uploads:

```python
from azure.storage.blob import BlobServiceClient

# Store uploaded files in blob storage
blob_service_client = BlobServiceClient.from_connection_string(
    os.getenv("AZURE_STORAGE_CONNECTION_STRING")
)
```

### Issue: Slow vector store initialization

**Solution**: Cache embeddings or use a persistent vector database

## Cost Estimation

**Monthly costs** (approximate):
- App Service B1: ~$13/month
- Azure OpenAI: Pay per token usage
- Outbound data transfer: First 100GB free

**Recommendations**:
- Start with B1, monitor performance
- Set spending limits on Azure OpenAI
- Use Application Insights free tier

## Security Best Practices

1. **API Keys**: Always use environment variables, never hardcode
2. **CORS**: Restrict to your domain only
3. **HTTPS**: Azure provides free SSL certificates
4. **Authentication**: Consider adding Azure AD authentication
5. **Rate Limiting**: Implement rate limiting for API endpoints
6. **Input Validation**: Already implemented via Pydantic models

## Next Steps

1. Set up CI/CD pipeline with GitHub Actions
2. Implement proper vector store persistence
3. Add user authentication
4. Set up monitoring and alerts
5. Configure custom domain
6. Implement caching strategy

## Support Resources

- Azure App Service Documentation: https://docs.microsoft.com/en-us/azure/app-service/
- FastAPI Deployment: https://fastapi.tiangolo.com/deployment/
- LangChain on Azure: https://python.langchain.com/docs/integrations/platforms/microsoft

## GitHub Actions CI/CD (Optional)

Create `.github/workflows/azure-deploy.yml`:

```yaml
name: Deploy to Azure Web App

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2

    - name: Set up Python
      uses: actions/setup-python@v2
      with:
        python-version: '3.11'

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt

    - name: Deploy to Azure Web App
      uses: azure/webapps-deploy@v2
      with:
        app-name: 'td-ai-assistant'
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
```

Add your Azure publish profile to GitHub Secrets as `AZURE_WEBAPP_PUBLISH_PROFILE`.

---

**Deployment complete!** Your AI Assistant is now running on Azure Web App.
