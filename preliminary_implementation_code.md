```python
import os
from openai import OpenAI
from time import sleep
import yaml
from typing import Dict, Any, List
import json
import sqlite3
from datetime import datetime, timedelta
import shutil
import pandas as pd
import subprocess
import logging
from logging.handlers import RotatingFileHandler

# Initialize OpenAI client
client = OpenAI()

# Set up logging
def setup_logging(log_file: str = 'tao_agent.log'):
    logger = logging.getLogger('TAOAgent')
    logger.setLevel(logging.DEBUG)
    handler = RotatingFileHandler(log_file, maxBytes=10*1024*1024, backupCount=5)
    formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
    handler.setFormatter(formatter)
    logger.addHandler(handler)
    return logger

logger = setup_logging()

# Load configuration
def load_config(config_file: str) -> Dict[str, Any]:
    with open(config_file, 'r') as file:
        return yaml.safe_load(file)

# Create or retrieve assistant
def create_assistant(tools: List[Dict[str, Any]], config: Dict[str, Any]) -> Any:
    assistant = client.beta.assistants.create(
        name=config['assistant']['name'],
        instructions=config['assistant']['instructions'],
        model=config['assistant']['model'],
        tools=tools
    )
    logger.info(f"Assistant created with ID: {assistant.id}")
    return assistant

# Create a thread
def create_thread() -> Any:
    thread = client.beta.threads.create()
    logger.info(f"Thread created with ID: {thread.id}")
    return thread

# Send a message to the thread
def send_message(thread_id: str, content: str) -> None:
    client.beta.threads.messages.create(
        thread_id=thread_id,
        role="user",
        content=content
    )
    logger.info(f"Message sent to thread {thread_id}: {content}")

# Run the assistant
def run_assistant(thread_id: str, assistant_id: str) -> Any:
    run = client.beta.threads.runs.create(
        thread_id=thread_id,
        assistant_id=assistant_id
    )
    logger.info(f"Run created with ID: {run.id}")
    return run

# Check run status
def get_run_status(thread_id: str, run_id: str) -> str:
    run = client.beta.threads.runs.retrieve(thread_id=thread_id, run_id=run_id)
    logger.info(f"Run {run_id} status: {run.status}")
    return run.status

# Tool implementations

def determine_processing_scope(db_path: str, force_user_input: bool = False) -> Dict[str, Any]:
    logger.info("Determining processing scope")
    if force_user_input:
        start_date = input("Enter the start date for processing (YYYY-MM-DD): ")
        end_date = input("Enter the end date for processing (YYYY-MM-DD): ")
    else:
        try:
            conn = sqlite3.connect(db_path)
            cursor = conn.cursor()
            cursor.execute("SELECT MAX(processing_date) FROM processing_history")
            last_date = cursor.fetchone()[0]
            if last_date:
                last_date = datetime.strptime(last_date, '%Y-%m-%d').date()
                start_date = (last_date + timedelta(days=1)).strftime('%Y-%m-%d')
                end_date = datetime.today().strftime('%Y-%m-%d')
            else:
                start_date = input("Enter the start date for processing (YYYY-MM-DD): ")
                end_date = input("Enter the end date for processing (YYYY-MM-DD): ")
            conn.close()
        except sqlite3.Error as e:
            logger.error(f"Database error in determine_processing_scope: {e}")
            raise ValueError(f"Database error: {e}")
    logger.info(f"Processing scope determined: {start_date} to {end_date}")
    return {"start_date": start_date, "end_date": end_date}

def clean_workspace(archive_directory: str, file_types: List[str]) -> Dict[str, Any]:
    logger.info("Cleaning workspace")
    timestamp = datetime.now().strftime('%Y%m%d%H%M%S')
    archive_folder = os.path.join(archive_directory, f'archive_{timestamp}')
    os.makedirs(archive_folder, exist_ok=True)

    archived_files = []

    for file_type in file_types:
        for file in os.listdir('.'):
            if file.endswith(f'.{file_type}'):
                shutil.move(file, os.path.join(archive_folder, file))
                archived_files.append(file)

    logger.info(f"Workspace cleaned. Archived {len(archived_files)} files to {archive_folder}")
    return {"archive_folder": archive_folder, "archived_files": archived_files}

def retrieve_new_input_files(source_directory: str, start_date: str, end_date: str, destination_directory: str) -> Dict[str, Any]:
    logger.info(f"Retrieving new input files from {source_directory}")
    os.makedirs(destination_directory, exist_ok=True)

    copied_files = []

    for file in os.listdir(source_directory):
        if file.startswith("NCR") and file.endswith(".xlsx"):
            file_date = datetime.strptime(file[3:11], '%Y%m%d').date()
            if datetime.strptime(start_date, '%Y-%m-%d').date() <= file_date <= datetime.strptime(end_date, '%Y-%m-%d').date():
                shutil.copy(os.path.join(source_directory, file), os.path.join(destination_directory, file))
                copied_files.append(file)

    logger.info(f"Retrieved {len(copied_files)} new input files")
    return {"copied_files": copied_files, "destination_directory": destination_directory}

def consolidate_input_files(input_directory: str, output_file: str, file_pattern: str) -> Dict[str, Any]:
    logger.info("Consolidating input files")
    combined_data = pd.DataFrame()

    for file in os.listdir(input_directory):
        if file.startswith("NCR") and file.endswith(".xlsx"):
            file_path = os.path.join(input_directory, file)
            data = pd.read_excel(file_path)
            combined_data = pd.concat([combined_data, data])

    combined_data = combined_data.dropna()
    combined_data.to_excel(output_file, index=False)

    logger.info(f"Input files consolidated. {len(combined_data)} records processed")
    return {"output_file": output_file, "records_processed": len(combined_data)}

def calculate_statute_of_limitations(input_file: str, output_file: str, state_laws_file: str) -> Dict[str, Any]:
    logger.info("Calculating Statute of Limitations")
    with open(state_laws_file, 'r') as f:
        state_laws = json.load(f)

    data = pd.read_excel(input_file)
    sol_dates = []

    for index, row in data.iterrows():
        state = row['State']
        contract_date = row['ContractDate']
        charge_off_date = row['ChargeOffDate']

        sol_period = state_laws[state]
        base_date = contract_date if not pd.isnull(contract_date) else charge_off_date

        sol_date = base_date + timedelta(days=sol_period * 365)
        sol_dates.append(sol_date)

    data['SoLDate'] = sol_dates
    data.to_csv(output_file, index=False)

    logger.info(f"SoL calculation completed. {len(data)} records processed")
    return {"output_file": output_file, "records_processed": len(data)}

def generate_input_files(sol_data_file: str, utimphis_output: str, imdiary_output: str, lcimp002_output: str) -> Dict[str, Any]:
    logger.info("Generating input files")
    data = pd.read_csv(sol_data_file)

    utimphis_data = data[['AccountNumber', 'SoLDate']]
    imdiary_data = data[['AccountNumber', 'SoLDate', 'DiaryNotes']]
    lcimp002_data = data[['AccountNumber', 'SoLDate', 'Lcimp002Field']]

    utimphis_data.to_csv(utimphis_output, index=False)
    imdiary_data.to_csv(imdiary_output, index=False)
    lcimp002_data.to_csv(lcimp002_output, index=False)

    logger.info(f"Input files generated. {len(data)} records processed")
    return {
        "utimphis_output": utimphis_output,
        "imdiary_output": imdiary_output,
        "lcimp002_output": lcimp002_output,
        "records_processed": len(data)
    }

def process_input_files(utimphis_file: str, imdiary_file: str, lcimp002_file: str, acuthin_path: str, log_file: str) -> Dict[str, Any]:
    logger.info("Processing input files")
    try:
        with open(log_file, 'w') as log:
            subprocess.run([acuthin_path, utimphis_file, imdiary_file, lcimp002_file], check=True, stdout=log, stderr=log)
        logger.info("Input files processed successfully")
        return {"log_file": log_file, "status": "success"}
    except subprocess.CalledProcessError as e:
        logger.error(f"Processing failed: {e}")
        raise ValueError(f"Processing failed: {e}")

def copy_to_lcs_data(source_directory: str, destination_drive: str, network_path: str, files_to_copy: List[str]) -> Dict[str, Any]:
    logger.info("Copying files to LCS data")
    try:
        os.system(f'net use {destination_drive} {network_path}')
        copied_files = []

        for file in files_to_copy:
            source_file = os.path.join(source_directory, file)
            destination_file = os.path.join(destination_drive, file)
            shutil.copy(source_file, destination_file)
            copied_files.append(destination_file)

        logger.info(f"Files copied to LCS data: {', '.join(copied_files)}")
        return {"copied_files": copied_files, "destination_drive": destination_drive}
    except Exception as e:
        logger.error(f"File copy failed: {e}")
        raise ValueError(f"File copy failed: {e}")

def update_processing_history(db_path: str, processing_date: str, files_processed: int, records_processed: int, summary: str) -> Dict[str, Any]:
    logger.info("Updating processing history")
    try:
        conn = sqlite3.connect(db_path)
        cursor = conn.cursor()
        cursor.execute("""
            INSERT INTO processing_history (processing_date, files_processed, records_processed, summary)
            VALUES (?, ?, ?, ?)
        """, (processing_date, files_processed, records_processed, summary))
        conn.commit()
        conn.close()
        logger.info("Processing history updated successfully")
        return {"status": "success"}
    except sqlite3.Error as e:
        logger.error(f"Database error in update_processing_history: {e}")
        raise ValueError(f"Database error: {e}")

# Process tool calls
def process_tool_calls(tool_calls: List[Dict[str, Any]], config: Dict[str, Any]) -> List[Dict[str, Any]]:
    tool_outputs = []
    for tool_call in tool_calls:
        function_name = tool_call.function.name
        function_args = json.loads(tool_call.function.arguments)
        
        try:
            if function_name == "determine_processing_scope":
                result = determine_processing_scope(**function_args, **config['determine_processing_scope'])
            elif function_name == "clean_workspace":
                result = clean_workspace(**function_args, **config['clean_workspace'])
            elif function_name == "retrieve_new_input_files":
                result = retrieve_new_input_files(**function_args, **config['retrieve_new_input_files'])
            elif function_name == "consolidate_input_files":
                result = consolidate_input_files(**function_args, **config['consolidate_input_files'])
            elif function_name == "calculate_statute_of_limitations":
                result = calculate_statute_of_limitations(**function_args, **config['calculate_statute_of_limitations'])
            elif function_name == "generate_input_files":
                result = generate_input_files(**function_args, **config['generate_input_files'])
            elif function_name == "process_input_files":
                result = process_input_files(**function_args, **config['process_input_files'])
            elif function_name == "copy_to_lcs_data":
                result = copy_to_lcs_data(**function_args, **config['copy_to_lcs_data'])
            elif function_name == "update_processing_history":
                result = update_processing_history(**function_args, **config['update_processing_history'])
            else:
                result = {"error": f"Unknown function: {function_name}"}
        except Exception as e:
            logger.error(f"Error in {function_name}: {str(e)}")
            result = {"error": f"Error in {function_name}: {str(e)}"}
        
        tool_outputs.append({
            "tool_call_id": tool_call.id,
            "output": json.dumps(result)
        })
    
    return tool_outputs

# Main orchestrator function
def main():
    config = load_config('config/sample_config.yaml')
    
    # Define tools based on the functions in your tools directory
    tools = [
        {
            "type": "function",
            "function": {
                "name": "determine_processing_scope",
                "description": "Determines the date range for files to process",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "db_path": {"type": "string"},
                        "force_user_input": {"type": "boolean"}
                    }
                }
            }
        },
        {
            "type": "function",
            "function": {
                "name": "clean_workspace",
                "description": "Archives temporary files from the working directory",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "archive_directory": {"type": "string"},
                        "file_types": {
                            "type": "array",
                            "items": {"type": "string"}
                        }
                    }
                }
            }
        },
        {
            "type": "function",
            "function": {
                "name": "retrieve_new_input_files",
                "description": "Copies new input files from a source directory to the working directory based on the specified date range",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "source_directory": {"type": "string"},
                        "start_date": {"type": "string"},
                        "end_date": {"type": "string"},
                        "destination_directory": {"type": "string"}
                    },
                    "required": ["source_directory", "start_date", "end_date"]
                }
            }
        },
        {
            "type": "function",
            "function": {
                "name": "consolidate_input_files",
                "description": "Preprocesses and consolidates all input Excel files into a single file",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "input_directory": {"type": "string"},
                        "output_file": {"type": "string"},
                        "file_pattern": {"type": "string"}
                    }
                }
            }
        },
        {
            "type": "function",
            "function": {
                "name": "calculate_statute_of_limitations",
                "description": "Calculates the Statute of Limitations (SoL) date for each record",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "input_file": {"type": "string"},
                        "output_file": {"type": "string"},
                        "state_laws_file": {"type": "string"}
                    }
                }
            }
        },
        {
            "type": "function",
            "function": {
                "name": "generate_input_files",
                "description": "Generates three input files (utimphis.csv, imdiary.csv, lcimp002.csv) based on the calculated SoL data",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "sol_data_file": {"type": "string"},
                        "utimphis_output": {"type": "string"},
                        "imdiary_output": {"type": "string"},
                        "lcimp002_output": {"type": "string"}
                    }
                }
            }
        },
        {
            "type": "function",
            "function": {
                "name": "process_input_files",
                "description": "Processes the generated input files using the acuthin.exe program",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "utimphis_file": {"type": "string"},
                        "imdiary_file": {"type": "string"},
                        "lcimp002_file": {"type": "string"},
                        "acuthin_path": {"type": "string"},
                        "log_file": {"type": "string"}
                    }
                }
            }
        },
        {
            "type": "function",
            "function": {
                "name": "copy_to_lcs_data",
                "description": "Copies the generated files to the LCS data directory",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "source_directory": {"type": "string"},
                        "destination_drive": {"type": "string"},
                        "network_path": {"type": "string"},
                        "files_to_copy": {
                            "type": "array",
                            "items": {"type": "string"}
                        }
                    }
                }
            }
        },
        {
            "type": "function",
            "function": {
                "name": "update_processing_history",
                "description": "Updates the SQLite database with the latest processing information",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "db_path": {"type": "string"},
                        "processing_date": {"type": "string"},
                        "files_processed": {"type": "integer"},
                        "records_processed": {"type": "integer"},
                        "summary": {"type": "string"}
                    },
                    "required": ["processing_date", "files_processed", "records_processed", "summary"]
                }
            }
        }
    ]

    # Create assistant
    assistant = create_assistant(tools, config)

    # Create thread
    thread = create_thread()

    # Send initial message
    send_message(thread.id, config['initial_message'])

    # Run the assistant
    run = run_assistant(thread.id, assistant.id)

    # Handle long-running processes
    timeout = config['run_timeout']  # Get timeout from config
    start_time = datetime.now()

    while True:
        if (datetime.now() - start_time).total_seconds() > timeout:
            logger.warning(f"Run timed out after {timeout} seconds")
            break

        status = get_run_status(thread.id, run.id)

        if status == 'completed':
            logger.info("Run completed successfully")
            break
        elif status == 'requires_action':
            required_action = client.beta.threads.runs.retrieve(thread_id=thread.id, run_id=run.id).required_action
            tool_outputs = process_tool_calls(required_action.submit_tool_outputs.tool_calls, config)
            client.beta.threads.runs.submit_tool_outputs(thread_id=thread.id, run_id=run.id, tool_outputs=tool_outputs)
        elif status in ['failed', 'cancelled', 'expired']:
            logger.error(f"Run ended with status: {status}")
            break
        
        sleep(config['poll_interval'])  # Wait before checking again

    # Retrieve and log the final messages
    messages = client.beta.threads.messages.list(thread_id=thread.id)
    for message in messages.data:
        if message.role == "assistant":
            logger.info(f"Assistant's final message: {message.content[0].text.value}")

if __name__ == "__main__":
    main()	
```
