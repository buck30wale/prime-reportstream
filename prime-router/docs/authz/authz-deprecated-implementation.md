# Authorization Deep Dive

## Current Design

To authorize a user for an endpoint, we compare the scopes that are present on the access token versus what scope is 
required.

### Scope Design

- A client can have multiple scopes
- Scopes are in the format of `{organization}.{senderOrReceiver}.{role}` 
- `organization` and `senderOrReceiver` can be omitted with a wildcard `*` or `default`

There are four possible roles. 

| Role       | Definition                                                                |
|------------|---------------------------------------------------------------------------|
| primeadmin | system administrator with full access to all endpoints                    |
| admin      | administrator with access to endpoints that manage their own organization |
| user       | user with read access to endpoints scoped to their organization           |
| report     | able to submit a report as an organization                                |


Here are some examples given an organization of md-phd (Maryland Public Health Department)

| Scope                   | Access                                                                 |
|-------------------------|------------------------------------------------------------------------|
| `*.*.primeadmin`        | System Administrator                                                   |
| `md-phd.*.admin`        | Organization Administrator for md-phd                                  |
| `md-phd.*.user`         | Organization User for md-phd                                           |
| `md-phd.*.report`       | Organization Report Submitter for md-phd                               |
| `md-phd.default.admin`  | Organization Administrator for md-phd                                  |
| `md-phd.default.user`   | Organization User for md-phd                                           |
| `md-phd.default.report` | Organization Report Submitter for md-phd                               |
| `md-phd.abc.admin`      | Organization Administrator for md-phd scoped to abc sender or receiver |
| `md-phd.abc.user`       | Organization User for md-phd scoped to abc sender or receiver          |



We currently have three methods of authenticating and authorizing a user or machine for accessing 
ReportStream endpoints.

### Server2Server

This is ReportStream code that handles the client credentials OAuth 2.0 flow. This is most often used in machine-to-machine 
communication such as senders submitting reports. The important bit for authorization happens based on the scope requested 
in the call to the `/api/token` endpoint. A member of the engagement team sets up an organization's public key under a 
specific scope in the organization settings. When a request for an access token comes in with a requested scope, we check 
the JWT assertion was signed with the public key under that scope in settings.

If the request is successful, an access token is granted with a scope matching the pattern above.

### Okta

Users are all set up under our Okta instance. We add users to appropriate groups in Okta that correspond to what they 
should be allowed to access. Those groups are in a custom string array claim called `organization` in the access token 
which we then map to scopes that our system can handle. Scopes internal to okta are ignored.

Here is the Okta group to scope mapping strategy given an organization of md-phd (Maryland Public Health Department)

| Okta Group              | ReportStream Scope |
|-------------------------|--------------------|
| `DHPrimeAdmins`         | `*.*.primeadmin`   |
| `DHmd-phdAdmins`        | `md-phd.*.admin`   |
| `DHSender_md-phdAdmins` | `md-phd.*.admin`   |
| `DHSender_md-phd`       | `md-phd.*.user`    |
| `DHmd-phd`              | `md-phd.*.user`    |
* Note: There is no group that maps to the `report` role


### Azure Function Keys

This is the default authentication process built into the Azure functions library. It is a simple shared secret that is 
stored in Azure and shared with the client. It is still used on some endpoints for its simplicity of setup. It includes
no authorization check at all and should be deprecated for that reason.

### Current system pros
- Flexible for onboarding senders/receivers to allow them to use the system they prefer
- The scopes are fine-grained allowing us to be specific about what resources a client should be able to access

### Current system cons
- x-function-key authentication is almost a backdoor into our system given that it does not do authorization
- Scopes are difficult to understand and are often tailored to a single client rather than generic permissions
- Authorization code is spread widely across the codebase with a lot of duplication
- Okta group to scope mapping is clunky and prone to errors based on group naming in the Okta admin portal
- Difficult to keep track of which client is using what authentication system


## Authorization requirements by endpoint

