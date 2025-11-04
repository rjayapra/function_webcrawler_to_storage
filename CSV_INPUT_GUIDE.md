# CSV Input Guide for Azure Function Web Crawler

This guide explains how to provide a CSV file containing URLs and custom filenames to your Azure Function App.

## CSV File Format

Your CSV file should have the following structure:

```csv
Url,Filename,Fr_Filename
https://example.com/page1,custom-name-1.html,french-name-1.html
https://example.com/page2,custom-name-2.html,french-name-2.html
```

- **Column 1 (Url)**: The URL to crawl
- **Column 2 (Filename)**: The custom filename to use for storage (without path)
- **Column 3 (Fr_Filename)**: Optional third column (currently not used)

## Methods to Input CSV File

### 1. Azure Blob Storage (Recommended for Production)

Upload your CSV file to Azure Blob Storage and configure the Function App to read from there.

#### Steps:

1. **Create a container for configuration files** (if not exists):
   ```bash
   az storage container create --name config --account-name <your-storage-account>
   ```

2. **Upload your CSV file**:
   ```bash
   az storage blob upload \
     --account-name <your-storage-account> \
     --container-name config \
     --name input.csv \
     --file ./input.csv
   ```

3. **Set environment variables in your Function App**:
   ```bash
   az functionapp config appsettings set \
     --name <your-function-app-name> \
     --resource-group <your-resource-group> \
     --settings "CSV_CONTAINER_NAME=config" "CSV_FILE_PATH=input.csv"
   ```

#### Environment Variables for Blob Storage Method:
- `CSV_CONTAINER_NAME`: Container name where CSV file is stored (default: "config")
- `CSV_FILE_PATH`: Path/name of the CSV file in the container (default: "input.csv")

### 2. Include in Deployment Package (Simple for Development)

For development or small deployments, include the CSV file in your function deployment.

#### Steps:

1. **Place your CSV file** in the function app root directory:
   ```
   azurefunction_webcrawler_to_blobstorage/
   ├── input.csv          # <-- Your CSV file here
   ├── function_app.py
   ├── crawler.py
   └── ...
   ```

2. **Deploy the function** (the CSV will be included automatically):
   ```bash
   func azure functionapp publish <your-function-app-name>
   ```

3. **Set environment variable** (optional, if using different filename):
   ```bash
   az functionapp config appsettings set \
     --name <your-function-app-name> \
     --resource-group <your-resource-group> \
     --settings "CSV_FILE_PATH=input.csv"
   ```

### 3. Azure File Share

Mount an Azure File Share to your Function App for shared file access.

#### Steps:

1. **Create a file share**:
   ```bash
   az storage share create --name csvfiles --account-name <your-storage-account>
   ```

2. **Upload CSV file to file share**:
   ```bash
   az storage file upload \
     --account-name <your-storage-account> \
     --share-name csvfiles \
     --source ./input.csv
   ```

3. **Mount file share to Function App**:
   ```bash
   az webapp config storage-account add \
     --name <your-function-app-name> \
     --resource-group <your-resource-group> \
     --custom-id csvmount \
     --storage-type AzureFiles \
     --share-name csvfiles \
     --account-name <your-storage-account> \
     --mount-path /csvfiles \
     --access-key <your-storage-key>
   ```

4. **Set environment variable**:
   ```bash
   az functionapp config appsettings set \
     --name <your-function-app-name> \
     --resource-group <your-resource-group> \
     --settings "CSV_FILE_PATH=/csvfiles/input.csv"
   ```

## Environment Variables Reference

| Variable | Description | Default | Example |
|----------|-------------|---------|---------|
| `CSV_CONTAINER_NAME` | Blob container name for CSV file | "config" | "config" |
| `CSV_FILE_PATH` | Path to CSV file | "input.csv" | "input.csv" or "/csvfiles/urls.csv" |

## How the Function Reads CSV

The function follows this priority order:

1. **Try Azure Blob Storage**: If `CSV_CONTAINER_NAME` is set, attempts to read from blob storage
2. **Fallback to Local File**: If blob storage fails, reads from local file system
3. **Error Handling**: Logs errors and continues with empty URL list if both fail

## Updating CSV File

### For Blob Storage Method:
```bash
# Update the CSV file in blob storage
az storage blob upload \
  --account-name <your-storage-account> \
  --container-name config \
  --name input.csv \
  --file ./updated-input.csv \
  --overwrite
```

### For Deployment Package Method:
Redeploy the function app with the updated CSV file.

## Monitoring

Check the Function App logs to verify CSV loading:

```bash
# View recent logs
az functionapp log tail --name <your-function-app-name> --resource-group <your-resource-group>
```

Look for log messages like:
- `"Successfully loaded CSV from blob storage with X rows"`
- `"Loaded X URLs from CSV with custom filenames"`

## Troubleshooting

### Common Issues:

1. **BOM (Byte Order Mark) in CSV**: The function handles UTF-8 BOM automatically
2. **Missing blob container**: Ensure the container exists and the Function App has access
3. **Permissions**: Verify the Function App's managed identity has "Storage Blob Data Reader" role
4. **CSV Format**: Ensure first column is URL, second column is filename

### Verify CSV format:
```bash
# Check first few lines of your CSV
head -5 input.csv
```

### Test blob access:
```bash
# Verify the Function App can access the storage account
az storage blob list --container-name config --account-name <your-storage-account>
```