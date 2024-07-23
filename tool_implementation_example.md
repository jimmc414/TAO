<Tool Implementation Examples for Task Automation Orchestrator Agent>

# Python tool definitions that connect to the programs and functions in the "Import Statute of Limitations Data on Files" task. These definitions will be designed to work with TAO Agent via the Anthropic or OpenAI API.

```python
tools = [
    {
        "name": "determine_processing_scope",
        "description": "Determines the date range for files to process by checking the last processed date in a SQLite database or prompting the user. It allows the user to confirm or modify the suggested date range.",
        "input_schema": {
            "type": "object",
            "properties": {
                "db_path": {
                    "type": "string",
                    "description": "Path to the SQLite database storing the last processed date. Default is './processing_history.db'."
                },
                "force_user_input": {
                    "type": "boolean",
                    "description": "If true, always prompt the user for date range, ignoring the database. Default is false."
                }
            }
        }
    },
    {
        "name": "clean_workspace",
        "description": "Archives temporary files from the working directory to a timestamped subfolder in the archive directory. It handles various file types including .csv, .txt, .xlsx, and others.",
        "input_schema": {
            "type": "object",
            "properties": {
                "archive_directory": {
                    "type": "string",
                    "description": "Path to the archive directory. Default is './archive'."
                },
                "file_types": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "List of file extensions to archive. Default is ['csv', 'txt', 'xlsx', 'log', 'old', 'docx', 'sql', 'err', 'aud']."
                }
            }
        }
    },
    {
        "name": "retrieve_new_input_files",
        "description": "Copies new input files from a source directory to the working directory based on the specified date range. It handles NCR*.xlsx files and ensures file integrity during copying.",
        "input_schema": {
            "type": "object",
            "properties": {
                "source_directory": {
                    "type": "string",
                    "description": "Path to the source directory containing new input files. Default is 'T:/EDISHARE/NEW CLAIMS/2024'."
                },
                "start_date": {
                    "type": "string",
                    "format": "date",
                    "description": "Start date for file selection in YYYY-MM-DD format."
                },
                "end_date": {
                    "type": "string",
                    "format": "date",
                    "description": "End date for file selection in YYYY-MM-DD format."
                },
                "destination_directory": {
                    "type": "string",
                    "description": "Path to the destination directory for copied files. Default is './temp'."
                }
            },
            "required": ["start_date", "end_date"]
        }
    },
    {
        "name": "consolidate_input_files",
        "description": "Preprocesses and consolidates all input Excel files into a single file, removing invalid records. It handles data validation, header row detection, and ensures data consistency across files.",
        "input_schema": {
            "type": "object",
            "properties": {
                "input_directory": {
                    "type": "string",
                    "description": "Directory containing the input Excel files. Default is './temp'."
                },
                "output_file": {
                    "type": "string",
                    "description": "Path and filename for the consolidated output file. Default is './temp/NCR_combined_output.xlsx'."
                },
                "file_pattern": {
                    "type": "string",
                    "description": "Glob pattern to match input files. Default is 'NCR*.xlsx'."
                }
            }
        }
    },
    {
        "name": "calculate_statute_of_limitations",
        "description": "Calculates the Statute of Limitations (SoL) date for each record based on contract date or charge-off date and state laws. It handles complex date calculations and state-specific rules.",
        "input_schema": {
            "type": "object",
            "properties": {
                "input_file": {
                    "type": "string",
                    "description": "Path to the consolidated input file. Default is './temp/NCR_combined_output.xlsx'."
                },
                "output_file": {
                    "type": "string",
                    "description": "Path and filename for the output file with calculated SoL dates. Default is './temp/output_data.csv'."
                },
                "state_laws_file": {
                    "type": "string",
                    "description": "Path to the JSON file containing state-specific SoL laws. Default is './state_sol_laws.json'."
                }
            }
        }
    },
    {
        "name": "generate_input_files",
        "description": "Generates three input files (utimphis.csv, imdiary.csv, lcimp002.csv) based on the calculated SoL data. It handles complex data transformations and ensures output file integrity.",
        "input_schema": {
            "type": "object",
            "properties": {
                "sol_data_file": {
                    "type": "string",
                    "description": "Path to the file containing calculated SoL data. Default is './temp/output_data.csv'."
                },
                "utimphis_output": {
                    "type": "string",
                    "description": "Path and filename for the utimphis output file. Default is './utimphis.csv'."
                },
                "imdiary_output": {
                    "type": "string",
                    "description": "Path and filename for the imdiary output file. Default is './imdiary.csv'."
                },
                "lcimp002_output": {
                    "type": "string",
                    "description": "Path and filename for the lcimp002 output file. Default is './lcimp002.csv'."
                }
            }
        }
    },
    {
        "name": "process_input_files",
        "description": "Copies the generated input files to their respective directories and processes them using the acuthin.exe program. It handles file transfers, subprocess management, and detailed logging.",
        "input_schema": {
            "type": "object",
            "properties": {
                "utimphis_file": {
                    "type": "string",
                    "description": "Path to the utimphis.csv file. Default is './utimphis.csv'."
                },
                "imdiary_file": {
                    "type": "string",
                    "description": "Path to the imdiary.csv file. Default is './imdiary.csv'."
                },
                "lcimp002_file": {
                    "type": "string",
                    "description": "Path to the lcimp002.csv file. Default is './lcimp002.csv'."
                },
                "acuthin_path": {
                    "type": "string",
                    "description": "Path to the acuthin.exe program. Default is '\\\\THINCLIENT\\thinclient\\acuthin.exe'."
                },
                "log_file": {
                    "type": "string",
                    "description": "Path and filename for the detailed log file. Default is './import_log.txt'."
                }
            }
        }
    },
    {
        "name": "copy_to_lcs_data",
        "description": "Copies the generated files (utimphis.csv, imdiary.csv, lcimp002.csv) to the LCS data directory. It handles network path mapping, file integrity checks, and overwrites existing files.",
        "input_schema": {
            "type": "object",
            "properties": {
                "source_directory": {
                    "type": "string",
                    "description": "Directory containing the files to be copied. Default is './'."
                },
                "destination_drive": {
                    "type": "string",
                    "description": "Drive letter to map the network path. Default is 'U:'."
                },
                "network_path": {
                    "type": "string",
                    "description": "Network path to map. Default is '\\\\thinclient\\cp\\lcs_data'."
                },
                "files_to_copy": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "List of files to copy. Default is ['utimphis.csv', 'imdiary.csv', 'lcimp002.csv']."
                }
            }
        }
    },
    {
        "name": "update_processing_history",
        "description": "Updates the SQLite database with the latest processing date and summary information. It handles database connections, SQL transactions, and error recovery.",
        "input_schema": {
            "type": "object",
            "properties": {
                "db_path": {
                    "type": "string",
                    "description": "Path to the SQLite database. Default is './processing_history.db'."
                },
                "processing_date": {
                    "type": "string",
                    "format": "date",
                    "description": "Date of the current processing run in YYYY-MM-DD format."
                },
                "files_processed": {
                    "type": "integer",
                    "description": "Number of files processed in this run."
                },
                "records_processed": {
                    "type": "integer",
                    "description": "Number of records processed in this run."
                },
                "summary": {
                    "type": "string",
                    "description": "Brief summary of the processing run."
                }
            },
            "required": ["processing_date", "files_processed", "records_processed", "summary"]
        }
    }
]
```

These tool definitions provide a comprehensive and detailed description of each tool's functionality, inputs, and default values. They cover all the major steps in the "Import Statute of Limitations Data on Files" task, including:

1. Determining the processing scope
2. Cleaning the workspace
3. Retrieving new input files
4. Consolidating input files
5. Calculating Statute of Limitations dates
6. Generating input files for different systems
7. Processing the generated input files
8. Copying files to the LCS data directory
9. Updating the processing history

Each tool definition includes:

- A clear and descriptive name
- A detailed description of its functionality
- An input schema that specifies the expected input parameters, their types, and descriptions
- Default values for optional parameters
- Required parameters where applicable

These definitions allow TAO Agent to understand the purpose and requirements of each tool, enabling it to make informed decisions about when and how to use them in the workflow. The level of detail provided ensures that the TAO agent can handle various scenarios and edge cases that might arise during the execution of the task.
</Tool Implementation Examples for Task Automation Orchestrator Agent>
