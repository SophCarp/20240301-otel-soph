***OtelWalkthrough***
Copied over 3/11/24
Sophia Carpenter
Before sharing this with folks: Need to remove any keys and IDs that are strings. 

-------------------------------------------------------------------------
**Quick Commands**
- All the simple string variables:
```bash
RESOURCE_GROUP="otel-stage-rg"
APP_NAME="webapp"
ENVIRONMENT="otel-stage-env"
LOCATION="northcentralusstage"
APP_INSIGHTS_NAME="appinsightsotel-stage"
SITE="us3.datadoghq.com"
DD_KEY="fc8d2e785a30993b1ec7f81d1f154323"
```

- All the constructed variables:
```bash
ENVIRONMENT_ID=`az containerapp show -n $APP_NAME -g $RESOURCE_GROUP --query 'properties.managedEnvironmentId' -o tsv`
AI_CONNECTION=`az monitor app-insights component show -a $APP_INSIGHTS_NAME -g $RESOURCE_GROUP --query "connectionString" -o tsv`
```

- Quick! Get... 
```bash
"Environment details:"
az rest -m GET -u "$ENVIRONMENT_ID?api-version=2023-11-02-preview" -o jsonc

"ACA details:"
az containerapp show -n $APP_NAME -g $RESOURCE_GROUP

"ACA Link:"
az containerapp show -n $APP_NAME -g $RESOURCE_GROUP --query 'properties.configuration.ingress.fqdn' -o tsv 
```

- Quick! Update...
```bash
"Code and Deploy:"
az containerapp up -n $APP_NAME -g $RESOURCE_GROUP --environment $ENVIRONMENT --location $LOCATION --source .

```
-------------------------------------------------------------------------------------------

**Start up app**
1. Set up variables - these are Sophia's specific variables she used
```bash
RESOURCE_GROUP="otel-stage-rg"
APP_NAME="webapp"
ENVIRONMENT="otel-stage-env"
LOCATION="northcentralusstage"
```
2. Deploy RG outside of stage
```bash
az group create -n $RESOURCE_GROUP -l eastus
```
2. Deploy Environment
```bash
az containerapp env create -n $ENVIRONMENT -g $RESOURCE_GROUP -l $LOCATION --logs-destination none 
```
2. Deploy the app
```bash
az containerapp up -n $APP_NAME -g $RESOURCE_GROUP --environment $ENVIRONMENT --location $LOCATION --ingress external --source .
```
3. Get the environment ID
```bash
ENVIRONMENT_ID=`az containerapp show -n $APP_NAME -g $RESOURCE_GROUP --query 'properties.managedEnvironmentId' -o tsv`
```

4. Get environment resource - It doesn't have OTEL configured!
```bash
az rest -m GET -u "$ENVIRONMENT_ID?api-version=2023-11-02-preview" -o jsonc
```
------------------------------------------------------------------------------------------

**Connect to App Insights**
1. Create an App Insights resource
```bash
APP_INSIGHTS_NAME="appinsightsotel"
az monitor app-insights component create -a $APP_INSIGHTS_NAME -l eastus -g $RESOURCE_GROUP
```
2. Get Connection string
```bash
AI_CONNECTION=`az monitor app-insights component show -a $APP_INSIGHTS_NAME -g $RESOURCE_GROUP --query "connectionString" -o tsv`
```
3. Update the environment
```bash
az containerapp env telemetry app-insights set --name $ENVIRONMENT --resource-group $RESOURCE_GROUP --connection-string $AI_CONNECTION --enable-open-telemetry-logs true --enable-open-telemetry-traces true 
```
4. Check that it worked
```bash
az containerapp env telemetry app-insights show --name $ENVIRONMENT --resource-group $RESOURCE_GROUP
```
You'll get back: 
```json
{
  "appInsightsConfiguration": {
    "connectionString": "Configured",
    "enable-open-telemetry-logs": true,
    "enable-open-telemetry-traces": true
  }
}
```
5. Clear out App Insights settings
```bash
az containerapp env telemetry app-insights delete --name $ENVIRONMENT --resource-group $RESOURCE_GROUP
```



------------------------------------------------------------------------------------------------

