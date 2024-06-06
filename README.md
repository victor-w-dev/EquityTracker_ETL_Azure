## Project Overview
This project demonstrates a comprehensive ETL pipeline leveraging various Azure services for an equity tracking report system, including followings:
1. Azure Virtual Machine (VM)
2. Azure Data Factory (ADF)
3. Azure Data Lake Storage (ADLS) Gen2
4. Databricks
5. Azure Logic Apps

## Architecture
<img src="https://github.com/victor-w-dev/EquityTracker_ETL_Azure/blob/main/img/project_architecture.PNG" width="80%" height="80%"><br>

### Azure Virtual Machine (VM)
- Purpose: Hosts the data collection and processing application.
- Log Generation: Generates log files for daily equity tracking reports, saved in a folder on the VM. Log files are named by date (e.g., 20240601.log).
- Log Frequency: Four new log files are created daily at UTC 00:00, corresponding to different equity tracking strategies. <br>
### Azure Data Factory (ADF)
- Integration Runtime: Connects to the VM, ADLS, and Databricks.
- Pipeline: Automates the process of copying log files from the VM to ADLS.
- Step 1: Copy files from the VM to ADLS.
- Step 2: Databricks activity for incremental ingestion into a bronze layer.
- Incremental Ingestion: Uses COPY INTO to identify and ingest only new files, ensuring efficiency.
- Monitoring and Notification: Integrates with Azure Logic Apps to send email notifications upon successful or failed pipeline runs.
### Azure Data Lake Storage Gen2 (ADLS)
- Purpose: Stores the log files and acts as the primary data lake.
### Databricks
- Mounting ADLS: Uses a service principal to mount the ADLS folder, managed by DBFS.
- Data Processing: Processes and ingests log files into the bronze layer for further analysis.
### Azure Logic App
- Notification: Sends email notifications upon successful or failed ADF pipeline runs.
- Integration: Uses a web activity in ADF to trigger the Logic App.
