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