**Setting Up DataDog**
1. Get datadog details
In Datadog agent manager- go to settings, and collect "site" and "api_key"
```bash
DD_SITE="us3.datadoghq.com"
DD_KEY="fc8d2e785a30993b1ec7f81d1f154323"
```
2. Set DataDog configurations
```bash
az containerapp env telemetry data-dog set --name $ENVIRONMENT --resource-group $RESOURCE_GROUP --site $DD_SITE --key $DD_KEY --enable-open-telemetry-metrics true --enable-open-telemetry-traces true 
```
3. Check that it worked: 
```bash
az containerapp env telemetry data-dog show --name $ENVIRONMENT --resource-group $RESOURCE_GROUP
```
You should get back: 
```json
{
  "dataDogConfiguration": {
    "enable-open-telemetry-metrics": true,
    "enable-open-telemetry-traces": true,
    "key": "Configured",
    "site": "us3.datadoghq.com"
  }
}
```
4. Delete DataDog configurations
```bash
az containerapp env telemetry data-dog delete --name $ENVIRONMENT --resource-group $RESOURCE_GROUP
```

----------------------------------------------------------------------------------------------
**Setting Up HoneyComb**
1. Make a HoneyComb account (???)
Did it! 
2. In honeycomb: Make an Ingestion API with name "x-honeycomb-team"

2. set up the variables
```bash
OTLP_1="honeycomb"
OTLP_ENDPOINT_1="api.honeycomb.io:443"
OTLP_HEADERS_1='x-honeycomb-team=hcaik_01hrss2mvszw1efe5yd4t3w6dajz0f0et1wmdxr5crbje3knfzf0nzzsb2'
```
3. Add otlp1 
```bash
az containerapp env telemetry otlp add --name $ENVIRONMENT --resource-group $RESOURCE_GROUP --endpoint $OTLP_ENDPOINT_1 --otlp-name $OTLP_1 --insecure false --headers $OTLP_HEADERS_1 --enable-open-telemetry-metrics true --enable-open-telemetry-traces true 
```
5. Check out one specific otlp endpoint details: 
```bash
az containerapp env telemetry otlp show --name $ENVIRONMENT --resource-group $RESOURCE_GROUP --otlp-name $OTLP_1
```
expected payload: 
```json
{
  "otlpConfiguration": {
    "enable-open-telemetry-logs": false,
    "enable-open-telemetry-metrics": true,
    "enable-open-telemetry-traces": true,
    "endpoint": "api.honeycomb.io:443",
    "headers": [
      {
        "key": "honecombingestapi",
        "value": "Configured"
      }
    ],
    "insecure": false,
    "name": "honeycomb"
  }
}
```

4. Check all the otlp stuff: 
```bash
az containerapp env telemetry otlp list --name $ENVIRONMENT --resource-group $RESOURCE_GROUP 
```
expected payload: 
```json
{
  "otlpConfigurations": [
    {
      "enable-open-telemetry-logs": false,
      "enable-open-telemetry-metrics": true,
      "enable-open-telemetry-traces": true,
      "endpoint": "api.honeycomb.io:443",
      "headers": [
        {
          "key": "honecombingestapi",
          "value": "Configured"
        }
      ],
      "insecure": false,
      "name": "honeycomb"
    }
  ]
}
```
6. Delete Otlp details
```bash
az containerapp env telemetry otlp remove --name $ENVIRONMENT --resource-group $RESOURCE_GROUP --otlp-name $OTLP_1
```



-------
**Clearing Out OTEL details**
To completely clear all OTEL settings from the environment, and then check it
```bash
az rest -m PATCH -u "$ENVIRONMENT_ID?api-version=2023-11-02-preview" --body @env-enable-otel-blank.json
az rest -m GET -u "$ENVIRONMENT_ID?api-version=2023-11-02-preview" -o jsonc
```
-----------------------------------------
**(old) Patch and Get Commands**
- view environment resource
```bash
az rest -m GET -u "$ENVIRONMENT_ID?api-version=2023-11-02-preview" -o jsonc
```

- enable app insights
```bash
az rest -m PATCH -u "$ENVIRONMENT_ID?api-version=2023-11-02-preview" --body @env-enable-otel-ai.json
```

- enable DataDog
```bash
az rest -m PATCH -u "$ENVIRONMENT_ID?api-version=2023-11-02-preview" --body @env-enable-otel-datadog.json
```

