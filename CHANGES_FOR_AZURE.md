# Summary of Changes for Azure Web App Deployment

This document summarizes all changes made to prepare the application for Azure Web App deployment.

## Files Created

### 1. **requirements.txt**
   - **Location**: Root directory
   - **Purpose**: Lists all Python dependencies
   - **Key Addition**: Added `gunicorn==23.0.0` for production server
   - **Usage**: Azure automatically installs these packages during deployment

### 2. **startup.txt**
   - **Location**: Root directory
   - **Content**: `gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app --bind 0.0.0.0:8000 --timeout 600`
   - **Purpose**: Tells Azure how to start the application
   - **Configuration**: 4 workers, 600-second timeout for long operations

### 3. **.gitignore**
   - **Location**: Root directory
   - **Purpose**: Excludes unnecessary files from version control
   - **Includes**: Virtual environment, .env files, test files, logs

### 4. **AZURE_DEPLOYMENT_GUIDE.md**
   - **Location**: Root directory
   - **Purpose**: Comprehensive deployment guide
   - **Covers**: Step-by-step deployment, troubleshooting, best practices

### 5. **CHANGES_FOR_AZURE.md** (this file)
   - **Location**: Root directory
   - **Purpose**: Quick reference of all changes made

## Files Modified

### 1. **main.py**

#### Change 1: Updated Root Endpoint (Line 209-212)
**Before**:
```python
@app.get("/")
def read_root():
    return {"message": "Hello World"}
```

**After**:
```python
@app.get("/")
async def read_root():
    """Serve the UI at the root path"""
    return FileResponse("static/index.html")
```

