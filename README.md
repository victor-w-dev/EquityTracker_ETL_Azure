## Project Overview
This project demonstrates an equity tracking system leveraging Azure Data Factory ETL pipeline to load trading activity log data incrementally from Azure VM through Azure Data Lake Storage (ADLS) Gen2 to DataBricks Lakehouse Platform, using Azure Logic Apps for email notification.

## Architecture
What we’ll cover:
- Step 1: Create an ADF Pipeline to use Copy Activity to copy log files from the VM to ADLS container enabled with Hierarchical Namespace.
- Step 2: Run a Databricks Notebook with an activity set in the ADF Pipeline, transform extracted log file data.
- Step 3: Through the Multi-hop architecture approach, Databricks finalized a delta table for simple visulization for equity tracking.
- Step 4: Set a Web activity leveraging Azure Logic Apps to send email, noticing the Pipeline run successfully or fail
<img src="https://github.com/victor-w-dev/EquityTracker_ETL_Azure/blob/main/img/project_architecture.PNG" width="90%" height="90%"><br>
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
  - reference: [Create a self-hosted integration runtime - Azure Data Factory & Azure Synapse | Microsoft Learn](https://learn.microsoft.com/en-us/azure/data-factory/create-self-hosted-integration-runtime?tabs=data-factory)
### Azure Data Factory (ADF)
- Integration Runtime: Connects to the VM, ADLS, and Databricks.
- Pipeline: Automates the process of copying log files from the VM to ADLS.
- Step 1: Copy files from the VM to ADLS.
- Step 2: Databricks activity for incremental ingestion into a bronze layer.
- Monitoring and Notification: Integrates with Azure Logic Apps to send email notifications upon successful or failed pipeline runs.
### Azure Data Lake Storage Gen2 (ADLS)
- Purpose: Stores the log files and acts as the primary data lake.
### Databricks
- Mounting ADLS: Uses a service principal to mount the ADLS folder, managed by DBFS.
- Data Processing: Processes and ingests log files into the bronze layer for further analysis.
### Azure Logic App
- Notification: Sends email notifications upon successful or failed ADF pipeline runs.
- Integration: Uses a web activity in ADF to trigger the Logic App.
