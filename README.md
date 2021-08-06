# Anypoint API Assets API

## Description
The Anypoint API Assets API (aka **Publish Exchange Assets API**) is a Mule application that syncs **select** Anypoint API assets from a **source** Anypoint Platform to a **target** Anypoint Platform. For example, from PCE (Private Cloud Edition) to MuleSoft GovCloud.

**Purpose**: The main purpose of this app is to supply Anypoint Platform (Commerical or GovCloud iPaaS) with API assets and content so that it can in turn supply content for API Community Manager (ACM).

**Additional Context**: For a recent MuleSoft customer, ACM serves as the Developer Portal to market and promote the customer's API products as well as to facilitate engagement and use of their APIs.

## Implementation
This section provides details on the implementation of the Anypoint API Assets API. 

### Mule project files

#### Mule flows (src/main/mule)
- **api.xml**: Contains the main HTTP listener and the APIkit router implementation which directs the request to the proper flows based on the API resource called.
- **main.xml**: Contains the main flows for multiple assets or a single asset, initializes the main variables, and handles the token retrieval used for authentication and authorization to access the APIs in the source and target platforms.
- **exchange-assets.xml**: Contains the implementation for processing Exchange assets.
- **api-instances.xml**: Contains the implementation for processing API instances and SLA tiers.
- **global.xml**: Contains the secure properties configuration, HTTP request configurations for source and target, and general error handler.

#### Resources (src/main/resources)
- **api**: Folder that contains the RAML specification, examples, and exchange modules.
- **properties**: Folder that contains application and environment-specific property yaml files.

