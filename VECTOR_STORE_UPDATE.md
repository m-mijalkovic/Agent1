# Vector Store Initialization Update

## Problem Solved

The application was failing to start in Azure Web App because:
1. The `documents/` folder did not exist or was empty
2. The `load_documents()` function required documents to initialize the vector store
3. Application returned "Vector store not initialized" error when queried

## Solution Implemented

### Changed Function: `load_documents()` → `initialize_vector_store()`

#### Old Behavior:
```python
def load_documents():
    # Required documents folder to exist
    # Required at least one .txt file in the folder
    # Failed silently if conditions not met
    # Vector store remained None if no documents found
```

#### New Behavior:
```python
def initialize_vector_store():
    # Always creates an empty Chroma vector store
    # No dependency on documents folder existing
    # Optionally loads documents from folder if it exists
    # Gracefully handles missing or empty documents folder
    # Vector store is always initialized (even if empty)
```

## Key Benefits

### 1. **Azure Deployment Ready**
- Application starts successfully without pre-loaded documents
- No dependency on documents folder structure
- Works in any cloud environment

### 2. **UI-Driven Document Management**
- Users upload all documents through the web interface
- Cleaner separation of concerns
- More intuitive user experience

### 3. **Flexible Configuration**
- **Option A**: Start with empty vector store (UI uploads only)
- **Option B**: Pre-load documents from `documents/` folder if it exists
- **Option C**: Both - pre-load and allow UI uploads

### 4. **Faster Startup**
- Empty vector store initializes in seconds
- No waiting for document processing on startup
- Documents added on-demand as users upload them

## What Changed in the Code

### File: `main.py`

#### Change 1: Function Renamed and Rewritten (Lines 136-179)

**Before:**
```python
def load_documents():
    """Load documents from the documents folder and create vector store"""
    global vector_store

    documents_path = "./documents"
    if not os.path.exists(documents_path):
        print(f"Documents folder not found at {documents_path}")
        return  # Vector store stays None!

    loader = DirectoryLoader(documents_path, glob="**/*.txt", loader_cls=TextLoader)
    documents = loader.load()

    if not documents:
        print("No documents found in the documents folder")
        return  # Vector store stays None!

    # ... create vector store only if documents exist
```

**After:**
```python
def initialize_vector_store():
    """Initialize an empty vector store for document uploads via UI"""
    global vector_store

    try:
        # ALWAYS create vector store (empty)
        vector_store = Chroma(
            collection_name="company_docs",
            embedding_function=embeddings
        )

        print("Vector store initialized successfully (empty - ready for uploads)")

        # OPTIONAL: Load documents from folder if it exists
        documents_path = "./documents"
        if os.path.exists(documents_path):
            try:
                loader = DirectoryLoader(documents_path, glob="**/*.txt", ...)
                documents = loader.load()

                if documents:
                    # Add to existing vector store
                    vector_store.add_documents(splits)
                    print(f"Loaded {len(documents)} documents from folder")
                else:
                    print("Documents folder empty - use UI to upload")
            except Exception as e:
                print(f"Could not load documents (this is okay): {str(e)}")
                print("Use UI to upload documents")
        else:
            print("Documents folder not found - use UI to upload documents")

    except Exception as e:
        print(f"Error initializing vector store: {str(e)}")
        raise  # Only fail if vector store creation fails
```

#### Change 2: Startup Event Updated (Lines 202-208)

**Before:**
```python
@app.on_event("startup")
async def startup_event():
    """Load documents into vector store on application startup"""
    print("Loading documents into vector store...")
    load_documents()
    print("Documents loaded successfully!")
```

**After:**
```python
@app.on_event("startup")
async def startup_event():
    """Initialize vector store on application startup"""
    print("Initializing vector store...")
    initialize_vector_store()
    print("Vector store ready!")
```

## Testing Results

### Test 1: Empty Documents Folder
```bash
# Scenario: No documents in folder
$ curl http://localhost:8000/ask-rag -d '{"prompt": "Hello"}'

# Result: SUCCESS
{
  "response": "The context does not contain enough information...",
  "context_used": [],
  "num_documents_retrieved": 0
}
```
✅ Application works with empty vector store

### Test 2: Upload Document via UI
```bash
# Upload a .txt file
$ curl -X POST -F "file=@test.txt" http://localhost:8000/upload-document

# Result: SUCCESS
{
  "message": "Successfully uploaded and processed test.txt",
  "chunks_created": 1,
  "status": "success"
}
```
✅ File upload adds to empty vector store

### Test 3: Query Uploaded Document
```bash
# Query after upload
$ curl http://localhost:8000/ask-rag -d '{"prompt": "What was uploaded?"}'

# Result: SUCCESS
{
  "response": "A test document was uploaded...",
  "context_used": ["Test Document for Upload..."],
  "num_documents_retrieved": 1
}
```
✅ RAG retrieval works after upload

### Test 4: Upload .docx File
```bash
# Upload a Word document
$ curl -X POST -F "file=@test.docx" http://localhost:8000/upload-document

# Result: SUCCESS
{
  "message": "Successfully uploaded and processed test.docx",
  "chunks_created": 1,
  "file_type": "word",
  "status": "success"
}
```
✅ Both .txt and .docx uploads work

## User Experience

### Before This Change:
1. User deploys to Azure
2. Application fails to start (no documents folder)
3. User sees errors: "Vector store not initialized"
4. User must troubleshoot and add documents folder
5. Application must be redeployed

### After This Change:
1. User deploys to Azure
2. Application starts successfully (empty vector store)
3. User opens the web UI
4. User uploads documents through the UI
5. User can immediately query uploaded documents

## Migration Guide

### For Existing Users:

**No action required!** The change is backward compatible:
- If you have a `documents/` folder with .txt files, they will still be loaded automatically
- Your existing setup continues to work as before
- You now have the additional option to upload documents via UI

### For New Users:

You can now deploy the application without any documents:
1. Deploy the application to Azure
2. Application starts with empty vector store
3. Upload documents through the web interface
4. Start asking questions immediately

## Production Recommendations

For a production Azure deployment, consider:

1. **Azure Blob Storage**: Store uploaded documents persistently
   ```python
   from azure.storage.blob import BlobServiceClient

   # Store uploads in Blob Storage
   blob_client = blob_service.get_blob_client(container="documents", blob=filename)
   blob_client.upload_blob(file_content)
   ```

2. **Azure Cosmos DB**: Use for persistent vector storage
   - Store document metadata
   - Track upload history
   - Enable document versioning

3. **Startup Document Loader**: Reload documents from Blob Storage on startup
   ```python
   def initialize_vector_store():
       # Initialize empty vector store
       vector_store = Chroma(...)

       # Load documents from Azure Blob Storage
       documents = load_from_blob_storage()
       if documents:
           vector_store.add_documents(documents)
   ```

## Summary

✅ **Fixed**: Application no longer fails on startup without documents
✅ **Improved**: UI-driven document management
✅ **Azure-Ready**: No dependency on pre-existing files
✅ **Flexible**: Optional document folder support
✅ **Tested**: All functionality working correctly

The application is now truly cloud-native and ready for Azure deployment!
