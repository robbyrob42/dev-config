

# High Level Overview

- Use the AIDER_README.md file for instructions on how to interact with me the engineer and the process by which we will build the {{external_service}} integration.
- This module will build a {{external_service}} integration using FastAPI, HTTPX, Pydantic, Ruff, and Pytest, using {{external_service}} as the workflow engine.

# General Tooling Direction

- **FastAPI**: Use FastAPI for building web APIs. It is a modern, fast, and easy-to-use web framework for building APIs with Python.
- **HTTPX**: Use HTTPX for making HTTP requests, when needed. It is a modern, asynchronous HTTP client for Python.
- **Pydantic**: Use Pydantic for data validation and serialization. It is a fast, modern, and easy-to-use data validation and settings management library.
- **Ruff**: Use Ruff for linting and formatting Python code. It is a fast, extensible, and opinionated linter for Python.
- **Pytest**: Use Pytest for testing Python code. It is a mature, well-maintained, and easy-to-use testing framework for Python.

# Functional Architecture of {{project_name}}

## Functional Design of the API: use this to build the {{module_name}} integration.

1. step 1
2. step 2
3. step 3

# Communication Architecture: {{external_service}}

## {{external_service}} API instructions

{{external_service}} enables communication between {{core_service}} and 3rd party/other services. It handles OAuth authentication and authorization and user token management. When we need to interact with {{external_service}}'s API, we will use {{external_service}}'s Connect API
with a [{{external_service}} User Token](<AIDER_CONVENTIONS#{{external_service}} Authentication Process, Token Management, and User Metadata Enrichment>).

## {{external_service}} Authentication Process, Token Management, and User Metadata Enrichment

1.  Users will use {{external_service}}'s Connect Page to authenticate and authorize their {{external_service}} account. We will publish an api to enrich the 
2.  {{internal token requirements}}
    a. JWT Requirements:

    - sub: {{core_service}}'s UUID/{{core_service}}'s Org ID
    - exp: Expiration Time: current time + 1 hour
    - iat: Issued At Time: current time
    - alg: "RS256"

3.  {{external_service}} Connect API usage instructions:
    {{external_service}} works but providing a proxy api, called the connect api which will be used along with the {{external_service}} User token to authenticate and authorize requests to the {{external_service}} API.
    a. Connect API:

    - Endpoint Root: 
      where:
      - `< Project ID>`: The ID of the  project that the integration is associated with.
      - `<Integration Name>`: The name of the integration that is being used.
      - `<API Path>`: The path of the API endpoint that is being accessed.
    - Authorization: Bearer <{{external_service}} User Token>
    - Content-Type: application/json
    - Accept: application/json
    - when there's a response body that is not JSON such as a binary file or image or pdf, passing the `Raw-Response: 1` header
      will allow the receipt of raw response data including HTTP headers from teh 3rd party API.
      b. example json call:

      ````POST https://p..

          Authorization: Bearer ...
          Content-Type: application/json

          {
              "": "",
              "text": "This message was sent with :exploding_head:"
          }```
      ````



4. API Details:
   a. {{external_service}} User API:

   - Endpoint Root: `https://api.use{{external_service}}.com/projects/<{{external_service}} Project ID>/sdk/...`
   

# {{external_service}} APIs/{{external_service}} Requests and Schema Data Considerations

1. We will not use the {{external_service}} API's directly. We will call {{external_service}}'s Connect API to trigger workflows, those workflows will be responsible for interacting with {{external_service}}. We will provide endpoints to capture the data those workflows return, for now.
2. For the raw {{external_service}} response data, we will Consume it, and only move allowed data to be stored in the internal models

## Request Sequence Flow

There are 2 main requests that we'll make it {{external_service}}'s APIs and 3 that we'll listen for.

| Request Step | Direction (from this application's reference) | Request Name                       | Description                                                                          | {{external_service}} Workflow (or step)   | Request Type |
| ------------ | --------------------------------------------- | ---------------------------------- | ------------------------------------------------------------------------------------ | ---------------------------- | ------------ |
| 1            | Incoming (need API Route)                     | Get all {{external_service}} Channels             | Capture all {{external_service}} channels accessible to the user                                    | Get All {{external_service}} Channels       | POST         |
| 2            | Incoming (need API Route)                     | Get all {{external_service}} Users                | Capture all {{external_service}} users in team                                                      | Get all {{external_service}} Users          | POST         |
| 3            | Outgoing                                      | Populate Allowlisted Channels Meta | Create {{external_service}} User API Metadata                                                     | N/A                          | PATCH        |
| 4            | Outgoing                                      | Get All Allowlisted Messages       | Call {{external_service}} Connect API to retrieve all messages from allowlisted channels          | Get All Allowlisted Messages | POST         |
| 5            | Incoming (Need API Route)                     | Receive Allowlisted Messages       | Endpoint to Receive Allowlisted Messages invoked post "Get All Allowlisted Messages" | N/A                          | POST         |

### Outgoing Request Details

3. Populate Allowlisted Channels Meta

   - Description: Create {{external_service}} User API Metadata
   - {{external_service}} Workflow:
   - Request Type: PATCH
   - Endpoint: {{external_service}} User API Endpoint: https://api.use{{external_service}}.com/projects/<Project ID>/sdk/me

   // Headers
   Authorization: <{{external_service}} User Token>
   Content-Type: application/json

   // Body
   ```{
      userid: 'rob@rootsystem.com/end10843',
      allowListedChannels: [
      'C0567MFSJH2',
      'C056AV2JDLK',
      'C05T41DMUTE',
      '......',
      ],
   }```

4. Get All Allowlisted Messages
   - Description: Call {{external_service}} Connect API to retrieve all messages from allowlisted channels
   - {{external_service}} Workflow: N/A
   - Request Type: POST
   - Endpoint: https://zeus.use{{external_service}}.com/projects/24a3dcf7-6e6e-4d2f-b87a-3d521435c5f3/sdk/triggers/66cc3817-38fa-4099-9025-2054dac0e89e
   - Parameters:
   - include_all_metadata
   - inclusive
   - oldest
   - limit
   - latest

# Models for Data Collection to Service Mesh

The following models are my expectations for data collection to service mesh but ARCHITECT should recommend others as are needed:

1. Response Data From {{external_service}}
   a. {{external_service}}_user: represents a user in the {{external_service}} organization.
   c. {{external_service}}_channel: represents a channel in the {{external_service}} organization. Contains TeamID information.
   d. {{external_service}}_message: represents a message in the {{external_service}} channel.
   e. Examples of raw {{external_service}} api response jsons are stored in [./tests/{{external_service}}/schematestdata/]
2. Internal Models
   a. user: represents a user in the {{core_service}} organization with two types: connection user and org user
   b. allowed_{{external_service}}_channel: represents a {{external_service}} channel in the {{core_service}} Customer's {{external_service}} organization that {{core_service}} is allowed to monitor
   c. message: represents a message in the {{core_service}} Customer's {{external_service}} organization that {{core_service}} is allowed to monitor.
3. Service operational Models
   a. as needed

# Interactions with {{core_service}}'s Service Mesh and Using Common-py modules

for

1. Writing data to {{core_service}}'s Service mesh:
   a. Using the common-py/mesh/ library:

   ````## Usage

     ### Basic Usage

     `MeshEngine` is a singleton that is created once and reused for the lifetime of the application or thread. The `MeshSession` is created for each organization, module, and model and is used to send data to the mesh service. It is not thread safe, so you should not share a `MeshSession` between threads. This follows a similar approach as SQLAlchemy.


     The simplest way to use Common Mesh is with a context manager:

     ```python
     from common_mesh import MeshEngine

     with MeshEngine(base_url="http://data-mesh:8675") as engine:
         session = engine.create_session("org", "mod", "mdl")
         session.send_all_models(my_models)
     ```

     `send_all_models` accepts an iterable of Pydantic models, allowing for memory-efficient streaming of large datasets. Models are processed one at a time and sent in compressed batches.

     You can use it with a generator function like this:
     ```python
     from common_mesh import MeshEngine

     def model_generator():
         for item in source_data:
             yield MyDataModel(
                 id=item.id,
                 name=item.name,
                 value=item.value
             )

     with MeshEngine(base_url="http://data-mesh:8675") as engine:
         session = engine.create_session("org", "mod", "mdl")
         session.send_all_models(model_generator())
     ```
   ```
   2. Modules Available for Use from common-py/core
      Common-py/Core Modules

      - **strings.py** - String manipulation utilities focusing on diacritic/accent handling and case-insensitive operations. Useful for international text processing and searching.

      - **temporal.py** - DateTime parsing and manipulation with strong UTC timezone handling. Provides robust parsing of various date formats while ensuring consistent timezone handling.

      - **lists.py** - List manipulation utilities, currently focused on flattening nested structures. Useful for simplifying complex data structures.

      - **markdown.py** - Markdown to plain text conversion while preserving document structure. Handles complex Markdown elements including nested lists, links, and images.

      - **log_manager.py** - Advanced logging configuration with JSON support. Provides structured logging capabilities with customizable formatters and log record decoration.

      - **timing.py** - Performance measurement utilities through decorators. Enables easy function execution timing for performance monitoring.

      - **validators.py** - Basic validation utilities designed to work with Pydantic models. Provides common validation functions for positive numbers and non-empty strings.

      - **serialization.py** - JSON serialization helpers for complex Python objects. Handles datetime, sets, and Pydantic models in a consistent way.
   ````

# Settings and Variables

For this project environment variables and settings are used to configure the application. Here's a summary of where those are located:

- Environment variables are stored in a `op.env` file in the root directory of the project. Any variables coming from 1Password are already mapped to environment variables.
- Settings are stored in a `settings.py` file in the root directory of the project.

# Documentation Sources

