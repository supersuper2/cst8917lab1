
# Azure Functions in Python

Demo Link Video: https://youtu.be/Fzexb-M3ulw

---

## 1. Create Azure Function

### Steps

1. Open VS Code → `F1` → `Azure Functions: Create New Project...`
2. Choose:
   - Language: Python 
   - Trigger: HTTP trigger
   - Function Name: `HttpExample`
   - Authorization Level: Anonymous
3. Select Python interpreter.
4. Edit `local.settings.json`:
   ```json
   "AzureWebJobsStorage": "UseDevelopmentStorage=true",
   ```
   
### Run Locally

- Press `F5`
- Trigger via:
  - Azure panel → Local Project > Functions > HttpExample → Execute Function Now
  - Input JSON:
    ```json
    {
      "name": "Azure"
    }
    ```

---

## 2. Output Binding: Azure Storage Queue

### Setup

1. Run `Azure Functions: Download Remote Settings...`
2. Overwrite local settings and copy `AzureWebJobsStorage`.

Ensure `host.json` includes:

```json
{
  "version": "2.0",
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[3.*, 4.0.0)"
  }
}
```

### Update Function Code

Edit `function_app.py`:

```python
import azure.functions as func
import logging

app = func.FunctionApp(http_auth_level=func.AuthLevel.ANONYMOUS)

@app.route(route="HttpExample")
@app.queue_output(arg_name="msg", queue_name="outqueue", connection="AzureWebJobsStorage")
def HttpExample(req: func.HttpRequest, msg: func.Out [func.QueueMessage]) -> func.HttpResponse:
    logging.info('Python HTTP trigger function processed a request.')

    name = req.params.get('name')
    if not name:
        try:
            req_body = req.get_json()
        except ValueError:
            pass
        else:
            name = req_body.get('name')

    if name:
        msg.set(name)
        return func.HttpResponse(f"Hello, {name}. This HTTP triggered function executed successfully.")
    else:
        return func.HttpResponse(
             "This HTTP triggered function executed successfully. Pass a name in the query string or in the request body for a personalized response.",
             status_code=200
        )
```

### Test & Verify

- Press `F5`, execute function.
- Use Storage Explorer to view `outqueue`.

### Deploy

- `F1 → Azure Functions: Deploy to function app...`
- Choose your app and redeploy.

---

## 3. Output Binding: Azure SQL Database

### Setup SQL DB

1. [Follow the Microsoft link to create SQL Database](https://learn.microsoft.com/en-us/azure/azure-sql/database/single-database-create-quickstart)
2. Use SQL auth and allow Azure services access.
3. Create table:

```sql
CREATE TABLE dbo.ToDo (
    [Id] UNIQUEIDENTIFIER PRIMARY KEY,
    [order] INT NULL,
    [title] NVARCHAR(200) NOT NULL,
    [url] NVARCHAR(200) NOT NULL,
    [completed] BIT NOT NULL
);
```

4. Copy **ADO.NET** connection string (SQL auth).
5. Add app setting in VS Code:
   - `SqlConnectionString` → use the full connection string.

6. Download remote settings again.

### Ensure `host.json` has:

```json
{
  "version": "2.0",
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.*, 5.0.0)"
  },
  "concurrency": {
      "dynamicConcurrencyEnabled": true,
      "snapshotPersistenceEnabled": true
    }
}
```

### Update Function Code

Edit `function_app.py`:

```python
import azure.functions as func
import logging
from azure.functions.decorators.core import DataType
import uuid

app = func.FunctionApp()

@app.function_name(name="HttpTrigger1")
@app.route(route="hello", auth_level=func.AuthLevel.ANONYMOUS)
@app.generic_output_binding(arg_name="toDoItems", type="sql", CommandText="dbo.ToDo", ConnectionStringSetting="SqlConnectionString",data_type=DataType.STRING)
def test_function(req: func.HttpRequest, toDoItems: func.Out[func.SqlRow]) -> func.HttpResponse:
     logging.info('Python HTTP trigger function processed a request.')
     name = req.get_json().get('name')
     if not name:
        try:
            req_body = req.get_json()
        except ValueError:
            pass
        else:
            name = req_body.get('name')

     if name:
        toDoItems.set(func.SqlRow({"Id": str(uuid.uuid4()), "title": name, "completed": False, "url": ""}))
        return func.HttpResponse(f"Hello {name}!")
     else:
        return func.HttpResponse(
                    "Please pass a name on the query string or in the request body",
                    status_code=400
                )
```

### Test & Verify

- Press `F5`, run function locally.
- Use portal SQL Query Editor → `SELECT * FROM dbo.ToDo`

### Deploy

- `F1 → Azure Functions: Deploy to function app...`
- Trigger function again and verify DB writes.

---

## Clean Up

To remove all resources:

1. `F1 → Azure: Open in Portal`
2. Navigate to the resource group
3. Click **Delete resource group**
