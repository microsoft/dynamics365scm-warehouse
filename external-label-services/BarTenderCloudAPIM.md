# BarTender Cloud Azure API Management facade

The basic steps of integrating with BarTender Cloud are documented in [Print labels using the Seagull Scientific BarTender® label service solution](https://learn.microsoft.com/en-us/dynamics365/supply-chain/supply-chain-dev/label-printing-using-bartender). Using this guide, you can deploy an Azuire API Management (APIM) facade in front of the BarTender Cloud. This facade provides two improvements:

* In the steps, a personal authentication token is used to authenticate Dynamics 365 SCM to BarTender Cloud. This token has a 30 days lifetime and must be refreshed before the expiration. Using APIM policies, it is possible for APIM to fetch an authentication token from BarTender Cloud's authentication service (using the [BarTender Cloud REST API Authentication Using Password-Based OAuth](https://support.seagullscientific.com/hc/en-us/articles/16272131900567-BarTender-Cloud-REST-API-Authentication-Using-Password-Based-OAuth) support) and store it in a Redis cache. Warehouse Management then uses a long lived Subscription key to authenticate against the APIM facade, removing the need to update the authentication secret every 30 days.
* The response from the external label service is not stored in the request log. Using the service facade, it is possible to log information returned by the BarTender Cloud's Actions API to an Azure Application Insights instance for monitoring.

Please note that this document is intended to only give a high-level overview of the configuration steps. For more information on the BarTender Cloud, Azure API Management, Azure Cache for Redis or Azure Application Insights, please refer to the documentation for the respective service.

## Prerequisites

1) An Azure API Management instance.

2) Optionally, an Azure Cache for Redis instance. This is needed only the Consumption tier of Azure API Management is used.

3) Optionally, An Application Insights instance if you wish to log action IDs returned from BarTender Cloud.

4) A working connection between BarTender Cloud and Warehouse Management.

## Configuration steps

1) Configure BarTender Cloud for Password-Based OAuth as described in the document linked above. Note the URL of the authorization server (ending with the `/oauth/token` path), the client ID and the client secret as well as the username and password of the user used for the integration.

2) Create the following Named Values in the APIM portal:
* **authorizationServer** contains the URL to the OAuth server used by BarTender Cloud, for example `https://bartendercloud-production.us.auth0.com/oauth/token`.
* **clientId** contains the client ID of the application registered in BarTender Cloud
* **clientSecret** contains the client secret of the application registered BarTender Cloud
* **username** contains the username of the user that will be used to integrate with BarTender Cloud
* **password** contains the password of the above user

3) Connect the Redis Cache and Application Insights to your Azure API Management instance.

4) Create a new Subscription in APIM. Note the Subscription key, as it will be used in the configuration later. 

5) Create a new API definition in APIM to represent the BarTender Cloud API
* Set the backend to the appropriate BarTender Cloud instance (use the URL that is currently configured in Warehouse Management's External Service Instance).
* Mark the API as "Subscription required"
* Enable Application Insights monitoring for the API

6) On the API definition, use the following Inbound processing policy:
```(xml)
<inbound>
    <base />
    <cache-lookup-value key="BarTenderBearerToken" variable-name="BarTenderBearerToken" />
    <choose>
        <when condition="@(!context.Variables.ContainsKey("BarTenderBearerToken"))">
            <set-variable name="BarTenderUsername" value="{{Username}}" />
            <set-variable name="BarTenderPassword" value="{{Password}}" />
            <send-request ignore-error="true" timeout="20" response-variable-name="authorizationServerResponse" mode="new">
                <set-url>{{AuthorizationServer}}</set-url>
                <set-method>POST</set-method>
                <set-header name="Content-Type" exists-action="override">
                    <value>application/x-www-form-urlencoded</value>
                </set-header>
                <set-body>@{
                    var _urlEncodedUsername = System.Net.WebUtility.UrlEncode((string)context.Variables["BarTenderUsername"]);
                    var _urlEncodedPassword = System.Net.WebUtility.UrlEncode((string)context.Variables["BarTenderPassword"]);

                    return "grant_type=password&client_id={{clientId}}&client_secret={{ClientSecret}}&username=" + _urlEncodedUsername + "&password={{Password}}&audience=https://BarTenderCloudServiceApi&scope=openid profile user:query offline_access";
                }</set-body>
            </send-request>
            <set-variable name="authorizationServerResponseObject" value="@(((IResponse)context.Variables["authorizationServerResponse"]).Body.As<JObject>())" />
            <set-variable name="BarTenderBearerToken" value="@((string)((JObject)context.Variables["authorizationServerResponseObject"])["access_token"])" />
            <set-variable name="expiresIn" value="@(int.Parse((string)((JObject)context.Variables["authorizationServerResponseObject"])["expires_in"]))" />
            <set-variable name="randomBackOffDelay" value="@(new Random().Next(0,(int)context.Variables["expiresIn"]/3))" />
            <cache-store-value key="BarTenderBearerToken" value="@((string)context.Variables["BarTenderBearerToken"])" duration="@((int)context.Variables["expiresIn"]  - (int)context.Variables["randomBackOffDelay"])" />
        </when>
    </choose>
    <set-header name="Authorization" exists-action="override">
        <value>@("Bearer " + (string)context.Variables["BarTenderBearerToken"])</value>
    </set-header>
    <!--  Don't expose APIM subscription key to the backend. -->
    <set-header name="Ocp-Apim-Subscription-Key" exists-action="delete" />
</inbound>
```

7) Create a new operation called "Submit a script to run" for the POST HTTP operation and the URL `/api/actions`

8) Change the configuration of the External Service Instance in Warehouse Management used to connect to BarTender Cloud:
* Replace the BarTender Cloud URL with the URL of your APIM instance.
* Replace the authentication secret with the Subscription key created in step 4 above.

9) Change the configuration of the External Service Definition in Warehouse Management: 
* Remove the `Authorization` header from the operation.
* Add a new header `Ocp-Apim-Subscription-Key` with value `$auth.secret$`.

## Logging BarTender Action API responses

If you wish to log the responses from the Action API in Application Insights, you can add the following Outbound processing policy to the "Submit a script to run" operation created in step 7:
```(xml)
<outbound>
    <base />
    <set-variable name="responseBodyString" value="@(context.Response.Body.As<string>(preserveContent: true))" />
    <trace source="BarTenderActionsAPIPost" severity="verbose">
        <message>Action submitted to BarTender</message>
        <metadata name="response" value="@((string)context.Variables["responseBodyString"])" />
    </trace>
</outbound>
```

This will store the whole response in Application Insights. You can then use a log query to extract the Action ID and the Status URL, such as:
```(kusto)
traces
| where message == "Action submitted to BarTender"
| extend bartenderResponse = parse_json(tostring(customDimensions.response))
| extend bartenderActionId = bartenderResponse.Id, bartenderActionStatusURL = bartenderResponse.StatusUrl
| project-away bartenderResponse
| join dependencies on operation_Id
| project timestamp, bartenderActionId, bartenderActionStatusURL, target, name, data, success, resultCode, duration, performanceBucket
```
