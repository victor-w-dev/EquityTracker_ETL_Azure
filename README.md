## Project Overview
This project demonstrates an equity tracking system leveraging Azure Data Factory ETL pipeline to load trading activity log data incrementally from Azure VM through Azure Data Lake Storage (ADLS) Gen2 to DataBricks Lakehouse Platform, using Azure Logic Apps for email notification.

## Architecture
What we’ll cover:
- Step 1: Create an ADF Pipeline to use Copy Activity to copy log files from the VM to ADLS container enabled with Hierarchical Namespace.
- Step 2: Run a Databricks Notebook with an activity set in the ADF Pipeline, transform extracted log file data.
- Step 3: Through the Multi-hop architecture approach, Databricks finalized a delta table for simple visulization for equity tracking.
- Step 4: Set a Web activity leveraging Azure Logic Apps to send email, noticing the Pipeline run successfully or fail
<img src="https://github.com/victor-w-dev/EquityTracker_ETL_Azure/blob/main/img/project_architecture.PNG" width="90%" height="90%"><br>
- ADF Pipeline Setup:
<img src="https://github.com/victor-w-dev/EquityTracker_ETL_Azure/blob/main/img/adf_pipeline_structure.PNG" width="90%" height="90%"><br>

## Prerequisites
<!--ts-->
[1. Azure Virtual Machine (VM)](#Azure-VM)<br>
[2. Azure Data Factory (ADF)](#Azure-Data-Factory-ADF)<br>
[3. Azure Data Lake Storage (ADLS) Gen2](#Azure-Data-Lake-Storage-Gen2-ADLS)<br>
[4. Databricks](#Databricks)<br>
[5. Azure Logic Apps](#Azure-Logic-App)<br>
<!--te-->

### Azure Virtual Machine (VM)
- Purpose: Hosts the data collection and processing application.
- Log Generation: Generates log files for daily equity tracking reports, saved in a folder on the VM. Log files are named by date (e.g., 20240601.log).
- Log Frequency: Four new log files are created daily at UTC 00:00, corresponding to different equity tracking strategies. <br>
- Integration Runtime Setup for connection with ADF:
  - Step 1: Create a Self-hosted Integration Runtime in Azure Data Factory.
  - Step 2: Download and install the Integration Runtime on the Azure VM.
  - Step 3: Configure the Integration Runtime to connect to Azure Data Factory using the provided key.
  - Step 4: Verify the setup by testing the connection between the Integration Runtime and Azure Data Factory.
  - reference: 
    - [Create a self-hosted integration runtime - Azure Data Factory & Azure Synapse | Microsoft Learn](https://learn.microsoft.com/en-us/azure/data-factory/create-self-hosted-integration-runtime?tabs=data-factory)
### Azure Data Factory (ADF)
- Dataset and Linked Services Setup:
  - Datasets: Define the structure of data to be used in the pipeline.
    - VM Dataset: Represents log files stored on the VM.
    - ADLS Dataset: Represents log files stored in ADLS.<br>
    <img src="https://github.com/victor-w-dev/EquityTracker_ETL_Azure/blob/main/img/adf_dataset.PNG" width="30%" height="30%"><br>
  - Linked Services: Configure connections to the VM, ADLS, and Databricks.
    - VM Linked Service: Connects ADF to the Azure VM.
    - ADLS Linked Service: Connects ADF to Azure Data Lake Storage.
    - Databricks Linked Service: Connects ADF to Databricks.<br>
    <img src="https://github.com/victor-w-dev/EquityTracker_ETL_Azure/blob/main/img/adf_linked_services.PNG" width="50%" height="50%"><br>
- Integration Runtime: Connects to the VM, ADLS, and Databricks.<br>
  <img src="https://github.com/victor-w-dev/EquityTracker_ETL_Azure/blob/main/img/adf_IR.PNG" width="50%" height="50%"><br>
- [Pipeline Setup](#ADF-Pipeline-Setup)<br>: Automates the process of copying and transforming log files.
  - Copy Activity: Copies files from the VM to ADLS.
  - Databricks Activity: Processes data for incremental ingestion into a notebook.
  - Web Activity: Integrates with Azure Logic Apps to send email notifications upon successful or failed pipeline runs.
  
### Azure Data Lake Storage Gen2 (ADLS)
- Purpose: Stores the log files and acts as the primary data lake.
### Databricks
- Mounting ADLS: Uses a service principal to mount the ADLS folder, managed by DBFS.
- Data Processing: Processes and ingests log files into the bronze layer for further analysis.
- Multi-Hop Architecture: 
  - Bronze Layer: Raw data ingestion from ADLS.
    - Purpose: Capture raw log files and store them in a Delta table for incremental ingestion.
    - Process: Incremental ingestion using COPY INTO, a command that loads data from ADLS into the Delta table. The command utilizes metadata to track ingested files, ensuring that only new files are ingested.
    ```SQL
    %sql
    CREATE TABLE IF NOT EXISTS logging_raw(
        logging STRING,
        account STRING, 
        date_end date
    ) USING DELTA;
    ```
    ```SQL
    %sql
    COPY INTO logging_raw
    FROM (
      SELECT value as logging, 
            regexp_extract(_metadata.file_path, r"/skinnydew/([^/]+)/", 1) as account,
            to_date(regexp_extract(_metadata.file_path, r"/(\d{8})\.log$", 1), 'yyyyMMdd') as date_end
      FROM 'dbfs:/mnt/cryptotrader/trading_logging/MainAccount'
    )
    FILEFORMAT = TEXT
    PATTERN = '*/*/*/*.log'
    COPY_OPTIONS ('mergeSchema' = 'true');
    ```
  - Silver Layer: Cleansed and enriched data.
    - Purpose: Transform raw data into a structured format, applying business logic and data quality checks.
    - Process: Deduplication, filtering, and other transformations of the raw data.
    ```SQL
    %sql
    CREATE OR REPLACE TABLE account_records
    USING delta AS
    WITH extracted_logging AS (
        SELECT 
            account,
            date_end,
            to_timestamp(regexp_extract(logging, r"^([^,]+)", 1), "yyyy-MM-dd HH:mm:ss") as update_time,
            regexp_extract(logging, r'INFO - USDT amount: ([0-9]+\.[0-9]+)', 1) AS usdt_amount
        FROM logging_raw
        WHERE logging LIKE '%INFO - USDT amount:%'
    ),
    ranked_logging AS (
        SELECT 
            account,
            date_end,
            update_time,
            usdt_amount,
            ROW_NUMBER() OVER (PARTITION BY account, date_end ORDER BY update_time) AS row_num
        FROM extracted_logging
    )
    SELECT 
        date_end,
        account,
        update_time,
        usdt_amount,
        row_num
    FROM ranked_logging
    WHERE row_num = 1
    ORDER BY date_end, account, update_time
    ```
  - Gold Layer: Aggregated and ready-for-analysis data.
    - Purpose: Finalize data for consumption by analytics and reporting tools.
    - Process: Aggregation and creation of curated Delta tables for specific reporting needs.
    ```python
    # Read the Delta table into a DataFrame
    account_records_df = spark.read.table("account_records")
    ```
    ```python
    from pyspark.sql.functions import sum as spark_sum, col, round
    from pyspark.sql.types import FloatType
    ```
    ```python
    # Pivot the data
    pivot_df = account_records_df.groupBy('date_end').pivot('account').agg(spark_sum('usdt_amount')).fillna(0).orderBy('date_end')
    
    # Calculate the total daily equity, summing up all USDT amounts
    total_equity_df = pivot_df.withColumn('USDT_Sum', sum(pivot_df[col] for col in pivot_df.columns if col != 'date_end'))
    total_equity_df = total_equity_df.withColumn("ML0001", round(col("ML0001").cast(FloatType()), 3))\
                                 .withColumn("ML0002", round(col("ML0002").cast(FloatType()), 3))\
                                 .withColumn("ML0003", round(col("ML0003").cast(FloatType()), 3))\
                                 .withColumn("ML0004", round(col("ML0004").cast(FloatType()), 3))\
                                 .withColumn("USDT_Sum", round(col("USDT_Sum").cast(FloatType()), 3))

    total_equity_df.write.mode("overwrite").saveAsTable("equity_time_series")
    ```  
  - Visualization
    - Equity Curve Time Series: Utilize Databricks' built-in visualization tools to plot equity curves from the finalized Delta table in the Gold Layer.
    ```SQL
    %sql
    SELECT *
    FROM equity_time_series
    WHERE date_end >= '2024-05-09'
    ```
    <img src="https://github.com/victor-w-dev/EquityTracker_ETL_Azure/blob/main/img/plot.PNG" width="50%" height="50%"><br>
### Azure Logic App
- Notification: Sends email notifications upon successful or failed ADF pipeline runs.
- Integration: Uses a web activity in ADF to trigger the Logic App.
- reference:
  - [Copy data and send email notifications on success and failure | Microsoft Learn](https://learn.microsoft.com/en-us/azure/data-factory/tutorial-control-flow-portal)
  - [Send an email with an Azure Data Factory or Azure Synapse pipeline | Microsoft Learn](https://learn.microsoft.com/en-us/azure/data-factory/how-to-send-email)
