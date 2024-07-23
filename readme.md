# Task Automation Orchestrator Agent (TAO Agent)

The Task Automation Orchestrator Agent (TAO Agent) is a configurable Python-based system designed to automate complex, multi-step workflows. While it's built to be adaptable to various tasks, this specific implementation focuses on automating the Statute of Limitations (SoL) data import and processing workflow.

### Core concepts:

- Provides a flexible framework for defining and executing sequences of tasks
- Uses YAML configuration files to specify workflow steps and parameters
- Integrates with OpenAI API for decision-making support during task execution
- using OpenAI Assistants API and function calling for custom python Tools

### Current implementation:

This specific version of TAO Agent is configured to handle the SoL (Statute of Limitations) data processing workflow, which includes:
- Retrieving new input files from specified UNC path locations
- Consolidating data from multiple documents into one consistent, structured import file
- Calculating SoL dates based on state-specific rules
- Generating specific layout output files for import into different systems
- Running external import programs to import SoL data into databases
- Archiving final data to specified network locations

The system manages error handling, logging, and provides completion notifications throughout the process. While currently set up for SoL processing, the underlying architecture can be easily adapted to orchestrate other complex, multi-step tasks by modifying the configuration and adding task-specific modules. 

## Architectural Diagram

https://github.com/jimmc414/TAO/blob/master/architectural_diagram.md

## Data Flow Diagram

https://github.com/jimmc414/TAO/blob/master/data_flow_diagram.md

## Sequence Diagram

https://github.com/jimmc414/TAO/blob/master/sequence_diagram.md

## Call Graph

