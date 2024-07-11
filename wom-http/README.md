# Dynamcis 365 WOM Sample Inbound/Outbound Shipment Order OData requests

This folder contains a selection of sample OData requests to demonstrate the creation of outbound and inbound shipment orders, getting the inventory update logs and exchanging product master information.

## Executing the requests

The *.http files can be executed with the REST Client extension (https://github.com/Huachao/vscode-restclient). Before connecting, you will need to update the .vscode/settings.json file with your connection parameters and select the environment in the REST Client.

## Acquiring the token

In order to connect to Dynamics 365 Supply Chain Management endpoint, you need to provide an OAuth 2.0 access token and save it in the EnvAOSToken variable. You can get the token in two ways:

    1. Using the capabilities of the vscode-restclient extension, by using the `{{$aadV2Token}}` system variable with scope `https://usnconeboxax1aos.cloud.onebox.dynamics.com/Odata.FullAccess` (replacing the URL with the URL of your environment).
    2. Registering an application with a client secret and then using the [Auth-GetTokenForApp.http] request, filling in the tenant, client ID and client secret.