OtelWalkthrough


**Start up app**
1. Set up variables - these are Sophia's specific variables she used
```bash
RESOURCE_GROUP="otel-michael-rg"
APP_NAME="webapp"
ENVIRONMENT="otel-michael-env"
LOCATION="westus2"
```
```ps
$RESOURCE_GROUP = "otel-michael-rg"
APP_NAME="webapp"
ENVIRONMENT="otel-michael-env"
LOCATION="westus2"
```

2. Deploy the app
```bash
az containerapp up -n $APP_NAME -g $RESOURCE_GROUP --environment $ENVIRONMENT --location $LOCATION --source .
```
3. Get the environment ID
```bash
ENVIRONMENT_ID=`az containerapp show -n $APP_NAME -g $RESOURCE_GROUP --query 'properties.managedEnvironmentId' -o tsv`
```

4. Get environment resource - It doesn't have OTEL configured!
```bash
az rest -m GET -u "$ENVIRONMENT_ID?api-version=2023-11-02-preview" -o jsonc
```


**Connect to App Insights**
1. Create an App Insights resource
```bash
APP_INSIGHTS_NAME="appinsightsotel"
az monitor app-insights component create -a $APP_INSIGHTS_NAME -l $LOCATION -g $RESOURCE_GROUP
```
2. Get Connection string
```bash
CONNECTION=`az monitor app-insights component show -a $APP_INSIGHTS_NAME -g $RESOURCE_GROUP --que
ry "connectionString" -o tsv`
```
3. Add connection string to env-enable-otel-ai.json (probably not the safest method)
4. Update OTEL
```bash
az rest -m PATCH -u "$ENVIRONMENT_ID?api-version=2023-11-02-preview" --body @env-enable-otel-ai.json
```
5. Get environment resource - It has OTel configured to send to app insights!
```bash
az rest -m GET -u "$ENVIRONMENT_ID?api-version=2023-11-02-preview" -o jsonc
```


**Setting Up DataDog**
1. Get datadog details
In Datadog agent manager- go to settings, and collect "site" and "api_key"
2. Insert details into "env-enable-otel-datadog.json"
3. Patch in the new OTEL configuration
```bash
az rest -m PATCH -u "$ENVIRONMENT_ID?api-version=2023-11-02-preview" --body @env-enable-otel-datadog.json
```
4. Check the environment settings to ensure it's correct: 
```bash
az rest -m GET -u "$ENVIRONMENT_ID?api-version=2023-11-02-preview" -o jsonc
```
5. Validate on DataDog that it works
I'm not sure how this looks. But also I think I'm only sending logs? So maybe datadog isn't going to get any data until I send other types of components. 

**Setting Up HoneyComb**
1. Make a HoneyComb account (???)



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



**Bug(?)**
I couldn't set appInsightsConfiguration to null. 
```bash
$ az rest -m PATCH -u "$ENVIRONMENT_ID?api-version=2023-11-02-preview" --body @env-enable-otel-datadog.json

Bad Request({"error":{"code":"InvalidRequestParameterWithDetails","message":"AppInsightsConfiguration.ConnectionString is invalid.  AppInsightsConfiguration.ConnectionString can not be set to null during update"}})

```


Actions for SOphia: 
1. Download dotnet to computer in terminal


