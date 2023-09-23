# smappee-services-integration

## What

A Home Assistant integration for smappee adding services to consume public API provided by Smappee. This is different from the already existing smappee integration that only provides sensors and devices. 

The goal of this integration is to build on top of that to provide services also so that for example you can change your smappee wall charger output depending on solar production or whatever automations you can imagine. 

## Technical documentation

I'll start by posting some of the API snippets that I'll need to accomplish above scenario. You can find all smappee public API documentation at https://smappee.atlassian.net/wiki/spaces/DEVAPI/overview

Including instructions on how to get your personal api access. 

The following are curl commands (with my personal credentials replcade) so that you can easily experiment yourself. 

### Authorization

All these endpoints rely on an authorization token. To retrieve such a token you have 2 requests to provide client and account details and retrieve a token.

To retrieve an initial token, valid for 10 hours, you can execute this command:

```
curl --location 'https://app1pub.smappee.net/dev/v1/oauth2/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'grant_type=password' \
--data-urlencode 'client_id=YOUR_CLIENT_ID_HERE' \
--data-urlencode 'client_secret=YOUR_SECRET_HERE' \
--data-urlencode 'username=YOUR_USERNAME_HERE' \
--data-urlencode 'password=YOUR_PASSWORD_HERE'
```

This will return you something similar to the below extract:

```
{
    "access_token": "SOME_VALID_ACCESS_TOKEN",
    "refresh_token": "SOME_VALID_REFRESH_TOKEN",
    "expires_in": 86400
}
```

That `access_token` is what you put on the requests using the `Authorization: Bearer` header. 

If you already have a token but it's no longer valid (is expired) you can also use the provided `refresh_token` to get it replaced. An example request for that is:

```
curl --location 'https://app1pub.smappee.net/dev/v1/oauth2/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'grant_type=refresh_token' \
--data-urlencode 'refresh_token=YOUR_REFRESH_TOKEN_HERE' \
--data-urlencode 'client_id=YOUR_CLIENT_ID_HERE' \
--data-urlencode 'client_secret=YOUR_SECRET_HERE'
```

### Retrieving service locations

In order to update smappee charger charge speed you'll need the serial number of that smappee charger. To get that you can request an overview of service locations linked to your account. For this request to work you'll need the `access token` from the previous calls. 

```
curl --location 'https://app1pub.smappee.net/dev/v3/servicelocation' \
--header 'Authorization: Bearer YOUR_ACCESS_TOKEN_HERE'
```

This renders something similar to this (again all of this is altered to hide my details):

```
{
    "appName": "YOUR_CLIENT_ID_HERE",
    "serviceLocations": [
        {
            "serviceLocationId": SOME_NUMERIC_IDENTIFIER,
            "serviceLocationUuid": "A_UUID_HERE",
            "name": "HOW_YOU_NAMED_THIS",
            "deviceSerialNumber": "YOUR_SERIAL_NUMBER_HERE"
        }
    ]
}
```

Note that the `deviceSerialNumber` you receive in this step is the serial of your connect hub. This is different from the serial number of your actual charger that you'll need for the next steps. If you need to find the serial number of your charger the easy way is to check in the app config. 

That `serviceLocationId` can be used to retrieve charging sessions for all chargers on that location (tested, not yet documented here).  

The appName in my case matched my `client_id`. And the `name` property is something I could pick myself in my app to name the chargers/locations I have in use.

### Setting charge speed

Now in order to control chargers we have a few options. The general endpoint used for this is shown below, just the options change depending on what you want to achieve. Note that this is a `PUT` and that the url contains your serial number and the position of the plug. A JSON formatted body is required. 

```
curl --location --request PUT 'https://app1pub.smappee.net/dev/v3/chargingstations/YOUR_SERIAL/connectors/CONNECTOR/mode' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer YOUR_TOKEN' \
--data '{
  "mode": "NORMAL",
  "limit": {
    "unit": "PERCENTAGE",
    "value": 75
  }
}'
```

The connector depends on your charger model, from their documentation:

```
When facing a charging station with two connectors (i.e. you are looking at the LED light) the right connector has position 1, the left has position 2. A charging station with one connector only has position 1.
```

#### Charge state

First of all we can set the charge mode to one of `NORMAL`, `SMART` or `PAUSED`. What I know so far is that `NORMAL` is just charging and `PAUSED` is where the charger will stop charging but can resume (tested on Mercedes) by changing this state again. I personally don't use the `SMART` option but it's a way to have smappee manage charges by configuration in app. That configuration can be a schedule or solar if you have the right set up. 

Note that if you put the charger in `PAUSED` it will only remain paused as long as the connector remains plugged in. If you unplug the connector and plug it in again it will go back to `NORMAL` in my experience. 

Example json body request to PAUSE the current charge session:

```
{
  "mode": "PAUSE"
}
```

#### Charge speed

Charge speed can be controlled either by a limit in `PERCENTAGE` or a limit on `AMPERE`. 

If you use the `PRECENTAGE` option it will charge at that percentage of the max allowed charge speed (in amps) configured in the app. There is always a minimum draw of `6A` so at `0%` that is what you get. For a `32A` max setup at `10%` I get just above `8A`.

This setting remains between charges. 

Example json body to set charge limit to Amp level:

```
{
  "mode": "NORMAL",
  "limit": {
    "unit": "AMPERE",
    "value": 10
  }
}
```

Note that in my case testing shows that setting a specific A value doesn't render the expected results of that exact amp value being set to the charger. Might be related to my load balancing set up or some specific misconfiguration on my side. 