| Location                                                                                  | Function Name               | Verb             | URL                                                                     | Auth Strategy                                                                              | Role requirements                                                | Restrictions                     |
|-------------------------------------------------------------------------------------------|-----------------------------|------------------|-------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|------------------------------------------------------------------|----------------------------------|
| [SenderFilesFunction](../../src/main/kotlin/azure/SenderFilesFunction.kt)                 | getSenderFiles              | GET              | /api/sender-files                                                       | [OktaAuthentication](../../src/main/kotlin/tokens/OktaAuthentication.kt)                   | System Admin                                                     | None                             |
| [AdminApiFunctions](../../src/main/kotlin/azure/AdminApiFunctions.kt)                     | getSendFailures             | GET              | /api/adm/getsendfailures                                                | [OktaAuthentication](../../src/main/kotlin/tokens/OktaAuthentication.kt)                   | System Admin                                                     | None                             |
| [AdminApiFunctions](../../src/main/kotlin/azure/AdminApiFunctions.kt)                     | listreceiversconnstatus     | GET              | /api/adm/listreceiversconnstatus                                        | [OktaAuthentication](../../src/main/kotlin/tokens/OktaAuthentication.kt)                   | System Admin                                                     | None                             |
| [AdminApiFunctions](../../src/main/kotlin/azure/AdminApiFunctions.kt)                     | getresend                   | GET              | /api/adm/getresend                                                      | [OktaAuthentication](../../src/main/kotlin/tokens/OktaAuthentication.kt)                   | System Admin                                                     | None                             |
| [ApiKeysFunctions](../../src/main/kotlin/azure/ApiKeysFunctions.kt)                       | getApiKeys                  | GET              | /api/settings/organizations/{organizationName}/public-keys              | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, or org user                             | Organization                     |
| [ApiKeysFunctions](../../src/main/kotlin/azure/ApiKeysFunctions.kt)                       | getApiKeysV1                | GET              | /api/v1/settings/organizations/{organizationName}/public-keys           | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, or org user                             | Organization                     |
| [ApiKeysFunctions](../../src/main/kotlin/azure/ApiKeysFunctions.kt)                       | postApiKey                  | POST             | /api/settings/organizations/{organizationName}/public-keys              | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin or org admin                                        | Organization                     |
| [ApiKeysFunctions](../../src/main/kotlin/azure/ApiKeysFunctions.kt)                       | deleteApiKey                | DELETE           | settings/organizations/{organizationName}/public-keys/{scope}/{kid}     | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin or org admin                                        | Organization                     |
| [CheckFunction](../../src/main/kotlin/azure/CheckFunction.kt)                             | check                       | GET,POST         | /api/check                                                              | x-functions-key                                                                            | None                                                             | None                             |
| [CheckFunction](../../src/main/kotlin/azure/CheckFunction.kt)                             | checkreceiver               | POST             | /api/checkreceiver/org/{orgName}/receiver/{receiverName}                | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin or org admin                                        | Organization                     |
| [CovidResultMetadataFunction](../../src/main/kotlin/azure/CovidResultMetaDataFunction.kt) | save-covid-result-metadata  | GET              | /api/saveTestData                                                       | x-functions-key                                                                            | None                                                             | None                             |
| [EmailEngineFunction](../../src/main/kotlin/azure/EmailEngineFunction.kt)                 | createEmailSchedule         | POST             | /api/email-schedule                                                     | EmailEngineFunction.validateUser (Custom Okta)                                             | System Admin                                                     | None                             |
| [EmailEngineFunction](../../src/main/kotlin/azure/EmailEngineFunction.kt)                 | deleteEmailSchedule         | DELETE           | /api/email-schedule/{scheduleId}                                        | EmailEngineFunction.validateUser (Custom Okta)                                             | System Admin                                                     | None                             |
| [EmailSenderFunction](../../src/main/kotlin/azure/EmailSenderFunction.kt)                 | emailRegisteredOrganization | POST             | /api/email-registered                                                   | None                                                                                       | None                                                             | None                             |
| [SettingsFunctions](../../src/main/kotlin/azure/SettingsFunctions.kt)                     | getOrganizations            | GET,HEAD         | /api/settings/organizations                                             | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | HEAD: System Admin<br/>GET: System Admin, org admin, or org user | Head: None<br/>GET: Organization |
| [SettingsFunctions](../../src/main/kotlin/azure/SettingsFunctions.kt)                     | getOneOrganization          | GET              | /api/settings/organizations/{organizationName}                          | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, or org user                             | Organization                     |
| [SettingsFunctions](../../src/main/kotlin/azure/SettingsFunctions.kt)                     | getSenders                  | GET              | /api/settings/organizations/{organizationName}/senders                  | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, or org user                             | Organization                     |
| [SettingsFunctions](../../src/main/kotlin/azure/SettingsFunctions.kt)                     | updateOneOrganization       | DELETE,PUT       | /api/settings/organizations/{organizationName}                          | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin                                                     | None                             |
| [SettingsFunctions](../../src/main/kotlin/azure/SettingsFunctions.kt)                     | getOneSender                | GET              | /api/settings/organizations/{organizationName}/senders/{senderName}     | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, or org user                             | Organization                     |
| [SettingsFunctions](../../src/main/kotlin/azure/SettingsFunctions.kt)                     | updateOneSender             | DELETE,PUT       | /api/settings/organizations/{organizationName}/senders/{senderName}     | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin                                                     | None                             |
| [SettingsFunctions](../../src/main/kotlin/azure/SettingsFunctions.kt)                     | getReceivers                | GET              | /api/settings/organizations/{organizationName}/receivers                | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, or org user                             | Organization                     |
| [SettingsFunctions](../../src/main/kotlin/azure/SettingsFunctions.kt)                     | getOneReceiver              | GET              | /api/settings/organizations/{organizationName}/receivers/{receiverName} | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, or org user                             | Organization                     |
| [SettingsFunctions](../../src/main/kotlin/azure/SettingsFunctions.kt)                     | updateOneReceiver           | DELETE,PUT       | /api/settings/organizations/{organizationName}/receivers/{receiverName} | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin                                                     | None                             |
| [SettingsFunctions](../../src/main/kotlin/azure/SettingsFunctions.kt)                     | getSettingRevisionHistory   | GET              | /api/waters/org/{organizationName}/settings/revs/{settingSelector}      | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, or org user                             | Organization                     |
| [RequeueFunction](../../src/main/kotlin/azure/RequeueFunction.kt)                         | requeue                     | POST             | /api/requeue/send                                                       | x-functions-key                                                                            | None                                                             | None                             |
| [RequeueFunction](../../src/main/kotlin/azure/RequeueFunction.kt)                         | doResendFunction            | POST             | /api/adm/resend                                                         | [OktaAuthentication](../../src/main/kotlin/tokens/OktaAuthentication.kt)                   | System Admin                                                     | None                             |
| [TokenFunction](../../src/main/kotlin/azure/TokenFunction.kt)                             | token                       | POST             | /api/token                                                              | [Server2ServerAuthentication](../../src/main/kotlin/tokens/Server2ServerAuthentication.kt) | Requires properly signed client assertion JWT                    | None                             |
| [HistoryFunctions](../../src/main/kotlin/azure/HistoryFunctions.kt)                       | getReports                  | GET,HEAD,OPTIONS | /api/history/report                                                     | HistoryFunctions.checkAuthenticated (Custom)                                               | System Admin or org user (no org Admin)                          | Organization                     |
| [HistoryFunctions](../../src/main/kotlin/azure/HistoryFunctions.kt)                       | searchReports               | POST             | /api/v1/reports/search                                                  | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin                                                     | None                             |
| [HistoryFunctions](../../src/main/kotlin/azure/HistoryFunctions.kt)                       | getReportById               | GET              | /api/history/report/{reportId}                                          | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, org user, or report                     | Organization                     |
| [HistoryFunctions](../../src/main/kotlin/azure/HistoryFunctions.kt)                       | getFacilitiesByReportId     | GET              | /api/history/report/{reportId}/facilities                               | HistoryFunctions.checkAuthenticated (Custom)                                               | System Admin or org user (no org Admin)                          | Organization                     |
| [LookupTableFunctions](../../src/main/kotlin/azure/LookupTableFunctions.kt)               | getLookupTableList          | GET,HEAD         | /api/lookuptables/list                                                  | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | None                                                             | None                             |
| [LookupTableFunctions](../../src/main/kotlin/azure/LookupTableFunctions.kt)               | getLookupTableData          | GET              | /api/lookuptables/{tableName}/{tableVersion}/content                    | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | None                                                             | None                             |
| [LookupTableFunctions](../../src/main/kotlin/azure/LookupTableFunctions.kt)               | getActiveLookupTableData    | GET              | /api/lookuptables/{tableName}/content                                   | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | None                                                             | None                             |
| [LookupTableFunctions](../../src/main/kotlin/azure/LookupTableFunctions.kt)               | getLookupTableInfo          | GET              | /api/lookuptables/{tableName}/{tableVersion}/info                       | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | None                                                             | None                             |
| [LookupTableFunctions](../../src/main/kotlin/azure/LookupTableFunctions.kt)               | createLookupTable           | POST             | /api/lookuptables/{tableName}                                           | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin                                                     | None                             |
| [LookupTableFunctions](../../src/main/kotlin/azure/LookupTableFunctions.kt)               | activateLookupTable         | PUT              | /api/lookuptables/{tableName}/{tableVersion}/activate                   | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin                                                     | None                             |
| [MessagesFunctions](../../src/main/kotlin/azure/MessagesFunctions.kt)                     | messageSearch               | GET              | /api/messages                                                           | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin                                                     | None                             |
| [MessagesFunctions](../../src/main/kotlin/azure/MessagesFunctions.kt)                     | messageDetails              | GET              | /api/message/{id}                                                       | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin                                                     | None                             |
| [MetaDataFunction](../../src/main/kotlin/azure/MetaDataFunction.kt)                       | getLivdData                 | GET              | /api/metadata/livd                                                      | None                                                                                       | None                                                             | None                             |
| [ReportFunction](../../src/main/kotlin/azure/ReportFunction.kt)                           | reports                     | POST             | /api/reports                                                            | x-functions-key                                                                            | None                                                             | None                             |
| [ReportFunction](../../src/main/kotlin/azure/ReportFunction.kt)                           | getMessagesFromTestBank     | POST             | /api/reports/testing                                                    | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin                                                     | None                             |
| [ReportFunction](../../src/main/kotlin/azure/ReportFunction.kt)                           | processFhirDataRequest      | POST             | /api/reports/testing/test                                               | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin                                                     | None                             |
| [ReportFunction](../../src/main/kotlin/azure/ReportFunction.kt)                           | downloadReport              | GET              | /api/reports/download                                                   | x-functions-key                                                                            | None                                                             | None                             |
| [ReportFunction](../../src/main/kotlin/azure/ReportFunction.kt)                           | waters                      | POST             | /api/waters                                                             | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, org user, or report                     | Organization and Sender          |
| [ValidateFunction](../../src/main/kotlin/azure/ValidateFunction.kt)                       | validate                    | POST             | /api/validate                                                           | None                                                                                       | None                                                             | None                             |
| [DeliveryFunction](../../src/main/kotlin/history/azure/DeliveryFunction.kt)               | getDeliveriesV1             | POST             | /api/v1/receivers/{receiverName}/deliveries                             | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, org user, or report                     | Organization                     |
| [DeliveryFunction](../../src/main/kotlin/history/azure/DeliveryFunction.kt)               | getDeliveriesHistory        | POST             | /api/v1/waters/org/{organization}/deliveries                            | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, org user, or report                     | Organization                     |
| [DeliveryFunction](../../src/main/kotlin/history/azure/DeliveryFunction.kt)               | getDeliveries               | GET              | /api/waters/org/{organization}/deliveries                               | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, org user, or report                     | Organization                     |
| [DeliveryFunction](../../src/main/kotlin/history/azure/DeliveryFunction.kt)               | getDeliveryDetails          | GET              | /api/waters/report/{id}/delivery                                        | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, org user, or report                     | Organization                     |
| [DeliveryFunction](../../src/main/kotlin/history/azure/DeliveryFunction.kt)               | getEtorMetadataForDelivery  | GET              | /api/waters/report/{reportId}/delivery/etorMetadata                     | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, org user, or report                     | Organization                     |
| [DeliveryFunction](../../src/main/kotlin/history/azure/DeliveryFunction.kt)               | getDeliveryFacilities       | GET              | /api/waters/report/{id}/facilities                                      | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, org user, or report                     | Organization                     |
| [DeliveryFunction](../../src/main/kotlin/history/azure/DeliveryFunction.kt)               | getReportItemsV1            | GET              | /api/v1/report/{reportId}/items                                         | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin                                                     | None                             |
| [DeliveryFunction](../../src/main/kotlin/history/azure/DeliveryFunction.kt)               | getSubmittersV1             | POST             | /api/v1/receivers/{receiverName}/deliveries/submitters/search           | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, org user, or report                     | Organization                     |
| [SubmissionFunction](../../src/main/kotlin/history/azure/SubmissionFunction.kt)           | getOrgSubmissionsList       | GET              | /api/waters/org/{organization}/submissions                              | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, org user, or report                     | Organization                     |
| [SubmissionFunction](../../src/main/kotlin/history/azure/SubmissionFunction.kt)           | getReportDetailedHistory    | GET              | /api/waters/report/{id}/history                                         | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, org user, or report                     | Organization                     |
| [SubmissionFunction](../../src/main/kotlin/history/azure/SubmissionFunction.kt)           | getEtorMetadataForHistory   | GET              | /api/waters/report/{reportId}/history/etorMetadata                      | [AuthenticatedClaims](../../src/main/kotlin/tokens/AuthenticatedClaims.kt)                 | System Admin, org admin, org user, or report                     | Organization                     |

*This is valid as of 12/16/24