[https://github.com/jimmc414/TAO/blob/master/sequence_diagram.md](https://github.com/jimmc414/TAO/blob/master/call_graph.md)

# TAO Agent Implementation Guide

## 1. Environment Setup

1.1. Ensure Python 3.8+ is installed on your system.
1.2. Set up a virtual environment:

```
python -m venv venv
source venv/bin/activate  # On Windows use `venv\Scripts\activate`
```

1.3. Install required packages:

```
pip install openai PyYAML pandas python-dateutil numpy openpyxl SQLAlchemy requests tqdm python-dotenv aiofiles asyncio typing-extensions
```

 or
 
 ```
 pip install -r requirements.txt
 ```
		 
1.4. Set your OpenAI API key as an environment variable:

```
export OPENAI_API_KEY=your_api_key_here
```

or for Windows

```
setx OPENAI_API_KEY=your_api_key_here
```


## 2. Project Structure

Create the following directory structure:
```
tao-agent/
├── main.py
├── config/
│   └── sample_config.yaml
├── tools/
│   ├── __init__.py
│   ├── file_operations.py
│   ├── data_processing.py
│   └── external_execution.py
├── logs/
├── data/
│   ├── input/
│   ├── output/
│   └── temp/
└── tests/
```

## 3. Configuration

3.1. Create `config/sample_config.yaml` with the following structure:

```
assistant:
  name: "Statute of Limitations Import Assistant"
  instructions: |
    You are an AI assistant specializing in the Statute of Limitations (SoL) import process. Your role is to guide and execute the workflow for importing and processing SoL data efficiently and accurately. Follow these guidelines:

    1. Process Overview:
       - Understand that the SoL import process involves multiple steps, from data retrieval to final import.
       - Maintain awareness of the overall workflow and how each step contributes to the final goal.

    2. Data Handling:
       - Treat all data with utmost care and confidentiality.
       - Ensure data integrity throughout the process, especially during file operations and calculations.

    3. Error Handling:
       - Anticipate potential issues at each step and provide guidance on error resolution.
       - If an error occurs, analyze its impact on the overall process and suggest appropriate actions.

    4. Decision Making:
       - When faced with choices, consider the implications on data accuracy, processing time, and system resources.
       - Justify your decisions based on the specific context of the SoL import task.

    5. Optimization:
       - Continuously look for opportunities to improve the process efficiency.
       - Suggest optimizations when you identify potential bottlenecks or redundancies.

    6. Compliance:
       - Ensure all operations comply with relevant laws and regulations regarding SoL and data handling.
       - Flag any potential compliance issues you notice during the process.

    7. Documentation:
       - Keep clear records of actions taken, decisions made, and any anomalies encountered.
       - Prepare concise yet comprehensive summaries at key points in the process.

    8. User Interaction:
       - Provide clear, step-by-step guidance when user input is required.
       - Explain complex concepts or calculations in simple terms when necessary.

    9. Tool Usage:
       - Utilize the provided tools effectively, understanding the purpose and limitations of each.
       - Combine tools creatively when needed to solve complex problems.

    Remember, your primary goal is to ensure a smooth, accurate, and efficient SoL import process while maintaining data integrity and compliance.

  model: "gpt-4-0125-preview"

initial_message: |
  Welcome to the Statute of Limitations Import Process. I'm here to guide you through each step of this complex task. We'll begin by determining the scope of our processing, then move through data retrieval, consolidation, SoL calculation, and finally, data import and record updating. If you have any questions or concerns at any point, please don't hesitate to ask. Let's start by determining our processing scope. Shall we proceed?

run_timeout: 3600  # 1 hour in seconds
poll_interval: 5  # 5 seconds

determine_processing_scope:
  db_path: "./data/processing_history.db"
  description: |
    This step involves querying the processing history database to determine the date range for the current import. Consider the following:
    - Check the most recent processing date in the database.
    - Propose a date range from the day after the last processed date to the current date.
    - If no previous processing date exists, prompt the user for a start date.
    - Confirm the proposed date range with the user before proceeding.

clean_workspace:
  archive_directory: "./data/archive"
  file_types: ["csv", "txt", "xlsx", "log", "old", "docx", "sql", "err", "aud"]
  description: |
    Prepare the workspace for new data processing:
    - Move all files of specified types to a timestamped archive folder.
    - Ensure the workspace is clean to prevent mixing of old and new data.
    - Confirm successful archiving of all relevant files before proceeding.

retrieve_new_input_files:
  source_directory: "T:/EDISHARE/NEW CLAIMS/2024"
  destination_directory: "./data/temp"
  description: |
    Retrieve new input files for processing:
    - Scan the source directory for files matching the NCR*.xlsx pattern.
    - Copy only files within the determined date range.
    - Verify file integrity after copying.
    - Report the number and names of files retrieved.

consolidate_input_files:
  input_directory: "./data/temp"
  output_file: "./data/temp/NCR_combined_output.xlsx"
  file_pattern: "NCR*.xlsx"
  description: |
    Combine all retrieved input files into a single consolidated file:
    - Read all Excel files matching the specified pattern.
    - Ensure consistent column structure across all files.
    - Handle any data discrepancies or formatting issues.
    - Produce a single, clean, consolidated Excel file for further processing.

calculate_statute_of_limitations:
  input_file: "./data/temp/NCR_combined_output.xlsx"
  output_file: "./data/temp/output_data.csv"
  state_laws_file: "./data/state_sol_laws.json"
  description: |
    Calculate Statute of Limitations dates for each record:
    - Load state-specific SoL laws from the JSON file.
    - For each record, determine the applicable state law.
    - Calculate the SoL date based on the contract date or charge-off date.
    - Handle edge cases such as leap years or invalid dates.
    - Output the results to a CSV file for further processing.

generate_input_files:
  sol_data_file: "./data/temp/output_data.csv"
  utimphis_output: "./data/output/utimphis.csv"
  imdiary_output: "./data/output/imdiary.csv"
  lcimp002_output: "./data/output/lcimp002.csv"
  description: |
    Generate system-specific input files from the calculated SoL data:
    - Create utimphis.csv for the main system update.
    - Generate imdiary.csv for diary entries related to SoL.
    - Produce lcimp002.csv for additional system updates.
    - Ensure all generated files adhere to the required format for each system.

process_input_files:
  acuthin_path: "\\\\THINCLIENT\\thinclient\\acuthin.exe"
  log_file: "./logs/import_log.txt"
  description: |
    Execute the acuthin.exe program to process the generated input files:
    - Ensure network access to the acuthin.exe location.
    - Run acuthin.exe with appropriate parameters for each input file.
    - Monitor the execution and capture all output in the log file.
    - Analyze the log file for any errors or warnings post-execution.

copy_to_lcs_data:
  source_directory: "./data/output"
  destination_drive: "U:"
  network_path: "\\\\thinclient\\cp\\lcs_data"
  files_to_copy: ["utimphis.csv", "imdiary.csv", "lcimp002.csv"]
  description: |
    Copy the processed files to the LCS data directory:
    - Map the network drive if not already mapped.
    - Copy each file to the destination, overwriting existing files if necessary.
    - Verify the integrity of copied files.
    - Ensure proper permissions are maintained during the copy process.

update_processing_history:
  db_path: "./data/processing_history.db"
  description: |
    Update the processing history database with details of the current run:
    - Record the processing date, number of files processed, and records affected.
    - Generate a summary of the import process, including any notable events or issues.
    - Ensure the database is properly updated to inform future processing runs.

error_handling:
  max_retries: 3
  retry_delay: 60  # seconds
  description: |
    Guidelines for handling errors during the process:
    - Implement a retry mechanism for transient errors (e.g., network issues).
    - Log all errors with detailed context for troubleshooting.
    - For critical errors, pause the process and seek user intervention.
    - Provide clear error messages and suggested actions for resolution.

reporting:
  summary_file: "./logs/import_summary.txt"
  notification_email: "admin@example.com"
  description: |
    Generate comprehensive reports and notifications:
    - Create a detailed summary of the entire import process.
    - Highlight key statistics, any issues encountered, and resolutions applied.
    - Send email notifications for process completion or critical errors.
    - Maintain an audit trail of all significant actions and decisions.

```

3.2. Adjust paths and settings according to your specific setup.

## 4. Implement Tool Functions

4.1. In `tools/file_operations.py`, implement:
     - `clean_workspace()`
     - `retrieve_new_input_files()`
     - `copy_to_lcs_data()`

4.2. In `tools/data_processing.py`, implement:
     - `determine_processing_scope()`
     - `consolidate_input_files()`
     - `calculate_statute_of_limitations()`
     - `generate_input_files()`
     - `update_processing_history()`

4.3. In `tools/external_execution.py`, implement:
     - `process_input_files()`

Each function should follow the structure outlined in the tool creation guidelines, including proper error handling and logging.

## 5. Implement Main Orchestrator

5.1. In `main.py`, implement the main orchestrator function:
     - Load configuration
     - Set up logging
     - Create OpenAI assistant
     - Create thread
     - Run the orchestration loop

5.2. Implement error handling and timeout mechanisms in the main loop.

## 6. Set Up Database

6.1. Create a script to initialize the SQLite database:
     ```python
     import sqlite3

     conn = sqlite3.connect('./data/processing_history.db')
     cursor = conn.cursor()

     cursor.execute('''
     CREATE TABLE IF NOT EXISTS processing_history (
         id INTEGER PRIMARY KEY AUTOINCREMENT,
         processing_date DATE,
         files_processed INTEGER,
         records_processed INTEGER,
         summary TEXT
     )
     ''')

     conn.commit()
     conn.close()
     ```

6.2. Run this script to create the database and table.

## 7. Testing

7.1. Create unit tests for each tool function in the `tests/` directory.
7.2. Create integration tests to ensure the entire workflow functions correctly.

## 8. Documentation

8.1. Add docstrings to all functions explaining their purpose, parameters, and return values.
8.2. Create a user manual explaining how to use the TAO Agent system.

## 9. Error Handling and Logging

9.1. Implement comprehensive error handling in each function.
9.2. Set up logging to capture important events, errors, and the overall flow of execution.

## 10. Security Considerations

10.1. Ensure sensitive information (like file paths and network credentials) is stored securely.
10.2. Implement proper access controls for the SQLite database and output files.

## 11. Performance Optimization

11.1. Profile the code to identify any performance bottlenecks.
11.2. Optimize file operations and data processing functions as needed.

## 12. Deployment

12.1. Set up a production environment with all necessary dependencies.
12.2. Create a deployment script to automate the setup process.

## 13. Maintenance and Monitoring

13.1. Implement monitoring to track the system's performance and detect any issues.
13.2. Set up automated alerts for critical errors or process failures.

By following this implementation guide, you will create a robust, efficient, and maintainable Task Automation Orchestrator Agent capable of handling the complex workflow of importing Statute of Limitations data on files.