**Reason**: Makes the UI accessible at the root URL (https://your-app.azurewebsites.net/)

#### Change 2: Added CORS Comment (Line 172-174)
**Added**:
```python
# For Azure deployment, update allow_origins with your Azure Web App URL
# Example: ["https://your-app-name.azurewebsites.net"]
```

**Reason**: Reminds developers to update CORS for production security

#### Change 3: Vector Store Initialization (Lines 136-179)
**Before**:
```python
def load_documents():
    """Load documents from the documents folder and create vector store"""
    # Required documents folder with .txt files
    # Failed if folder missing or empty
```

**After**:
```python
def initialize_vector_store():
    """Initialize an empty vector store for document uploads via UI"""
    # Creates empty Chroma vector store
    # Optionally loads documents from folder if it exists
    # No failure if folder missing - ready for UI uploads
```

**Reason**:
- Fixes Azure deployment issue where documents folder may not exist
- Allows application to start successfully without pre-loaded documents
- Users can upload all documents via UI
- More flexible for cloud deployment

#### Change 4: SQLite3 Version Workaround (Lines 1-11)
**Added**:
```python
# Workaround for Azure App Service SQLite3 version issue
try:
    __import__('pysqlite3')
    import sys
    sys.modules['sqlite3'] = sys.modules.pop('pysqlite3')
except ImportError:
    pass
```

**Reason**:
- Azure App Service has SQLite3 < 3.35.0
- Chroma requires SQLite3 >= 3.35.0
- This replaces built-in sqlite3 with pysqlite3-binary on Azure
- Falls back to built-in sqlite3 on Windows (where newer version exists)
- **Critical fix** - without this, application fails to start on Azure

**Package Added to requirements.txt**: `pysqlite3-binary==0.5.4.post2`

### 2. **static/index.html**

#### Change 1: Updated RAG Endpoint URL (Line 343)
**Before**:
```javascript
const response = await fetch('http://localhost:8000/ask-rag', {
```

**After**:
```javascript
const response = await fetch('/ask-rag', {
```

**Reason**: Uses relative URL that works on any domain (localhost or Azure)

#### Change 2: Updated Upload Endpoint URL (Line 466)
**Before**:
```javascript
const response = await fetch('http://localhost:8000/upload-document', {
```

**After**:
```javascript
const response = await fetch('/upload-document', {
```

**Reason**: Uses relative URL that works on any domain (localhost or Azure)

## No Changes Required For These Files

- **.env** - Environment variables are configured in Azure App Service settings
- **documents/** folder - Can be included in deployment or moved to Azure Blob Storage
- **static/** folder (except index.html) - No changes needed

## Environment Variables to Configure in Azure

These must be set in Azure App Service Configuration:

1. `AZURE_OPENAI_ENDPOINT`
2. `AZURE_OPENAI_API_KEY`
3. `AZURE_OPENAI_API_VERSION`
4. `AZURE_OPENAI_DEPLOYMENT`
5. `AZURE_OPENAI_EMBEDDING_DEPLOYMENT`

## Testing Checklist

### Local Testing (Before Deployment)
- [x] Application starts successfully
- [x] Root URL (/) serves the UI
- [x] File upload works (.txt and .docx)
- [x] RAG queries return correct results
- [x] All relative URLs work correctly

### Azure Testing (After Deployment)
- [ ] Application accessible at https://your-app-name.azurewebsites.net
- [ ] Environment variables configured correctly
- [ ] UI loads and displays properly
- [ ] Chat functionality works
- [ ] File upload works
- [ ] Documents are searchable
- [ ] No console errors in browser

## Important Notes

### 1. Vector Store Persistence
⚠️ **Current Implementation**: In-memory Chroma vector store
- Vector store initializes **empty** on every app restart
- Uploaded documents are stored in-memory only (lost on restart)
- Optional: Documents in `documents/` folder are loaded if folder exists
- **No dependency on pre-existing documents** - app starts successfully even with empty folder

**Production Recommendation**:
- Use persistent storage (Azure Cosmos DB, Azure AI Search, or Pinecone)
- Store uploaded files in Azure Blob Storage
- Implement auto-reload from Blob Storage on startup

### 2. File Storage
⚠️ **Current Implementation**: UI-driven document management
- Users upload documents exclusively through the UI
- All uploads are stored in-memory (not persisted)
- Optional: Pre-load documents from `documents/` folder if it exists
- **Azure-friendly**: No requirement for documents folder to exist

⚠️ **Azure App Service File System**: Ephemeral (temporary)
- Files uploaded via UI at runtime are lost on restart
- `/documents` folder in deployment is persistent (if included)

**Recommendation**: Implement Azure Blob Storage for persistent user uploads

### 3. Startup Time
- Initial startup: 10-30 seconds (no document loading required)
- Faster startup compared to loading many documents
- Vector store initializes empty immediately
- Documents loaded on-demand when uploaded via UI

### 4. Scaling
- Currently configured for 4 workers
- Can be adjusted in startup command
- Consider App Service plan based on traffic

## Quick Deployment Commands

```bash
# Login to Azure
az login

# Create resource group
az group create --name myResourceGroup --location eastus

# Create App Service plan
az appservice plan create --name myPlan --resource-group myResourceGroup --sku B1 --is-linux

# Create Web App
az webapp create --resource-group myResourceGroup --plan myPlan --name td-ai-assistant --runtime "PYTHON|3.11"

# Configure startup command
az webapp config set --resource-group myResourceGroup --name td-ai-assistant --startup-file "gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app --bind 0.0.0.0:8000 --timeout 600"

# Set environment variables (replace with your values)
az webapp config appsettings set --resource-group myResourceGroup --name td-ai-assistant --settings \
  AZURE_OPENAI_ENDPOINT="<your-endpoint>" \
  AZURE_OPENAI_API_KEY="<your-key>" \
  AZURE_OPENAI_API_VERSION="<version>" \
  AZURE_OPENAI_DEPLOYMENT="<deployment>" \
  AZURE_OPENAI_EMBEDDING_DEPLOYMENT="<embedding>"

# Deploy from local git
az webapp deployment source config-local-git --name td-ai-assistant --resource-group myResourceGroup

# Push to Azure
git remote add azure <deployment-url>
git push azure master
```

## Rollback Plan

If deployment fails:

1. Check logs: `az webapp log tail --name td-ai-assistant --resource-group myResourceGroup`
2. Verify environment variables in Azure Portal
3. Check startup command configuration
4. Test locally with same Python version
5. Redeploy previous working version from git

## Cost Monitoring

- Set up Azure Budget alerts
- Monitor Azure OpenAI token usage
- Track App Service metrics
- Estimated cost: ~$13-50/month depending on tier

## Next Steps After Deployment

1. ✅ Verify application is running
2. ✅ Test all functionality
3. ⚠️ Update CORS to restrict to your domain only
4. ⚠️ Implement persistent vector store
5. ⚠️ Set up Azure Blob Storage for uploads
6. ⚠️ Configure custom domain (optional)
7. ⚠️ Enable Application Insights for monitoring
8. ⚠️ Set up CI/CD pipeline (GitHub Actions)
9. ⚠️ Implement rate limiting
10. ⚠️ Add authentication if needed

---

**Ready for Deployment!** 🚀

All changes have been made to support Azure Web App deployment while maintaining local development compatibility.
