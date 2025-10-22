## IBM-UDI Python SDK

The `ibm-udi` SDK for Python enables developers and data engineers to programmatically interact with the [unstructured data integration service](https://www.ibm.com/docs/en/watsonxdata/saas?topic=data-working-unstructured), IBM’s next-generation data integration platform built to support modern, hybrid-cloud data pipelines.

With this SDK, users can automate and manage Flow lifecycles such as creating, configuring, starting, stopping, and monitoring Flows directly in code.

This repo is just to host the documentation for the opensource SDK https://pypi.org/project/ibm-udi/. Any issues observed or new features, open an issue. 


## Quick Start Guide

## 1. Installation 
Install the SDK via pip

```
pip install ibm-udi
```

## 2. Authenticate
Create a config `dictionary` for authentication. Generate your [IBM Cloud API key](https://cloud.ibm.com/iam/apikeys), if you are using IBM Cloud.

```python
config = {
  "base_url": "<cluster-url>",
  "project_id": "<project-id>",
  "api_key": "<api-key>",   # For SaaS
  "env": "cloud-prod"       # options: cpd | cloud-prod 
}
```
🔑 Authentication options:
| Environment | Required Credentials    |
| ----------- | ----------------------- |
| SaaS        | `api_key`               |
| CPD         | `user_name` + `password` |
| CPD (alt)   | `token`                 |



## 3. Create and Run your first Flow
In this example, we demonstrate how to ingest CPD project assets into Milvus. Make sure to update the asset details (ID, name) and configure your Milvus connection before running the flow.

```python
from udi import UDIClient
from udi.flows import Flow

# Step 0: Config
config = {
  "base_url": "<cluster-url>",
  "project_id": "<project-id>",
  "api_key": "<api-key>",   # For SaaS
  "env": "cloud-prod"       # options: cpd | cloud-prod 
}

# Step 1: Connect
uc = UDIClient(config=config)

# Step 2: Define operators
operators = [
    {
      "type": "ingest_cpd_assets",
      "parameters": {"input_assets": [{"asset_id": "<asset-id>"}]}
    },
    {"type": "extract_cpd"},
    {"type": "embeddings"}
]

# Step 3: Build pipeline
pipeline = {
    "flow_name": "SDK_FLOW",
    "project_id": config["project_id"],
    "orchestrator": "python",
    "flow": operators
}

# Step 4: Run flow
flow = Flow(uc)
flow.create(pipeline["flow_name"], pipeline)
flow.run()
flow.poll_flow_status()   # Wait until it completes

```
Copy the code to `sample_flow.py` 

Run the code. 
```console
python sample_flow.py
```
If everything goes well, the output will be something like this.
```console
[0s] Current status: Queued
[10s] Current status: Completed
Final status: Completed
```

## Step-by-step explanation

### Step 0: Prerequisites (what `config` should contain)
`config` is a dictionary used by `UDIClient` to authenticate and scope the run to your project. At minimum, it must include your `project_id`, `base_url` depending on your environment and API key/token. 

```python
config = {
  "base_url": "<cluster-url>",
  "project_id": "<project-id>",
  "api_key": "<api-key>", # For SaaS
  "env": "cloud-prod" # options: cpd | cloud-prod
}
```
Environement options:
| Environment | env value    |
| ----------- | ----------------------- |
| CA-TOR      | `cloud-prod`             |
| CPD         | `cpd`             |

## Step 1: Connect to the cluster
```python
uc = UDIClient(config=config)
```
- Creates a client bound to your project and cluster provided in the config. 
- All subsequent actions use this client to talk to the UDI backend.

## Step 2: Define operators (the building blocks of the pipeline)
```python
operators = [
  {
    "type": "ingest_cpd_assets",
    "parameters": {"input_assets": [{"asset_id": "<asset-id>"}]}
  },
  {"type": "extract_cpd"},
  {"type": "embeddings"}
]
```
Each operator performs a specific task in the flow:
- `ingest_cpd_assets`: Pulls in assets from CPD using the provided asset_id.
- `extract_cpd`:       Extracts content from the ingested assets.
- `embeddings`:        Generates embeddings for further downstream tasks.

> 💡 **Note:** For a detailed explanation and usage guidelines for each operator, refer to the Operators section.

## Step 3: Build the pipeline
```python
pipeline = {
    "flow_name": "SDK_FLOW",
    "project_id": config["project_id"],
    "orchestrator": "python",
    "flow": operators
}
```
A pipeline (flow) ties operators together with metadata such as:
- flow_name:   A unique name for the UDI flow.
- project_id:  The project identifier from the config, where you want to create your flow.
- orchestrator: Defines the runtime engine(`Python/Spark`)
- flow: The sequence of operators.

## Step 4: Create and run the flow
```python
# - Create the flow in UDI with the given definition.
# - Run the flow immediately after creation.
# - Poll until the flow completes (blocks until done).
flow = Flow(uc)
flow.create(pipeline["flow_name"], pipeline)
flow.run()
flow.poll_flow_status()   # Waits until the flow finishes
```
- `Flow(uc)`: Binds a flow controller to your client.
- `create(name, spec)`: Registers the flow definition on the server with your chosen name.
- `run()`: Starts execution.
- `poll_flow_status()`: Blocks until the flow completes (success or failure).

# Operators
summarizes the available operators grouped by their functional categories.

---
Each operator is defined by both an attribute schema and a feature schema, which together describe how the operator functions and how its data can be processed and filtered.

You can programmatically inspect operators using helper functions:

```python
operator_attributes = get_attributes(metadata, "<operator_name>")
operator_features   = get_features(metadata, "<operator_name>")
```

🧩 Attributes: <br>
Attributes are the configuration options you can set when using an operator. Think of them like the knobs and switches on a machine.<br>
Each attribute usually includes:

        name: This is the display name shown in user interfaces or documentation.
        description: This explains the purpose of the attribute.
        Default: This is the default value used if the user does not provide one.
        Required: This indicates whether the attribute is mandatory.
        valid_values: This indicates allowed options for a particular attribute.

🧩 Features:<br>
Features describe the data that the operator works with or produces. These are like labels or fields in a dataset.<br>
Each Features usually includes:

        name: This is the display name of the feature, shown in interfaces or documentation.
        description: Explains that this field contains a unique identifier for each document.
        available_for_filter: Indicates that this feature can be used to filter documents (e.g., search by ID).
        available_for_vector_db: This feature is required when storing the document in a vector database.
        type: The data type of the feature<br>

### Available Operators
### Ingest Operators
Operators to bring external data into the system.
- ingest_cpd_assets → Ingest from Cloud Pak for Data (CPD) assets
- ingest_cpd_connections → Ingest from external connections (e.g., Amazon S3, ACL)
- ingest_document_set → Ingest from predefined document sets

### Extract Operators
Operators to extract content or metadata from ingested data.
- extract_cpd → Extract metadata or content from CPD assets

### Quality Operators
Operators to assess or improve data/document quality.
- lang_detect → Detect document language
- doc_quality → Evaluate document quality metrics
- sql_filter → Apply SQL-style filtering
- data_class_assignment → Assign data to predefined classes
- term_assignment_operator → Assign business terms or tags
- pii_and_hap_extract_redact → Detect & redact sensitive information (PII/HAP)
- redaction → Redact based on regex patterns

### Functional Operators
Operators for document transformations and processing.
- chunker → Split documents into smaller chunks
- embeddings → Generate embeddings for text

### VectorDB Operators
Operators to interact with vector databases.
- milvusdb_cp4d → Connect to Milvus DB in CPD
- document_set → Manage sets of documents in vector format

💡 Example Usage

Inspect available attributes and features for an operator:

```python
# Get attributes for ingestion operator
operator_attributes = get_attributes(metadata, "ingest_cpd_assets")
print(json.dumps(operator_attributes, indent=2))
```

```python
# Get features for extraction operator
operator_features = get_features(metadata, "extract_cpd")
print(json.dumps(operator_features, indent=2))
```


# Advanced Options
## Upload Custom Operators
Upload Custom Operator
Supports three types of files, all three files are optional and can be uploaded in any order
- custom_operator file which is a .py file
- dependency is a zip file with helper functions required for custom_operator to import from (just zip the dependent files)
- pacakge is a zip file but with third party dependencies like pyarrow etc..(to create this run generate_pacakge.py )

```python
import os

# provide a directory in which file_path, dpendency and package files are present
directory = "Documents/datasift-sdk"

# uncomment and add relavent file name
file_path = os.path.join(os.path.expanduser('~'), directory, "hello_world.py")
# dependency = os.path.join(os.path.expanduser('~'), directory, "custom_operator_package.zip")
package = os.path.join(os.path.expanduser('~'), directory, "my_custom_operator_site_packages.zip")


print(file_path,package)
try:
    test = uc.upload_custom_operator(file_path=file_path,package=package)
    print(test)
except Exception as e:
    print(f"An error occurred during the upload: {e}")
```
## Project Settings
The Project Settings API allows you to manage metadata and storage configuration for a project. You can:
- Retrieve project settings
- Create new project settings
- Update existing project settings

#### Get Project Settings
Retrieves the current project settings for the project.
``` python
get_project_settings = uc.get_project_settings()
print(get_project_settings)
```

#### Create Project Settings
Creates a new project settings object.
Must include container_kind (project) and container_id (project_id).
runtime, acls and storage can be customized as needed.
```python
request_body = {
    "container_kind": "project",
    "container_id": config.get('project_id'),
    "name": "UDI Project Settings",
    "runtime": {},
    "storage": {
        "dataset_storage": {
          "connection_name": "<connection_name>",
          "collection_name": "<collection_name>",
          "schema_name": "<schema_name>",
          "connection_id": "<connection_id>"
        }
    },
    "acls": {}
}

create_project_settings = uc.post_project_settings(request_body=request_body)
print(create_project_settings)
```
#### Update Project Settings
Updates the storage configuration (or other fields) for an existing project settings object.
```python
request_body = {
  "storage": {
    "dataset_storage": {
      "connection_name": "<connection_name>",
      "collection_name": "<collection_name>",
      "schema_name": "<schema_name>",
      "connection_id": "<connection_id>"
    }
  }
}
update_project_settings = uc.patch_project_settings(request_body=request_body)
print(update_project_settings)
```