### Authentication and Authorization to Anypoint ssource and target
Authentication and authorization to the Anypoint Platform, whether PCE or GovCloud iPaaS, is supported via OAuth tokens. Tokens are obtained in one of two ways:
1. [Connected app](https://docs.mulesoft.com/access-management/connected-apps-overview)
2. x-www-form-urlencoded username and password credentials

_Option 2 is the option used for securing tokens from the current releases of PCE 2.0.2 and 2.1.x. Starting with PCE 3.x and later, the connected app option should be used. MuleSoft GovCloud iPaaS already uses Option 1 (connected app)._

### API assets
Each API asset that is selected for syncing can consist of the following components:
- Exchange assets (of type REST API)
- API instances
    - Main API configuration settings
    - SLA tiers

The API and its available API resources provide for the flexibility to sync:
- Multiple API assets and all of its components
- A single API asset and all of its components
- A single component (e.g. Exchange asset or API instances)
- A single sub-component (e.g. a single API instance or SLA tiers only)

**NOTE**: The syncing and creation of an API asset must follow the proper sequence. That is, the components must be created in the following order:
1. Exchange asset
2. API instance (main configuration settings)
3. SLA tier

The proper sequence is handled automatically when API resources for creating whole assets (i.e. POST /assets or {POST /asset) are used. However, when executing API resources for single components or sub-components, the preceding/prerequisite component must already be in place at the target prior to execution, otherwise an error (e.g. 404 Not found) will occur.

### Single API asset
The following API resources allow for the selection of a single API asset, single API component, or single API sub-component to be synced:
- POST /asset
- POST /exchange/asset
- POST /api-manager/instances
- POST /api-manager/instances/{instanceLabel}
- POST /api-manager/instances/{instanceLabel}/sla-tiers

### Multiple API assets
The following API resources allow for the selection of multiple API assets to be synced:
- POST /assets
- POST /exchange/assets

**Selection criteria**
The selection process starts with the Exchange asset. There are many different types of assets in Anypoint Exchange and not all Exchange assets should be synced from source to target. So the assets are queried based on the following filter criteria:

_ACM-relevant assets_

- REST API asset types
- A mechanism called categories are leveraged to categorize content for ACM relevant assets or assets targeted for syncing.\_

_Sync new or changed assets only (deltas)_

- A mechanism called _tags_ are leveraged to support the identification of changed assets only (from the last sync date).

### Categories and Tags
Besides looking for REST APIs, the application will query for Exchanges assets based on _category_. When an Exchange asset is 
ready to be published to iPaaS, its _category_ will be set to the category designated by you for publishing (e.g. it
 would be something like "project:<my-category>", where <my-category> is an agreed upon value set by you). The use of categories are only relevant when processing _multiple_ API assets.

_Tags_ are used to support _delta_ handling of Exchange assets and API instances. Since we do not want to sync _all_ targeted assets upon each execution, we need a way to sync only _modified_ assets.  This is where _tags_ come in. Once an asset is synced, the asset is _tagged_ with the current execution timestamp. The next time the app is executed, the asset's _modifiedAt_ timestamp is compared to the asset tag timestamp to determine if the asset should be included in the sync.

For more information on [categories](https://docs.mulesoft.com/exchange/to-manage-categories) and [tags](https://docs.mulesoft.com/exchange/to-describe-an-asset#add-asset-tags), review the MuleSoft documentation site.

### Exchange assets
Exchange REST API assets (e.g. REST API specifications based on RAML or OAS) are created and maintained in Anypoint Exchange to support API discovery and consumption. These Exchange assets are identified based on category (for syncing multiple API assets or by the combination of organization name, asset ID, and API version. 

#### Asset Pages
When looking at an Exchange REST API asset, the asset includes asset portal content and pages that are not technically a part of the REST API asset. These must be accounted for and handled separately with additional API calls.

### API instances
An API _version_ can have one or more API _instances_. An API instance is created based on an Exchange API asset, so when looking at an Exchange asset, you can see all its associated API instances.

#### SLA tiers
Each API instance can potentially have one or more SLA tiers defined within it. If so, these SLA tiers must also be synced along with the API instance.

#### Why do API instances have to be synced?
One of the primary functions of Exchange is to support the discovery and consumption an API. Related to this is the ability to _Request API access_. In order to request API access, the Exchange asset must have at least one API instance associated to it, otherwise no API access request can be made.

### Error Handling
Given that there are API calls embedded within many FOR loops, we do not want to stop the execution if an API call fails or some other issue occurs inside an embedded invocation. In order to support continued execution of the flow, the **On-Error Continue** is the primary error component used throughout the implementation.

If an error occurs, the On-Error Continue component captures the exception, writes the error details to the response message, and writes the error details to the application log.

## Running the app

### Prerequisites
- Business groups: Matching business groups (by name) are expected to exist in the source and target Anypoint environments.
- Environments: Matching environments (by name) are expected to exist within each business group.
- Business group and environment name values: In file "nis-anypoint-exchange-assets-types.raml" within _src/main/resources/api/exchange_modules/284731da-b068-40d0-9261-42c8d4afe01f/nis-anypoint-exchange-assets-types/1.0.1_, set the allowable enum business group names and environment names so that execution doesn't fail RAML validation.
- Connected apps/platform user: Connected apps are required prior app execution.  If the source environment does not support connected apps, a platform user with appropriate permissions should be created instead. 
- Property placeholders: Configure the appropriate connection parameters for the source and target platforms in the environment-specific property (.yaml) file.

### Postman
A Postman collection is provided in the "src/main/resources" folder called "postman_collection.json". This can be used to get jump started on calling the API resources exposed for this API.

### Setup

#### Secure Properties Config
Secure Properties Config components are expected to be used.  There are 3 secure properties components that correspond to 3 properties files:
1. An externalized properties file
2. A local environment-specific properties file
3. A local environment-independent properties file

#### Runtime arguments
The following runtime arguments or environment variables are expected to be set:
- -Denv=local
- -Denv.properties.location=/opt/mule/conf/properties  (This is the folder location path that contains the externalized properties file. An example is provided in "src/main/resources" called "external-local-prop-file-example.properties")
- -Denv.keystore.location=/opt/mule/conf/security  (Location of the keystore if enabling HTTPS)
- -Dvault.key=mulesoft12345678  (Symmetric key for the Mule vault for encrypting/decrypting property values)
- -Dapi.id=12345 (This parameter is the API manager API ID and should be set in the Runtime Manager properties tab)
- -Danypoint.platform.gatekeeper=disabled (Used if API Gatekeeper should be disabled for local testing)

## Limitations
1. **Exchange asset (RAML file name)**: Posting with the actual RAML asset name (currently concatenated with asset name if not available). When the source is PCE 2.0.2, the Exchange asset data does NOT include the actual RAML file name (_mainFile_ field) when pulling the asset data. In PCE version 2.1 and above, it is included. Until production PCE moves to at least 2.1.x, the RAML file name is assumed to be the _asset_ name with a _.raml_ extension.  Appropriate guidance should be conveyed to LOBs when creating their API specifications in Design Center to keep the default RAML file name when creating an API specification in Design Center. If the RAML file name is changed (while still using PCE 2.0.2, the posting of the RAML specification to the target system will fail).
2. **API instance**: For API instance updates, the API _instance label_ cannot be **updated** once synced to the target.  The API instance is used to retrieve the API instance ID (from the target) prior to a PATCH call. Therefore, if the target API instance based on the updated instance label cannot be retrieved since it will not exist. If a PATCH is attempted without properly retrieving the API instance ID, a new API instance will result.
3. **API instances**: Setting the API instance "proxy version" (e.g. 2.1.0 or Latest, etc.).  API instance details retrieval does not include the "proxy version" data, so we currently must use a set property value. Currently, this value is set in property "api.manager.proxy.version" in the _app.yaml_ file.
4. **API instance label**: In order for API instances to be updated properly, each API instance MUST set an **instance label**.  This is because using an instance label is the only means to set apart one API instance from another where the Asset ID, API version, and Exchange Asset Version are the same.

## Potential features backlog
1. Allow for full Source and Target connection input data (including encrypted credentials) in the POST payload. This helps during testing so that the source and target Anypoint Platform connection data can be set dynamically from the input POST payload. Since sensitive credential fields are passed (i.e. client secret), the feature should allow for encrypted values for sensitive fields.

   