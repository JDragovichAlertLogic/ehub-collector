# Alert Logic Event Collector for Microsoft Azure Event Hubs

[![Build Status](https://secure.travis-ci.org/alertlogic/ehub-collector.png?branch=master)](http://travis-ci.org/alertlogic/ehub-collector)

# Overview

This repository contains the Microsoft Azure web application Node.js source code and Azure Resource Manager (ARM) template required to set up a data collector in Azure to collect and forward events from Event Hubs to Alert Logic.

# Installation

To perform the setup required to grant Alert Logic permission to access Events Hub for event collection, you must have:

* A Microsoft Azure account with administrative privileges
* An Alert Logic user account with administrative privileges


## Register a new Azure web application

In the Azure portal, you must register a new web application. Any application that wants to use the capabilities of Azure Active Directory must first be registered in an Azure AD tenant. This registration process involves giving Azure AD details about your application, such as the URL where it is located, the URL to send replies after a user is authenticated, and the URI that identifies the app.

**To register an Azure web application to collect logs:**

1. Log into the [Azure portal](https://portal.azure.com) as an Active Directory tenant administrator.
1. On the left side panel click **Azure Active Directory**, and then select **App Registrations**.
1. Click **+ New application registration**, and then provide the following configuration parameters: 
    * **Name** - Type a name for the new application (For example `alehubcollector`).
    * **Web app/ API** - Select `Application type`.
    * **Sign-on URL** - Type a URL for the application (for example `http://alehubcollector.com`). 
    
    **Note:** This information is not used anywhere within your subscription.
    
1. Click **Create**.
1. From the **All applications** tab on the App registration (Preview) blade, select **All apps**, and then click the application name you created. 
1. Note the Application ID (such as `a261478c-84fb-42f9-84c2-de050a4babe3`).

## Set up the required Active Directory security permissions

1. On the **Settings** panel for the application, select **Keys**.
1. In the **Description** field, type a key description, and then set **Duration** to **Never expires**. 
1. Click **Save**.

   **Note:** Save the key value, which you need during ARM template deployment.

1. From the Registered App blade, click the link under **Managed application in local directory**, and then click **Properties**.
1. Get the `Service Principal ID` (labeled as "Object ID" on the Properties page) associated with the application.

   **Caution:** Use the Object ID on the Properties page. Do not use Object ID found under the Registered App view or under Settings.

## Create an Alert Logic Access Key
<!--Not sure if AL capitalizes "Access Key," so I'm leaving it as-is until we have a style determination.-->

**From the Bash command line in [Azure Cloud Shell](https://docs.microsoft.com/en-us/azure/cloud-shell/quickstart) run the following commands, where `<username>` is your Alert Logic user name and `<password>` is your Alert Logic password:**

```
export AL_USERNAME='<username>'
auth=$(curl -SX POST -u $AL_USERNAME https://api.global-services.global.alertlogic.com/aims/v1/authenticate); export AL_ACCOUNT_ID=$(echo $auth | jq -r '.authentication.account.id'); export AL_USER_ID=$(echo $auth | jq -r '.authentication.user.id'); export AL_TOKEN=$(echo $auth | jq -r '.authentication.token'); if [ -z $AL_TOKEN ]; then echo "Authentication failure"; else roles=$(curl -SX GET -H "x-aims-auth-token: $AL_TOKEN" https://api.global-services.global.alertlogic.com/aims/v1/$AL_ACCOUNT_ID/users/$AL_USER_ID/roles | jq -r '.roles[].name'); if [ "$roles" != "Administrator" ]; then echo "The $AL_USERNAME doesn’t have Administrator role. Assigned role is '$roles'"; else curl -SX POST -H "x-aims-auth-token: $AL_TOKEN" https://api.global-services.global.alertlogic.com/aims/v1/$AL_ACCOUNT_ID/users/$AL_USER_ID/access_keys | jq .; fi; fi; unset AL_USERNAME;
```

**For accounts with multi-factor authentication (MFA) enabled:**
```
export AL_USERNAME='<username>'
auth=$(curl -SX POST -d '{"mfa_code": "<mfa_code_here>" }' -u $AL_USERNAME https://api.global-services.global.alertlogic.com/aims/v1/authenticate); export AL_ACCOUNT_ID=$(echo $auth | jq -r '.authentication.account.id'); export AL_USER_ID=$(echo $auth | jq -r '.authentication.user.id'); export AL_TOKEN=$(echo $auth | jq -r '.authentication.token'); if [ -z $AL_TOKEN ]; then echo "Authentication failure"; else roles=$(curl -SX GET -H "x-aims-auth-token: $AL_TOKEN" https://api.global-services.global.alertlogic.com/aims/v1/$AL_ACCOUNT_ID/users/$AL_USER_ID/roles | jq -r '.roles[].name'); if [ "$roles" != "Administrator" ]; then echo "The $AL_USERNAME doesn’t have Administrator role. Assigned role is '$roles'"; else curl -SX POST -H "x-aims-auth-token: $AL_TOKEN" https://api.global-services.global.alertlogic.com/aims/v1/$AL_ACCOUNT_ID/users/$AL_USER_ID/access_keys | jq .; fi; fi; unset AL_USERNAME;
```

An example of a successful response is:

```
{
  "access_key_id": "712c0b413eef41f6",
  "secret_key": "1234567890b3eea8880d292fb31aa96902242a076d3d0e320cc036eb51bf25ad"
}
```

**Note:** If the output is blank, verify the Alert Logic user account permissions. You must have administrator access. For more information about AIMS APIs, see [Access and Identity Management Service](https://console.product.dev.alertlogic.com/api/aims/).

Note the `access_key_id` and the `secret_key` values for use in the deployment steps below.

**Note:** An account can create only five access keys. If you receive a "limit exceeded" response, you must delete some access keys before you can create more. Use the following command to list access keys:

```
curl -s -X GET -H "x-aims-auth-token: $AL_TOKEN" https://api.global-services.global.alertlogic.com/aims/v1/$AL_ACCOUNT_ID/users/$AL_USER_ID/access_keys | jq
```

In the following curl command, replace ``ACCESS_KEY_ID_HERE`` with the access key you want to delete.

```

curl -X DELETE -H "x-aims-auth-token: $AL_TOKEN" https://api.global-services.global.alertlogic.com/aims/v1/$AL_ACCOUNT_ID/users/$AL_USER_ID/access_keys/<ACCESS_KEY_ID_HERE>
```

## Download and deploy the ARM template 

You can use either the Microsoft Azure portal or a command line to deploy the template. To perform either procedure, you must first log into the [Azure portal](https://portal.azure.com). 

**Note:** The steps in this section require an active Azure subscription. To verify your Azure subscription, visit [Azure subscriptions blade](https://portal.azure.com/#blade/Microsoft_Azure_Billing/SubscriptionsBlade).

If your organization uses multiple Active Directory tenants, log into the same tenant used to [Register a New Azure Web Application](#register-a-new-azure-web-application).

### Deploy with the custom ARM Template in an Azure Subscription

Click the button below to start deployment. 

[![Deploy to Azure](https://azuredeploy.net/deploybutton.png)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Falertlogic%2Fehub-collector%2Fmaster%2Ftemplates%2Fehub.json)

1. To start a deployment, provide the following required template parameters, and then click the `Purchase` button:
   - **Name** - Type the name of the log source to appear in the Alert Logic console.
   - **Storage Name** - Type any storage account name (that does not currently exist).
   - **Alert Logic Access Key ID** - Type the `access_key_id` you created above.
   - **Alert Logic Secret Key** - Type the `secret_key` you created above.
   - **Alert Logic API endpoint** - Leave the default value (api.global-services.global.alertlogic.com).
   - **Alert Logic Data Residency** - Leave the value as "default".
   - **Service Principal ID** - Type the `Object ID` of the application that created the subscription. 
   
   **Note:** You can obtain this value from _Azure_ > _AD_ > _App registrations_ > _Your app name_ > Link under _Managed application in local directory_ > _Properties_ > _Object ID_.
   
   - **App Client ID** - Type the GUID of your application that created the subscription. 
   
   **Note:** You can obtain this value from _Azure_ > _AD_ > _App registrations_ > _Your app name_
   - **App Client Secret** - Type the secret key of your application (available from from _App Registrations_).

1. Click **Purchase**.

### Deploy through the Azure CLI

If you want to deploy the template through the Azure command line interface (CLI), you can use either [Azure Cloud Shell](https://docs.microsoft.com/en-gb/azure/cloud-shell/quickstart#start-cloud-shell) or a local installation of [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest).

**To deploy through the Azure CLI:**

1. In the command line, type the following to create a new resource group: 

**Note:** The example below creates a new resource group in the "Central US" location.

    ``` az group create --name <new-resource-group-name> --location "Central US"  ```

1. In the Azure portal, access the `Resource groups` blade, and then select the resource group you created.
1. Select `Access Control (IAM)`, and add `Website Contributor` role to the Active Directory application identity you created above.
1. In the command line, type the following command to deploy a template, and enter the required parameters when prompted.
   
   ```
    az group deployment create \
        --resource-group <new-resource-group-name> \
        --template-uri "https://raw.githubusercontent.com/alertlogic/ehub-collector/master/templates/ehub.json"
    ```

## Verify the Installation
**To verify successful installation of the template:**

1. In the Azure portal, access `Function Apps`, and then choose the Alert Logic Event Hub collector function. 
1. Click `Functions` > `Master` > `Monitor` and verify the recent log entry has the status of `OK` and contains no error messages.
**Example:** `Ehub source checkin OK`.
1. In the Alert Logic console, navigate to `Configuration` > `Deployments` > `All Deployments` > `Log Sources`, and then filter the list by `Push (Office 365, EventHub)` collection method. 
1. Verify a new Azure Event Hub log source with the name provided during `az group deployment create` [above](#deploy-through-the-azure-cli) appears with the source status as `OK`.

# Integrating With Azure Event Hubs

The following links contain instructions to help you integrate different Azure services with Event Hubs. Use **alertlogic-log** as a target event hub name if available during configuration.

* [Azure Active Directory Logs](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/tutorial-azure-monitor-stream-logs-to-event-hub#stream-logs-to-an-event-hub)
* [Azure Diagnostic Logs](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/diagnostic-logs-stream-event-hubs#stream-diagnostic-logs-using-the-portal)
* [Azure Activity Logs](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/activity-logs-stream-event-hubs#enable-streaming-of-the-activity-log)
* [Azure Security Center Events](https://docs.microsoft.com/en-us/azure/security-center/security-center-partner-integration#exporting-data-to-a-siem)
* [Azure SQL Audit Logs](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-auditing#subheading-2)

# How the collector works

The [template](https://github.com/alertlogic/ehub-collector/blob/master/templates/ehub.json) creates an `AlertLogicIngest-<region-name>-<unique-string>` Event Hubs namespace where the "alertlogic-log" event hub is created. The Alert Logic collector listens to the Azure event hub and forwards incoming events to the Alert Logic Ingestion service. If event collection fails, Alert Logic stores the data in the `alertlogic-dl` Azure Blob container located in the storage account you specified during template deployment. 

## Master function

The `Master` function is a timer trigger function responsible for:
- Registering the Azure web application in the Alert Logic backend
- Reporting health checks to the backend

**Note:** When you release a new version of the collector, remember to increment the version number in the npm package.json file.

## Updater function

The `Updater` function is a timer triggered function that runs a deployment sync operation every 12 hours to keep the web application up to date.

## `EHubActivityLogs` function

The `EHubActivityLogs` function listens to the `insights-operational-logs` event hub located in the Event Hubs namespace created during [collector setup](#deploy-with-the-custom-arm-template-in-an-azure-subscription). The `insights-operational-logs` is created automatically by Azure when a subscription [Log Profile is integrated with Event Hub through the Azure Monitor service](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/stream-monitoring-data-event-hubs#azure-subscription-monitoring-data).
Follow this guide to [Stream the Azure Activity Log to Event Hubs](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/activity-logs-stream-event-hubs). 
Collected JSON objects are wrapped into the protocol buffer (protobuf) [structure](https://github.com/alertlogic/al-collector-js/blob/master/proto/common_proto.piqi.proto) and then forwarded to the Alert Logic Ingestion service.

## EHubGeneral function

The `EHubGeneral` function listens to `alertlogic-log`, which is created during [collector setup](#deploy-with-the-custom-arm-template-in-an-azure-subscription). The `alertlogicloghub` event hub can be used for integration with logs such as [diagnostic logs](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/diagnostic-logs-stream-event-hubs) or [Azure AD logs](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/tutorial-azure-monitor-stream-logs-to-event-hub).
Collected JSON objects are wrapped into the protobuf [structure](https://github.com/alertlogic/al-collector-js/blob/master/proto/common_proto.piqi.proto) and then forwarded to the Alert Logic Ingestion service.

## DLBlob function

If neither `EHubActivityLogs` or `EHubGeneral` cannot process incoming event hub records, unprocessed messages are saved as blobs to the `alertlogic-dl` container, so collection can be tried again later. The `alertlogic-dl` container is located in the collector web application storage account created during collector setup.
The `DLBlob` function processes dead letter blobs very 15 minutes. The `DLBlob` function lists all blobs located in  `alertlogic-dl` container and processes them according to the function to which the dead letter blob belongs. After a blob is processed, it is removed from the container.

# Local development

To enable local development, perform the following procedure:

1. Clone the repo `git clone git@github.com:alertlogic/ehub-collector.git`.
1. `cd ehub-collector`
1. Run `./local_dev/setup.sh`.
1. Edit `./local_dev/dev_config.js`.
1. Run the `Master` function locally: `npm run local-master`.
1. Run the `Updater` function locally: `npm run local-updater`.
1. Run the `EHubGeneral` function locally: `npm run local-ehub-general`.
1. Run the `EHubActivityLogs` function locally: `npm run local-ehub-activitylogs`.
1. Run `npm test` to perform code analysis and unit tests.

Please use the following [code style](https://github.com/airbnb/javascript) as much as possible.

## Setting environment in dev_config.js

- `process.env.APP_TENANT_ID` - The GUID of the tenant (such as `alazurealertlogic.onmicrosoft.com`)
- `process.evn.APP_RESOURCE_GROUP` - The name of the resource group where you deployed your application.
- `process.env.CUSTOMCONNSTR_APP_CLIENT_ID` - The GUID of your application that created the subscription.
**Note** You can obtain this value from _Azure_ > _AD_ > _App registrations_ > _Your app name_
- `process.env.CUSTOMCONNSTR_APP_CLIENT_SECRET` - The secret key of your application from _App Registrations_.
- `process.env.CUSTOMCONNSTR_APP_CI_ACCESS_KEY_ID` - The access key returned from AIMs [above](#create_an_alert_logic_access_key).
- `process.env.CUSTOMCONNSTR_APP_CI_SECRET_KEY`- The secret key returned from AIMs [above](#create_an_alert_logic_access_key).


# Useful Links

- [Node.js static code analysis tool](http://jshint.com/install/)
- [How to monitor Azure functions?](https://docs.microsoft.com/en-us/azure/azure-functions/functions-monitoring)
- [Server application to Web API authentication.](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-authentication-scenarios#daemon-or-server-application-to-web-api)
- [Azure Web apps API reference](https://docs.microsoft.com/en-us/rest/api/appservice/webapps)
