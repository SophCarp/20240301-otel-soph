***OtelWalkthrough***
Copied over 3/29/2024
Sophia Carpenter

**Notes**
- This is for internal testing.
- CLI is out
- Public documentations: https://learn.microsoft.com/en-us/azure/container-apps/opentelemetry-agents?tabs=arm 
- You should be able to make OTEL in any region. 


-------------------------------------------------------------------------


**Start up app**
1. Set up variables - these are Sophia's specific variables she used
```bash
RESOURCE_GROUP="otel-stage-rg"
APP_NAME="webapp"
ENVIRONMENT="otel-stage-env"
LOCATION="northcentralusstage"
```
2. Deploy RG 
```bash
az group create -n $RESOURCE_GROUP -l eastus
```
2. Deploy Environment
```bash
az containerapp env create -n $ENVIRONMENT -g $RESOURCE_GROUP -l $LOCATION --logs-destination none 
```
2. Deploy the app
```bash
az containerapp up -n $APP_NAME -g $RESOURCE_GROUP --environment $ENVIRONMENT --location $LOCATION --ingress external --target-port 8080 --source .
```
3. Get the environment ID
```bash
ENVIRONMENT_ID=`az containerapp show -n $APP_NAME -g $RESOURCE_GROUP --query 'properties.managedEnvironmentId' -o tsv`
```

4. Sanity check: 
  - Get environment resource details. It shouldn't have any otel configurations yet. 
  ```bash
  az rest -m GET -u "$ENVIRONMENT_ID?api-version=2023-11-02-preview" -o jsonc
  ```
  - Visit the app:
  ```bash
  APP_URL=`az containerapp show -n $APP_NAME -g $RESOURCE_GROUP --query 'properties.configuration.ingress.fqdn' -o tsv`"/WeatherForecast"
  ```
  Visit the app - you should see a list of forecasts. Reloading the page should create data that will be sent along the otel agent. 

------------------------------------------------------------------------------------------

**Set up App Insights OTEL connection**

1. Create an App Insights resource
Note: App insights resoure can't be made in stage
```bash
APP_INSIGHTS_NAME="appinsightsotel"
az monitor app-insights component create -a $APP_INSIGHTS_NAME -l eastus -g $RESOURCE_GROUP
```
2. Get App insights Connection string
```bash
AI_CONNECTION=`az monitor app-insights component show -a $APP_INSIGHTS_NAME -g $RESOURCE_GROUP --query "connectionString" -o tsv`
```
3. Update the environment to make a connection with OTEL
Sets up App insights as the destiation, and sets logs and traces to be sent along to app insights
```bash
az containerapp env telemetry app-insights set --name $ENVIRONMENT --resource-group $RESOURCE_GROUP --connection-string $AI_CONNECTION --enable-open-telemetry-logs true --enable-open-telemetry-traces true 
```
4. Sanity Check:
Check that it worked
```bash
az containerapp env telemetry app-insights show --name $ENVIRONMENT --resource-group $RESOURCE_GROUP
```
You should get back: 
```json
{
  "appInsightsConfiguration": {
    "connectionString": "Configured",
    "enable-open-telemetry-logs": true,
    "enable-open-telemetry-traces": true
  }
}
```
5. (Optional) Clear out App Insights settings
```bash
az containerapp env telemetry app-insights delete --name $ENVIRONMENT --resource-group $RESOURCE_GROUP
```

------------------------------------------------------------------------------------------------

**Setting Up DataDog**
1. Get datadog details
In Datadog agent manager- go to settings, and collect "site" and "api_key"
```bash
DD_SITE="us3.datadoghq.com"
DD_KEY="<YOUR_DD_API_KEY>"
```
2. Set DataDog to recieve OTEL data
This creates DataDog as a possible endpoints, and pipes metrics and traces to it.
```bash
az containerapp env telemetry data-dog set --name $ENVIRONMENT --resource-group $RESOURCE_GROUP --site $DD_SITE --key $DD_KEY --enable-open-telemetry-metrics true --enable-open-telemetry-traces true 
```
3. Sanity Check:
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
4. (Optional) Delete DataDog configurations
```bash
az containerapp env telemetry data-dog delete --name $ENVIRONMENT --resource-group $RESOURCE_GROUP
```

----------------------------------------------------------------------------------------------
**Setting Up HoneyComb**
1. Make a HoneyComb account:
https://www.honeycomb.io/ 
2. Set up a key in Honeycomb: 
  Make an Ingestion API key with name "x-honeycomb-team"
  Copy the API Key ID


3. Set up the variables
```bash
OTLP_1="honeycomb"
OTLP_ENDPOINT_1="api.honeycomb.io:443"
HONEY_API_KEY_ID="<YOUR_HONEYCOMB_KEY_ID>"
OTLP_HEADERS_1="x-honeycomb-team=$HONEY_API_KEY_ID"
```
4. Add honeycomb as an OTEL destination
Adds a honeycomb OTLP endpoint and pipes metrics, traces, and logs to it
```bash
az containerapp env telemetry otlp add --name $ENVIRONMENT --resource-group $RESOURCE_GROUP --endpoint $OTLP_ENDPOINT_1 --otlp-name $OTLP_1 --insecure false --headers $OTLP_HEADERS_1 --enable-open-telemetry-metrics true --enable-open-telemetry-traces true --enable-open-telemetry-logs true
```
5. Sanity Check: 
Check out one specific otlp endpoint details: 
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

6. Sanity Check:
Check all the otlp stuff: 
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
6. (Optional) Delete Otlp details
```bash
az containerapp env telemetry otlp remove --name $ENVIRONMENT --resource-group $RESOURCE_GROUP --otlp-name $OTLP_1
```
------------------------------------------------------------------------------------------------------------------
And you're done!



------------------------------
***Archive***
Alternative/reference commands or other notes
-----------------------------------------
**Quick Reference Commands**
For quick reference - all the variables and such in one space

- All the simple string variables:
```bash
RESOURCE_GROUP="otel-stage-rg"
APP_NAME="webapp"
ENVIRONMENT="otel-stage-env"
LOCATION="northcentralusstage"

APP_INSIGHTS_NAME="appinsightsotel-stage"

SITE="us3.datadoghq.com"
DD_KEY="<YOUR_DD_API_KEY>"
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
- Delete all OTEL settings
```bash
az rest -m PATCH -u "$ENVIRONMENT_ID?api-version=2023-11-02-preview" --body @env-enable-otel-blank.json
```
-------------------------------------------------
