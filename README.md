# Azure-Synapse-pipeline
# Provision an Azure Synapse Analytics Workspace and Implement a Data Flow

This document outlines the steps to provision an Azure Synapse Analytics workspace and implement a data flow to load data from Azure Data Lake Storage Gen2 into a dedicated SQL pool.

## Prerequisites

* An Azure subscription.
* Access to the Azure portal.

## Steps

### 1. Open Azure Cloud Shell

1.  Sign in to the [Azure portal](https://portal.azure.com).
2.  Click the **[\>_]** button located to the right of the search bar at the top of the page to open Azure Cloud Shell.
3.  If prompted, select **PowerShell** as the environment and create storage if necessary.
4.  If you have a Bash environment open, use the dropdown menu at the top left of the Cloud Shell pane to switch to **PowerShell**.

### 2. Clone the Repository

In the PowerShell pane of the Cloud Shell, execute the following commands to clone the MicrosoftLearning repository containing the lab files:

```powershell
rm -r dp-203 -f
git clone [https://github.com/MicrosoftLearning/dp-203-azure-data-engineer](https://github.com/MicrosoftLearning/dp-203-azure-data-engineer) dp-203
```
### 3.Run the Setup Script
```powershell
cd dp-203/Allfiles/labs/10
./setup.ps1
```
*If prompted, select the Azure subscription you wish to use.
*When asked, enter and remember a password for your Azure Synapse SQL pool.
*Wait for the script to complete. This process may take approximately 10 minutes or longer.
### 4.Verify Provisioned Resources
-Once the script finishes, navigate to the resource group named dp203-xxxxxxx (where xxxxxxx is a unique suffix) in the Azure portal.
-Select your Synapse workspace within this resource group
### 5.Open Synapse Studio
-On the Overview page of your Synapse Workspace, locate the Open Synapse Studio card.
-Click Open to launch Synapse Studio in a new browser tab. Sign in if prompted.
-Use the ›› icon on the left side to expand the menu
### 6.Start the Dedicated SQL Pool
-Go to the Manage page.
-Select the SQL pools tab.
-Find the row for the dedicated SQL pool named sqlxxxxxxx.
-Click the ▷ (Start) icon for this pool and confirm the resume action when prompted.
-Resuming the pool can take several minutes. Monitor the status using the ↻ Refresh button until it shows as Online.
### 7.View Source and Destination Data Stores
-Navigate to the Data page.
-On the Linked tab, verify the presence of a linked service to your Azure Data Lake Storage Gen2 account, named similar to synapsexxxxxxx (Primary - datalakexxxxxxx).
-Expand your storage account and confirm the existence of a file system container named files (primary).
-Select the files container and note the data folder within it.
-Open the data folder and observe the Product.csv file.
-Right-click Product.csv and select Preview to examine its contents. It contains a header row and product data.
-Return to the Manage page and ensure your dedicated SQL pool is Online.
-Go back to the Data page and select the Workspace tab.
-Expand SQL database, then your sqlxxxxxxx (SQL) database, and finally Tables.
-Select the dbo.DimProduct table.
-In the ... menu for dbo.DimProduct, select New SQL script > Select TOP 100 rows. This will execute a query showing the initial data in the table (likely a single row).
### 8. Create a Pipeline with a Data Flow Activity
-In Synapse Studio, go to the Integrate page.
-Click the + menu and select Pipeline to create a new pipeline.
-In the Properties pane for the new pipeline, change the Name to Load Product Data. Hide the Properties pane using the button above it.
-In the Activities pane, expand Move & transform and drag a Data flow activity onto the pipeline design surface.
-Under the pipeline design surface, in the General tab, set the Name property of the Data flow activity to LoadProducts.
-On the Settings tab of the Data flow activity, expand Staging and configure the following:
  -->Staging linked service: Select the synapsexxxxxxx-WorkspaceDefaultStorage linked service.
  -->Staging storage folder: Set container to files and Directory to stage_products.
### 9. Configure the Data Flow
-At the top of the Settings tab for the LoadProducts data flow activity, for the Data flow property, select + New.
-In the Properties pane for the new data flow, set the Name to LoadProductsData and hide the Properties pane.
### 10.Add Sources 
1.In the data flow design surface, click the Add Source dropdown and select Add Source. Configure the first source:
-Output stream name: ProductsText
-Description: Products text data
-Source type: Integration dataset
-Dataset: Click New and configure the dataset:
    -Type: Azure Datalake Storage Gen2
    -Format: Delimited text
    -Name: Products_Csv
    -Linked service: synapsexxxxxxx-WorkspaceDefaultStorage
    -File path: files/data/Product.csv
    -First row as header: Selected
    -Import schema: From connection/store
    -Allow schema drift: Selected
-On the Projection tab for the ProductsText source, set the following data types:
    -ProductID: string
    -ProductName: string
    -Color: string
    -Size: string
    -ListPrice: decimal
    -Discontinued: boolean
2.Add a second source by clicking the Add Source dropdown and selecting Add Source. Configure this source:
-Output stream name: ProductTable
-Description: Product table
-Source type: Integration dataset
-Dataset: Click New and configure the dataset:
    -Type: Azure Synapse Analytics
    -Name: DimProduct
    -Linked service: Click New and configure the linked service:
        Name: Data_Warehouse
        Description: Dedicated SQL pool
        Connect via integration runtime: AutoResolveIntegrationRuntime
        Version: Legacy
        Account selection method: From Azure subscription
        Azure subscription: Select your Azure subscription
        Server name: synapsexxxxxxx (Your Synapse workspace name)
        Database name: sqlxxxxxxx
        SQL pool: sqlxxxxxxx
        Authentication type: System Assigned Managed Identity
    -Table name: dbo.DimProduct
    -Import schema: From connection/store
    -Allow schema drift: Selected
-On the Projection tab for the ProductTable source, verify that the following data types are set:
  ProductKey: integer
  ProductAltKey: string
  ProductName: string
  Color: string
  Size: string
  ListPrice: decimal
  Discontinued: boolean
### 11. Add a Lookup
1.Select the + icon at the bottom right of the ProductsText source and choose Lookup.
2.Configure the Lookup settings:
  -Output stream name: MatchedProducts
  -Description: Matched product data
  -Primary stream: ProductText
  -Lookup stream: ProductTable
  -Match multiple rows: Unselected
  -Match on: Last row
  -Sort conditions: ProductKey ascending
  -Lookup conditions: ProductID == ProductAltKey
### 12. Add an Alter Row
1.Select the + icon at the bottom right of the MatchedProducts Lookup and choose Alter Row.
2.Configure the alter row settings:
  -Output stream name: SetLoadAction
  -Description: Insert new, upsert existing
  -Incoming stream: MatchedProducts
  -Alter row conditions: Edit the existing condition and add a second one:
    InsertIf: isNull(ProductKey)
    UpsertIf: not(isNull(ProductKey))
### 13. Add a Sink
1.Select the + icon at the bottom right of the SetLoadAction alter row step and choose Sink.
2.Configure the Sink properties:
  -Output stream name: DimProductTable
  -Description: Load DimProduct table
  -Incoming stream: SetLoadAction
  -Sink type: Integration dataset
  -Dataset: DimProduct (the Azure Synapse Analytics dataset created earlier)
  -Allow schema drift: Selected
3.On the Settings tab for the DimProductTable sink, specify:
  -Update method: Select Allow insert and Allow Upsert.
  -Key columns: Select List of columns, and then select the ProductAltKey column.
4.On the Mappings tab for the DimProductTable sink, clear the Auto mapping checkbox and specify the following column mappings:
  -ProductID: ProductAltKey
  -ProductsText@ProductName: ProductName
  -ProductsText@Color: Color
  -ProductsText@Size: Size
  -ProductsText@ListPrice: ListPrice
  -ProductsText@Discontinued: Discontinued
### 14. Debug the Data Flow
1.At the top of the data flow designer, enable Data flow debug. Review the default configuration and click OK. Wait for the debug cluster to start.
2.Select the DimProductTable sink and go to its Data preview tab.
3.Click the ↻ Refresh button to run data through the data flow for debugging.
4.Review the preview data, noting the indications for inserted (+) and upserted (*+) rows.
### 15. Publish and Run the Pipeline
1.Click the Publish all button to save and publish the pipeline and all associated assets.
2.Once publishing is complete, close the LoadProductsData data flow pane and return to the Load Product Data pipeline pane.
3.At the top of the pipeline designer, in the Add trigger menu, select Trigger now. Click OK to confirm the pipeline run.
### 16. Monitor the Pipeline Run
1.Navigate to the Monitor page.
2.View the Pipeline runs tab and check the status of the Load Product Data pipeline.
3.Use the ↻ Refresh button to update the status. The pipeline may take several minutes to complete.
### 17. Verify the Loaded Data
1.Once the pipeline run succeeds, go to the Data page.
2.In your SQL database, find the dbo.DimProduct table.
3.Open the ... menu for dbo.DimProduct and select New SQL script > Select TOP 100 rows.
4.Execute the query. The table should now contain the data loaded from the Product.csv file, with new products inserted and existing ones updated.

This completes the process of provisioning an Azure Synapse Analytics workspace and implementing a data flow to load data





