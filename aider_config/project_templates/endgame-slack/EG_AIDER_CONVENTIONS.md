

# High Level Overview

- Use the AIDER_README.md file for instructions on how to interact with me the engineer and the process by which we will build the Slack integration.
- This module will build a Slack integration using FastAPI, HTTPX, Pydantic, Ruff, and Pytest, using Paragon as the workflow engine.
- Paragon handles OAuth authentication and authorization and user token management. When we need to interact with Slack's API, we will use Paragon's Connect API
  with a [Paragon User Token](<AIDER_CONVENTIONS#Paragon Authentication Process, Token Management, and User Metadata Enrichment>) as a proxy.

# General Tooling Direction

- **FastAPI**: Use FastAPI for building web APIs. It is a modern, fast, and easy-to-use web framework for building APIs with Python.
- **HTTPX**: Use HTTPX for making HTTP requests, when needed. It is a modern, asynchronous HTTP client for Python.
- **Pydantic**: Use Pydantic for data validation and serialization. It is a fast, modern, and easy-to-use data validation and settings management library.
- **Ruff**: Use Ruff for linting and formatting Python code. It is a fast, extensible, and opinionated linter for Python.
- **Pytest**: Use Pytest for testing Python code. It is a mature, well-maintained, and easy-to-use testing framework for Python.

# Functional Architecture of Slack Integration

## Functional Design of the API: use this to build the Slack integration.

1. The Endgame Connection User will authenticate and authorize their Slack account. Paragon will trigger the Slack oAuth flow and handle the oAuth callback.
2. Paragon workflow will trigger a Call to Slack's Conversation API to list all Channels and publish the results to an API endpoint we'll create..
3. Paragon workflow will trigger a Call to Slack's User API to list all that Slack team's Users and publish the results to an API endpoint we'll create..
4. We will store the raw results in the database using Pydantic models and calls to the Mesh Engine.
5. We will generate a Paragon User Token and store it in the mesh database associated to the Endgame Connection User.
6. We will present the list of channels to the Endgame Connection User so that they can select the ones Endgame will be authorized to consume conversations as an Allow List.
7. We will track the list of channels the user has selected in a metadata field on the Paragon User's Record via the Paragon Users API.
8. We will generate an API call to Paragon's Connect API to trigger a workflow that will use the Allow Listed channels metadata field and retrieve all conversations since last sync. (first sync will be all time) 1. We will store the raw results in the database using Pydantic models and calls to the Mesh Engine and record the sync time in the database, therefore we will need to implement a mechanism to handle the storage of the raw results in the database and a model for tracking the sync status for the Slack Team.

# Communication Architecture: Paragon

## Paragon's API instructions

Paragon enables communication between Endgame and 3rd party and other services. It handles OAuth authentication and authorization and user token management. When we need to interact with Slack's API, we will use Paragon's Connect API
with a [Paragon User Token](<AIDER_CONVENTIONS#Paragon Authentication Process, Token Management, and User Metadata Enrichment>).

## Paragon Authentication Process, Token Management, and User Metadata Enrichment

1.  Users will use Paragon's Connect Page to authenticate and authorize their Slack account. We will publish an api to enrich the Paragon Connected User Metadata
    with Endgame's UUID/Org id.
2.  We will use Endgame's UUID, Org Id, and Paragon Signing Key to generate a Paragon User Token. (Signed JWT: see [slack/util/paragon-jwt.py](/slack/util/paragon-jwt.py))
    a. JWT Requirements:

    - sub: Endgame's UUID/Endgame's Org ID
    - exp: Expiration Time: current time + 1 hour
    - iat: Issued At Time: current time
    - alg: "RS256"

3.  Paragon Connect API usage instructions:
    Paragon works but providing a proxy api, called the connect api which will be used along with the Paragon User token to authenticate and authorize requests to the Slack API.
    a. Paragon Connect API:

    - Endpoint Root: `https://proxy.useparagon.com/projects/<Paragon Project ID>/sdk/proxy/<Integration Name>/<API Path>`
      where:
      - `<Paragon Project ID>`: The ID of the Paragon project that the integration is associated with.
      - `<Integration Name>`: The name of the integration that is being used.
      - `<API Path>`: The path of the API endpoint that is being accessed.
    - Authorization: Bearer <Paragon User Token>
    - Content-Type: application/json
    - Accept: application/json
    - when there's a response body that is not JSON such as a binary file or image or pdf, passing the `X-Paragon-Use-Raw-Response: 1` header
      will allow the receipt of raw response data including HTTP headers from teh 3rd party API.
      b. example json call:

      ````POST https://proxy.useparagon.com/projects/19d...012/sdk/proxy/slack/chat.postMessage

          Authorization: Bearer eyJ...
          Content-Type: application/json

          {
              "channel": "CXXXXXXX0",
              "text": "This message was sent with Paragon Connect :exploding_head:"
          }```
      ````

c. example non-json call:

`````GET https://proxy.useparagon.com/projects/19d...012/sdk/proxy/googledrive/files/<File ID>/?alt=media

      Authorization: Bearer eyJ...
      X-Paragon-Use-Raw-Response: 1```

4. Paragon Users API Details:
   a. Paragon User API:

   - Endpoint Root: `https://api.useparagon.com/projects/<Paragon Project ID>/sdk/...`
   - Authorization: Bearer <Paragon User Token>
     b. Example call adding custom metadata to the User profile:

   ````PATCH https://api.useparagon.com/projects/<Project ID>/sdk/me

   // Headers
   Authorization: <Paragon User Token>
   Content-Type: application/json

   // Body
   { "meta": { "Email": "sean@useparagon.com" } }```
   c. example call to retrieve the connected user's info and integration state:
     ```Request

     GET https://api.useparagon.com/projects/<Project ID>/sdk/me

     // Headers
     Authorization: Bearer <Paragon User Token>

     Response

      {
       "authenticated": true,
       "integrations": {
         "salesforce": {
           "enabled": true,
           "credentialStatus": "VALID",
           "providerData": {...},
           "providerId": "00502000..."
         }
       },
       "meta": {...},
       "userId": "12345"
     }```

`````

# Slack APIs/Paragon Requests and Schema Data Considerations

1. We will not use the Slack API's directly. We will call Paragon's Connect API to trigger workflows, those workflows will be responsible for interacting with Slack. We will provide endpoints to capture the data those workflows return, for now.
2. For the raw slack response data, we will Consume it, and only move allowed data to be stored in the internal models

## Request Sequence Flow

There are 2 main requests that we'll make it Paragon's APIs and 3 that we'll listen for.

| Request Step | Direction (from this application's reference) | Request Name                       | Description                                                                          | Paragon Workflow (or step)   | Request Type |
| ------------ | --------------------------------------------- | ---------------------------------- | ------------------------------------------------------------------------------------ | ---------------------------- | ------------ |
| 1            | Incoming (need API Route)                     | Get all Slack Channels             | Capture all Slack channels accessible to the user                                    | Get All Slack Channels       | POST         |
| 2            | Incoming (need API Route)                     | Get all Slack Users                | Capture all Slack users in team                                                      | Get all Slack Users          | POST         |
| 3            | Outgoing                                      | Populate Allowlisted Channels Meta | Create Paragon User API Metadata                                                     | N/A                          | PATCH        |
| 4            | Outgoing                                      | Get All Allowlisted Messages       | Call Paragon Connect API to retrieve all messages from allowlisted channels          | Get All Allowlisted Messages | POST         |
| 5            | Incoming (Need API Route)                     | Receive Allowlisted Messages       | Endpoint to Receive Allowlisted Messages invoked post "Get All Allowlisted Messages" | N/A                          | POST         |

### Outgoing Request Details

3. Populate Allowlisted Channels Meta

   - Description: Create Paragon User API Metadata
   - Paragon Workflow:
   - Request Type: PATCH
   - Endpoint: Paragon User API Endpoint: https://api.useparagon.com/projects/<Project ID>/sdk/me

   // Headers
   Authorization: <Paragon User Token>
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
   - Description: Call Paragon Connect API to retrieve all messages from allowlisted channels
   - Paragon Workflow: N/A
   - Request Type: POST
   - Endpoint: https://zeus.useparagon.com/projects/24a3dcf7-6e6e-4d2f-b87a-3d521435c5f3/sdk/triggers/66cc3817-38fa-4099-9025-2054dac0e89e
   - Parameters:
   - include_all_metadata
   - inclusive
   - oldest
   - limit
   - latest

# Models for Data Collection to Service Mesh

The following models are my expectations for data collection to service mesh but ARCHITECT should recommend others as are needed:

1. Response Data From Slack
   a. slack_user: represents a user in the Slack organization.
   c. slack_channel: represents a channel in the Slack organization. Contains TeamID information.
   d. slack_message: represents a message in the Slack channel.
   e. Examples of raw slack api response jsons are stored in [./tests/slack/schematestdata/]
2. Internal Models
   a. user: represents a user in the Endgame organization with two types: connection user and org user
   b. allowed_slack_channel: represents a slack channel in the Endgame Customer's slack organization that Endgame is allowed to monitor
   c. message: represents a message in the Endgame Customer's slack organization that Endgame is allowed to monitor.
3. Service operational Models
   a. as needed

# Interactions with Endgame's Service Mesh and Using Common-py modules

for

1. Writing data to Endgame's Service mesh:
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

- [common-py/mesh](../common-py/mesh/README.md)
- [common-py/core](../common-py/core/README.md)
- [common-py/gcp](../common-py/gcp/README.md)