- enable honeycomb
```bash
az rest -m PATCH -u "$ENVIRONMENT_ID?api-version=2023-11-02-preview" --body @env-enable-otel-otlp.json
```
-------------------------------------------------





**Setting up Traces**
1. Ask Stacy for help
2. Reference https://github.com/open-telemetry/opentelemetry-dotnet/blob/main/docs/trace/customizing-the-sdk/README.md 
3. Add a "tracer provider" to "WeatherForecastController.cs" 
```cs
var tracerProvider = Sdk.CreateTracerProviderBuilder().Build(); 
```
4. Add "activities" in the code to trace! 


**Questions:** 
Where do the environment variables come in? 
How do I add a metrics section?


How do I see things in DataDog other than the log stream?



**Bugs**
1. App Insights null
I couldn't set appInsightsConfiguration to null. 
```bash
$ az rest -m PATCH -u "$ENVIRONMENT_ID?api-version=2023-11-02-preview" --body @env-enable-otel-datadog.json

Bad Request({"error":{"code":"InvalidRequestParameterWithDetails","message":"AppInsightsConfiguration.ConnectionString is invalid.  AppInsightsConfiguration.ConnectionString can not be set to null during update"}})
```
2. setting DataDog to null: 
File env-enable-otel-blank.json:
```json
{
  "properties": {
    "openTelemetryConfiguration": {
      "destinationsConfiguration":{
        "otlpConfiguration": null,
        "dataDogConfiguration":{
          "site": null,
          "key": null
        }
      },
      "logsConfiguration":null,
      "tracesConfiguration":null,
      "metricsConfiguration": null
    }
  }
}
```
I try to patch it: 
```bash
$ az rest -m PATCH -u "$ENVIRONMENT_ID?api-version=2023-11-02-preview" --body @env-enable-otel-blank.json
```
Payload: 
Bad Request({"error":{"code":"InvalidRequestParameterWithDetails","message":"OpenTelemetryConfiguration.DestinationsConfiguration.DataDogConfiguration.Key is invalid.  OpenTelemetryConfiguration.DestinationsConfiguration.DataDogConfiguration.Key can not be set to null during update"}})

Variations of json: 
```json
{
  "properties": {
    "openTelemetryConfiguration": {
      "destinationsConfiguration":{
        "otlpConfiguration": null,
        "dataDogConfiguration":null
      },
      "logsConfiguration":null,
      "tracesConfiguration":null,
      "metricsConfiguration": null
    }
  }
}
```
Payload: 
Bad Request({"error":{"code":"InvalidRequestParameterWithDetails","message":"OpenTelemetryConfiguration.DestinationsConfiguration.DataDogConfiguration.Key is invalid.  OpenTelemetryConfiguration.DestinationsConfiguration.DataDogConfiguration.Key can not be set to null during update"}}) 

```json
{
  "properties": {
    "openTelemetryConfiguration": {
      "destinationsConfiguration":{
        "otlpConfiguration": null,
        "dataDogConfiguration": {
          "key": "",
          "site": ""
        }
      },
      "logsConfiguration":null,
      "tracesConfiguration":null,
      "metricsConfiguration": null
    }
  }
}
```
Payload
Bad Request({"error":{"code":"InvalidRequestParameterWithDetails","message":"OpenTelemetryConfiguration.DestinationsConfiguration.DataDogConfiguration.Key is invalid.  OpenTelemetryConfiguration.DestinationsConfiguration.DataDogConfiguration.Key can not be set to null during update"}}) 

```json
{
  "properties": {
    "openTelemetryConfiguration": {
      "destinationsConfiguration":{
        "otlpConfiguration": null,
        "dataDogConfiguration": {
          "site": null
        }
      },
      "logsConfiguration":null,
      "tracesConfiguration":null,
      "metricsConfiguration": null
    }
  }
}
```
$ az containerapp env telemetry data-dog show -n $ENVIRONMENT -g $RESOURCE_GROUP
Command group 'containerapp env telemetry data-dog' is in preview and under development. Reference and support levels: https://aka.ms/CLI_refstatus
{
  "dataDogConfiguration": {
    "enable-open-telemetry-metrics": false,
    "enable-open-telemetry-traces": false,
    "key": "Configured",
    "site": "us3.datadoghq.com"
  }
} 

Actions for SOphia: 
1. Download dotnet to computer in terminal


