# Cribl Azure Graph REST Collector Pack
----
## About this Pack

This Pack is designed to collect, process, and output Microsoft Azure Graph data via the Azure Graph REST API. It currently supports the following endpoints:
* [Azure (MSEntra) Signins](https://learn.microsoft.com/en-us/graph/api/signin-list?view=graph-rest-1.0&tabs=http)
* [Azure User Details](https://learn.microsoft.com/en-us/graph/api/user-list?view=graph-rest-1.0&tabs=http)
* [Azure Device Details]()
* [Azure Security Alerts V2](https://learn.microsoft.com/en-us/graph/api/security-list-alerts_v2?view=graph-rest-1.0&tabs=http)
* [Legacy Azure Security Alerts](https://learn.microsoft.com/en-us/graph/api/alert-get?view=graph-rest-1.0&tabs=http) - deprecated and will be [***removed*** April 2026](https://learn.microsoft.com/en-us/graph/api/resources/security-api-overview?view=graph-rest-1.0&viewFallbackFrom=graph-rest-v1.0&preserve-view=true#alerts)

The Pack includes OCSF and Splunk output processing. OCSF data is mapped to the following Classes:
* Azure Signins - [Authentication [3002] Class](https://schema.ocsf.io/1.4.0/classes/authentication)
* Azure User Details - [User Inventory Info [5003] Class](https://schema.ocsf.io/1.4.0/classes/user_inventory)
* Azure Device Details - [Device Inventory Info [5001] Class](https://schema.ocsf.io/1.4.0/classes/inventory_info)
* Azure Alerts (all) - [Detection Finding [2004] Class](https://schema.ocsf.io/1.4.0/classes/detection_finding)

Splunk data is mapped to the following sourcetypes - these are the sourcetypes used by the [Splunk Add on for Microsoft Azure](https://splunkbase.splunk.com/app/3757):
* Azure Signins: `sourcetype=azure:aad:signin`
* Azure Users: `sourcetype=azure:aad:user`
* Azure Device Details:`sourcetype=azure:aad:device`
* Legacy Azure Alerts: `sourcetype=ms:graph:security:alerts`
* Azure Alerts V2: `sourcetype=ms:graph:security:alerts:v2`


## Deployment
After installing the Pack, you must perform the following:

* Add the Event Breaker Ruleset included in Appendix A below to your Stream instance under Processing > Knowledge > Event Breaker Rules. You should perform a Commit/Deploy before proceeding.
* Obtain a ```Tenant ID```, ```Client ID``` and ```Client Secret``` from your Azure/MS Entra Administrator. These credentials must have the following permissions in order for all the Collectors to work:
  * AuditLog.Read.All
  * Device.Read.All
  * Policy.Read.All
  * SecurityAlert.Read.All
  * SecurityEvents.Read.All
  * User.Read.All

* Add the five Collector Sources in Appendix B to your Stream instance via Data > Sources > Collectors > REST. 
* Add the ```Tenant ID```, ```Client ID``` and ```Client Secret``` when prompted.
* Perform a Run > Preview to verify that each Collector works correctly.
* Schedule each Collector and enable State Tracking (with default configuration) for the Signins and Alerts Collectors *only*. Suggested schedules for each are:
   *  Signins: Every 5 minutes with default State Tracking
   *  Alerts: Every 5 minutes with default State Tracking
   *  Users: Once/day - this endpoint is configured to retrieve all available Users so keep that in mind.
   *  Devices: Once/day - this endpoint is configured to retrieve all available Devices so keep that in mind.
* Connect your Azure Rest Collectors to the Pack. On the global Routes page, add a new route, specify a filter expression (if using the default Collector names, than something like ```__inputId.includes('in_azure')``` will work) and choose the ```cribl-azure-graph-rest``` Pack in the Pipeline dropdown.

The following are the in-Pack configurable items - review/update them as needed. 

### Outputs

Each data type can be configured to output data in either OCSF or normalized JSON (Splunk) format. Enable *only one* format for each of the following pipelines:
* ```cribl_azure_graph_signins```
* ```cribl_azure_graph_users```
* ```cribl_azure_graph_devices```
* ```cribl_azure_graph_alerts```
* ```cribl_azure_graph_alerts_v2```

### Variables

The Pack has the following variables:
* `azure_graph_default_splunk_index`: Default index for the Splunk output - defaults to `azure`.

## Release Notes

### Version 0.1.0 - 2025-08-15

External Beta
* Contains the pipelines for processing the five Azure Graph data types
* Supports either OCSF or Splunk output formats

## Contributing to the Pack
To contribute to the Pack, please connect with us on [Cribl Community Slack](https://cribl-community.slack.com/). You can suggest new features or offer to collaborate.

## License
This Pack uses the following license: [Apache 2.0](https://github.com/criblio/appscope/blob/master/LICENSE).


## Appendix A
Consolidated Azure Graph API Event Breaker

```
{
  "minRawLength": 256,
  "id": "Azure Graph API Ruleset",
  "rules": [
    {
      "condition": "_raw.includes('@odata.context') && !_raw.includes('microsoft.graph.security.alert')",
      "type": "json_array",
      "timestampAnchorRegex": "/lastModifiedDateTime/",
      "timestamp": {
        "type": "auto",
        "length": 150
      },
      "timestampTimezone": "local",
      "timestampEarliest": "-420weeks",
      "timestampLatest": "+1week",
      "maxEventBytes": 134217728,
      "disabled": false,
      "parserEnabled": false,
      "shouldUseDataRaw": false,
      "jsonExtractAll": false,
      "eventBreakerRegex": "/[\\n\\r]+(?!\\s)/",
      "name": "Azure Graph API",
      "jsonArrayField": "value"
    },
    {
      "condition": "_raw.includes('@odata.context') && _raw.includes('microsoft.graph.security.alert')",
      "type": "json_array",
      "timestampAnchorRegex": "/lastUpdateDateTime/",
      "timestamp": {
        "type": "auto",
        "length": 150
      },
      "timestampTimezone": "local",
      "timestampEarliest": "-420weeks",
      "timestampLatest": "+1week",
      "maxEventBytes": 134217728,
      "disabled": false,
      "parserEnabled": false,
      "shouldUseDataRaw": false,
      "jsonExtractAll": false,
      "eventBreakerRegex": "/[\\n\\r]+(?!\\s)/",
      "name": "Azure Graph API - Alerts V2",
      "jsonArrayField": "value"
    }
  ],
  "description": "Event breaking rule for the Azure Graph API"
}
```

## Appendix B
Azure Graph Collector Sources JSON

### Azure Graph Signins
```
{
  "type": "collection",
  "ttl": "4h",
  "ignoreGroupJobsLimit": false,
  "removeFields": [],
  "resumeOnBoot": false,
  "schedule": {
    "cronSchedule": "*/5 * * * *",
    "maxConcurrentRuns": 1,
    "skippable": true,
    "run": {
      "rescheduleDroppedTasks": true,
      "maxTaskReschedule": 1,
      "logLevel": "info",
      "jobTimeout": "60m",
      "mode": "run",
      "timeRangeType": "relative",
      "timeWarning": {},
      "expression": "true",
      "minTaskSize": "1MB",
      "maxTaskSize": "10MB",
      "stateTracking": {},
      "timestampTimezone": "UTC"
    },
    "enabled": false
  },
  "streamtags": [],
  "workerAffinity": false,
  "collector": {
    "conf": {
      "discovery": {
        "discoverType": "none",
        "itemList": [],
        "discoverMethod": "get",
        "pagination": {
          "type": "response_body",
          "maxPages": 0,
          "nextRelationAttribute": "next",
          "curRelationAttribute": "@odata.nextLink",
          "attribute": [
            "@odata.nextLink"
          ]
        },
        "enableDiscoverCode": false,
        "discoverUrl": "`https://graph.microsoft.com/v1.0/auditLogs/signIns`",
        "discoverRequestHeaders": [
          {
            "name": "content-type",
            "value": "\"application/x-www-form-urlencoded\""
          }
        ],
        "discoverDataField": "value"
      },
      "collectMethod": "get",
      "pagination": {
        "type": "response_header_link",
        "nextRelationAttribute": "@odata.nextLink",
        "maxPages": 0,
        "attribute": [
          "@odata.nextLink"
        ]
      },
      "authentication": "login",
      "timeout": 0,
      "useRoundRobinDns": false,
      "disableTimeFilter": false,
      "decodeUrl": false,
      "rejectUnauthorized": true,
      "captureHeaders": false,
      "safeHeaders": [],
      "retryRules": {
        "type": "backoff",
        "interval": 1000,
        "limit": 5,
        "multiplier": 2,
        "maxIntervalMs": 20000,
        "codes": [
          429,
          503
        ],
        "enableHeader": true,
        "retryConnectTimeout": false,
        "retryConnectReset": false,
        "retryHeaderName": "retry-after"
      },
      "__scheduling": {
        "stateTracking": {}
      },
      "loginUrl": "`https://login.microsoftonline.com/<Tenant ID| This is your Azure Cloud [Tenant ID](https://learn.microsoft.com/en-us/azure/azure-portal/get-subscription-tenant-id#find-your-microsoft-entra-tenant)>/oauth2/v2.0/token`",
      "loginBody": "`client_secret=${password}&scope=https://graph.microsoft.com/.default&client_id=${username}&grant_type=client_credentials`",
      "authHeaderKey": "Authorization",
      "authHeaderExpr": "`Bearer ${token}`",
      "username": "<Username|>",
      "password": "<Password|>",
      "tokenRespAttribute": "access_token",
      "collectUrl": "`https://graph.microsoft.com/v1.0/auditLogs/signIns?$orderby=createdDateTime&$filter=createdDateTime+ge+`+C.Time.strftime((earliest * 1000 || Date.now()/1000-(5*60)),'%Y-%m-%dT%H:%M:%S.%fZ' )+`+and+createdDateTime+le+`+C.Time.strftime(latest * 1000 || Date.now()/1000,'%Y-%m-%dT%H:%M:%S.%fZ' )",
      "collectRequestParams": [],
      "collectRequestHeaders": [
        {
          "name": "content-type",
          "value": "\"application/x-www-form-urlencoded\""
        }
      ]
    },
    "destructive": false,
    "encoding": "utf8",
    "type": "rest"
  },
  "input": {
    "type": "collection",
    "staleChannelFlushMs": 10000,
    "sendToRoutes": true,
    "preprocess": {
      "disabled": true
    },
    "throttleRatePerSec": "0",
    "breakerRulesets": [
      "Azure Graph API Ruleset"
    ],
    "metadata": []
  },
  "savedState": {},
  "id": "in_azure_graph_signins"
}
```
### Azure Graph Users
```
{
  "type": "collection",
  "ttl": "4h",
  "ignoreGroupJobsLimit": false,
  "removeFields": [],
  "resumeOnBoot": false,
  "schedule": {
    "cronSchedule": "0 0 * * *",
    "maxConcurrentRuns": 1,
    "skippable": true,
    "run": {
      "rescheduleDroppedTasks": true,
      "maxTaskReschedule": 1,
      "logLevel": "info",
      "jobTimeout": "60m",
      "mode": "run",
      "timeRangeType": "relative",
      "timeWarning": {},
      "expression": "true",
      "minTaskSize": "1MB",
      "maxTaskSize": "10MB",
      "stateTracking": {}
    },
    "enabled": false
  },
  "streamtags": [],
  "workerAffinity": false,
  "collector": {
    "conf": {
      "discovery": {
        "discoverType": "none",
        "discoverMethod": "get",
        "pagination": {
          "type": "response_body",
          "maxPages": 0,
          "attribute": [
            "@odata.nextLink"
          ]
        },
        "enableDiscoverCode": false,
        "discoverUrl": "`https://graph.microsoft.com/v1.0/users?$select=id`",
        "discoverRequestHeaders": [
          {
            "name": "content-type",
            "value": "\"application/x-www-form-urlencoded\""
          }
        ],
        "discoverDataField": "value"
      },
      "collectMethod": "get",
      "pagination": {
        "type": "response_header_link",
        "nextRelationAttribute": "@odata.nextLink",
        "maxPages": 0
      },
      "authentication": "login",
      "timeout": 0,
      "useRoundRobinDns": false,
      "disableTimeFilter": false,
      "decodeUrl": false,
      "rejectUnauthorized": true,
      "captureHeaders": false,
      "safeHeaders": [],
      "retryRules": {
        "type": "backoff",
        "interval": 1000,
        "limit": 5,
        "multiplier": 2,
        "maxIntervalMs": 20000,
        "codes": [
          429,
          503
        ],
        "enableHeader": true,
        "retryConnectTimeout": false,
        "retryConnectReset": false,
        "retryHeaderName": "retry-after"
      },
      "__scheduling": {
        "stateTracking": {}
      },
      "loginUrl": "`https://login.microsoftonline.com/<Tenant ID| This is your Azure Cloud [Tenant ID](https://learn.microsoft.com/en-us/azure/azure-portal/get-subscription-tenant-id#find-your-microsoft-entra-tenant)>/oauth2/v2.0/token`",
      "loginBody": "`client_secret=${password}&scope=https://graph.microsoft.com/.default&client_id=${username}&grant_type=client_credentials`",
      "authHeaderKey": "Authorization",
      "authHeaderExpr": "`Bearer ${token}`",
      "collectUrl": "`https://graph.microsoft.com/v1.0/users/`",
      "collectRequestHeaders": [
        {
          "name": "content-type",
          "value": "\"application/x-www-form-urlencoded\""
        }
      ],
      "username": "<Username|>",
      "password": "<Password|>",
      "tokenRespAttribute": "access_token"
    },
    "destructive": false,
    "encoding": "utf8",
    "type": "rest"
  },
  "input": {
    "type": "collection",
    "staleChannelFlushMs": 10000,
    "sendToRoutes": true,
    "preprocess": {
      "disabled": true
    },
    "throttleRatePerSec": "0",
    "metadata": [],
    "breakerRulesets": [
      "Azure Graph API Ruleset"
    ]
  },
  "savedState": {},
  "id": "in_azure_graph_users"
}
```
### Azure Graph Devices
```
{
  "type": "collection",
  "ttl": "4h",
  "ignoreGroupJobsLimit": false,
  "removeFields": [],
  "resumeOnBoot": false,
  "schedule": {},
  "streamtags": [],
  "workerAffinity": false,
  "collector": {
    "conf": {
      "discovery": {
        "discoverType": "none"
      },
      "collectMethod": "get",
      "pagination": {
        "type": "response_header_link",
        "nextRelationAttribute": "@odata.nextLink",
        "maxPages": 0
      },
      "authentication": "login",
      "timeout": 0,
      "useRoundRobinDns": false,
      "disableTimeFilter": false,
      "decodeUrl": true,
      "rejectUnauthorized": false,
      "captureHeaders": false,
      "safeHeaders": [],
      "retryRules": {
        "type": "backoff",
        "interval": 1000,
        "limit": 5,
        "multiplier": 2,
        "maxIntervalMs": 20000,
        "codes": [
          429,
          503
        ],
        "enableHeader": true,
        "retryConnectTimeout": false,
        "retryConnectReset": false,
        "retryHeaderName": "retry-after"
      },
      "__scheduling": {
        "stateTracking": {}
      },
      "loginUrl": "'https://login.microsoftonline.com/<Tenant ID| This is your Azure Cloud [Tenant ID](https://learn.microsoft.com/en-us/azure/azure-portal/get-subscription-tenant-id#find-your-microsoft-entra-tenant)>/oauth2/v2.0/token'",
      "loginBody": "`client_secret=${password}&scope=https://graph.microsoft.com/.default&client_id=${username}&grant_type=client_credentials`",
      "authHeaderKey": "Authorization",
      "authHeaderExpr": "`Bearer ${token}`",
      "tokenRespAttribute": "access_token",
      "collectUrl": "'https://graph.microsoft.com/v1.0/devices'",
      "collectRequestParams": [],
      "collectRequestHeaders": [
        {
          "name": "content-type",
          "value": "'application/x-www-form-urlencoded'"
        }
      ],
      "username": "<Username|>",
      "password": "<Password|>"
    },
    "destructive": false,
    "type": "rest"
  },
  "input": {
    "type": "collection",
    "staleChannelFlushMs": 10000,
    "sendToRoutes": true,
    "preprocess": {
      "disabled": true
    },
    "throttleRatePerSec": "0",
    "breakerRulesets": [
      "Azure Graph API Ruleset"
    ]
  },
  "savedState": {},
  "id": "in_azure_graph_devices"
}
```
### Azure Graph Alerts (Legacy)
```
{
  "type": "collection",
  "ttl": "4h",
  "ignoreGroupJobsLimit": false,
  "removeFields": [],
  "resumeOnBoot": false,
  "schedule": {
    "run": {
      "mode": "run"
    }
  },
  "streamtags": [],
  "workerAffinity": false,
  "collector": {
    "conf": {
      "discovery": {
        "discoverType": "none"
      },
      "collectMethod": "get",
      "pagination": {
        "type": "response_header_link",
        "nextRelationAttribute": "@odata.nextLink",
        "maxPages": 0
      },
      "authentication": "login",
      "timeout": 0,
      "useRoundRobinDns": false,
      "disableTimeFilter": false,
      "decodeUrl": true,
      "rejectUnauthorized": false,
      "captureHeaders": false,
      "safeHeaders": [],
      "retryRules": {
        "type": "backoff",
        "interval": 1000,
        "limit": 5,
        "multiplier": 2,
        "maxIntervalMs": 20000,
        "codes": [
          429,
          503
        ],
        "enableHeader": true,
        "retryConnectTimeout": false,
        "retryConnectReset": false,
        "retryHeaderName": "retry-after"
      },
      "__scheduling": {
        "stateTracking": {}
      },
      "loginUrl": "'https://login.microsoftonline.com/<Tenant ID| This is your Azure Cloud [Tenant ID](https://learn.microsoft.com/en-us/azure/azure-portal/get-subscription-tenant-id#find-your-microsoft-entra-tenant)>/oauth2/v2.0/token'",
      "loginBody": "`client_secret=${password}&scope=https://graph.microsoft.com/.default&client_id=${username}&grant_type=client_credentials`",
      "authHeaderKey": "Authorization",
      "authHeaderExpr": "`Bearer ${token}`",
      "tokenRespAttribute": "access_token",
      "collectUrl": "'https://graph.microsoft.com/v1.0/security/alerts'",
      "collectRequestParams": [
        {
          "name": "filter",
          "value": "`createdDateTime+ge+'${earliest ? C.Time.strftime(earliest,'%Y-%m-%dT%H:%M:%S.%fZ') : C.Time.strftime(Date.now()/1000 - 5*60,'%Y-%m-%dT%H:%M:%S.%fZ')}'+and+createdDateTime+le+'${earliest ? C.Time.strftime(earliest,'%Y-%m-%dT%H:%M:%S.%fZ') : C.Time.strftime(Date.now()/1000,'%Y-%m-%dT%H:%M:%S.%fZ')}`"
        }
      ],
      "collectRequestHeaders": [
        {
          "name": "content-type",
          "value": "'application/x-www-form-urlencoded'"
        }
      ],
      "username": "<Username|>",
      "password": "<Password|>"
    },
    "destructive": false,
    "type": "rest"
  },
  "input": {
    "type": "collection",
    "staleChannelFlushMs": 10000,
    "sendToRoutes": true,
    "preprocess": {
      "disabled": true
    },
    "throttleRatePerSec": "0",
    "breakerRulesets": [
      "Azure Graph API Ruleset"
    ]
  },
  "savedState": {},
  "id": "in_azure_graph_security_alerts"
}
```
### Azure Graph Alerts V2
```
{
  "type": "collection",
  "ttl": "4h",
  "ignoreGroupJobsLimit": false,
  "removeFields": [],
  "resumeOnBoot": false,
  "schedule": {
    "cronSchedule": "*/5 * * * *",
    "maxConcurrentRuns": 1,
    "skippable": true,
    "run": {
      "rescheduleDroppedTasks": true,
      "maxTaskReschedule": 1,
      "logLevel": "info",
      "jobTimeout": "60m",
      "mode": "run",
      "timeRangeType": "relative",
      "timeWarning": {},
      "expression": "true",
      "minTaskSize": "1MB",
      "maxTaskSize": "10MB",
      "stateTracking": {}
    },
    "enabled": false
  },
  "streamtags": [],
  "workerAffinity": false,
  "collector": {
    "conf": {
      "discovery": {
        "discoverType": "none"
      },
      "collectMethod": "get",
      "pagination": {
        "type": "none"
      },
      "authentication": "login",
      "timeout": 0,
      "useRoundRobinDns": false,
      "disableTimeFilter": false,
      "decodeUrl": false,
      "rejectUnauthorized": true,
      "captureHeaders": false,
      "safeHeaders": [],
      "retryRules": {
        "type": "backoff",
        "interval": 1000,
        "limit": 5,
        "multiplier": 2,
        "maxIntervalMs": 20000,
        "codes": [
          429,
          503
        ],
        "enableHeader": true,
        "retryConnectTimeout": false,
        "retryConnectReset": false,
        "retryHeaderName": "retry-after"
      },
      "__scheduling": {
        "stateTracking": {}
      },
      "loginUrl": "'https://login.microsoftonline.com/<Tenant ID| This is your Azure Cloud [Tenant ID](https://learn.microsoft.com/en-us/azure/azure-portal/get-subscription-tenant-id#find-your-microsoft-entra-tenant)>/oauth2/v2.0/token'",
      "loginBody": "`client_secret=${password}&scope=https://graph.microsoft.com/.default&client_id=${username}&grant_type=client_credentials`",
      "authHeaderKey": "Authorization",
      "authHeaderExpr": "`Bearer ${token}`",
      "collectUrl": "'https://graph.microsoft.com/v1.0/security/alerts_v2'",
      "collectRequestHeaders": [
        {
          "name": "content-type",
          "value": "'application/x-www-form-urlencoded'"
        }
      ],
      "username": "<Username|>",
      "password": "<Password|>",
      "tokenRespAttribute": "access_token",
      "collectRequestParams": [
        {
          "name": "filter",
          "value": "`createdDateTime ge ${earliest ? C.Time.strftime(earliest,'%Y-%m-%dT%H:%M:%S.%fZ') : C.Time.strftime(Date.now()/1000 - 5*60,'%Y-%m-%dT%H:%M:%S.%fZ')} and createdDateTime le ${earliest ? C.Time.strftime(earliest,'%Y-%m-%dT%H:%M:%S.%fZ') : C.Time.strftime(Date.now()/1000,'%Y-%m-%dT%H:%M:%S.%fZ')}`"
        }
      ]
    },
    "destructive": false,
    "encoding": "utf8",
    "type": "rest"
  },
  "input": {
    "type": "collection",
    "staleChannelFlushMs": 10000,
    "sendToRoutes": true,
    "preprocess": {
      "disabled": true
    },
    "throttleRatePerSec": "0",
    "breakerRulesets": [
      "Azure Graph API Ruleset"
    ],
    "metadata": []
  },
  "savedState": {},
  "id": "in_azure_graph_security_alerts_v2"
}
```