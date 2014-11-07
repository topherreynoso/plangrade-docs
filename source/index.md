---
title: plangrade | API Docs

language_tabs:
  - shell
  - ruby
  - python

toc_footers:
  - <a href='https://plangrade.com/oauth/applications'>Manage Your Applications</a>
  - <a href='https://plangrade.com'>&copy; plangrade 2014</a>

includes:
  - errors

search: true
---

# Introduction

## Welcome

> Feel free to use the tabs above to select the code that suits your needs.

This is the plangrade API documentation. You can use our API to access plangrade endpoints, allowing clients to retrieve, create, update, and destroy information on companies, plans, participants, compliance documents, and more.

Our API is organized around REST with JSON responses. Our API is designed to use HTTP response codes to indicate API success/errors. We allow you to interact with our API from a client-side web application. 

<aside class="notice">
JSON will be returned in all responses from the API.
</aside>

## API v1

Currently this is the only public version.

> You'll find sample code in this pane throughout these docs.

**Base API Endpoint:** `https://plangrade.com/api/v1`

References will be made throughout this version to resources omitting any root urls. For instance, this line:

`https://plangrade.com/api/v1/participants`

will be referenced as:

`/api/v1/participants`

<aside class="success">
This entire API is *https only*.
</aside>

# Authentication

> You can provide your Application key and secret when you initialize our libraries.

```ruby

# set API keys
app_id = "blahblahblah"
secret = "shhhhhh"

# set an OAuth access token
token = "JRRTolkien"
```

To interact with the API, you will need API keys. [Create a consumer application](https://plangrade.com/oauth/applications) in order to get an Application `key` and `secret`.

Some API endpoints only require your application `key` and `secret` (for instance, [creating a user](#)), but most require an OAuth `access_token`.

For endpoints that require an OAuth access token, it should be included in the Authorization HTTP header like so:

`Authorization: Bearer <T0K3N_H3R3>`

<aside class="warning">
Be sure to replace `<T0K3N_H3R3>` with an OAuth `access_token`.
</aside> 

## OAuth2

plangrade's API lets you interact with a user's plangrade account and act on their behalf to view, create, update, and destroy things like companies, plans, and participants. To do so, your application first needs to request authorization from users.

plangrade implements the [OAuth 2.0 standard](http://oauth.net/2/) to facilitate this authorization. Similar to Facebook and Twitter's authentication flow, the user is first presented with a permission dialog for your application, at which point the user can either approve the permissions requested or reject them. Once the user aproves, an `authorization_code` is sent to your application which will then be [exchanged](#finid-authorization) for an `access_token` and a `refresh_token` pair.

The `access_token` can then be used to make API calls which require user authentication like [Participants](#) or [Companies](#).

### Token Lifetimes

**Authorization codes** expire *very quickly*: 10 minutes.

**Access tokens** are *short lived*: 2 hours.

**Refresh tokens** are *good forever* (or until revoked by the user).

A refresh token can be used to generate a new `access_token` and `refresh_token` pair. You can [refresh your authorization](#refresh-authorization) to regain active access to a user's account but so long as you store the `refresh_token` you can maintain authorization indefinitely without requiring the user to re-authorize.

## Request Authorization

```ruby
redirect_uri = "https://www.myredirect.com/redirect"
client = OAuth2::Client.new(app_id, secret, :site => "https://plangrade.com")
redirect_to client.auth_code.authorize_url(:redirect_uri => redirect_uri)
```

To start the Oauth process, construct the initiation URL which the user will visit in order to grant permission to your application. It describes the permissions your application requires (which grants read/write access to companies, plans, and participants), and who the client application is (your application's name).

### URL Format

`https://plangrade.com/api/v1/authenticate?client_id={client-id}&response_type=code&redirect_uri={redirect_uri}`

Parameter | Description
--------- | -----------
client_id | Application key
response_type | This must always be set to `code`
redirect_uri | URL where the user will be redirected to afterwards

<aside class="notice">
Remember to url-encode all querystring parameters.
</aside>

## Finish Authorization

Once the user returns to your application via the `redirect_uri` you specified, there will be a `code` querystring parameter appended to that URL. Exchange the authorization `code` for an `access_token` and `refresh_token` pair.

### HTTP Request

```ruby
client = OAuth2::Client.new(app_id, secret, :site => "https://plangrade.com")
token = client.auth_code.get_token(params[:code], :redirect_uri => redirect_uri)
```

> Successful Response:

```ruby
{
  "access_token": "YLAZSreh165CAD2tPhAEzFCYYIrVyFomLWUDMGFBZIw9KtIg4q",
  "expires_in": 7200,
  "refresh_token": "Pgk+l9okjwTCfsvIvEDPrsomE1er1txeyoaAkTIBAuXza8WvZY",
  "token_type": "bearer"
}
```

`POST https://plangrade.com/oauth/v1/token`

Parameter | Description
--------- | -----------
client_id | Application key
client_secret | Application secret
code | The authorization code included in the redirect URL
grant_type | This must be se to `authorization_code`
redirect_uri | The same redirect_uri specified in the initiation step

### Response Parameters

Parameter | Description
--------- | -----------
access_token | A new access token with requested scopes
expires_in | The lifetime of the access token, in seconds. Default is 7200
refresh_token | New refresh token
token_type | Always `bearer`

## Refresh Authorization

```ruby
new_token = token.refresh!
```

> Successful Response:

```ruby
{
  "access_token": "hr1tSJk23tOv9RxR8cQuDpUD/kpUgf0cb9WfKtRoPxTw8ymbVt",
  "expires_in": 7200,
  "refresh_token": "lqopeyjq5AJln5fpMedElUULz+XBNNw9nAkIinxE0g4aEzxmDc",
  "token_type": "bearer"
}
```

Use a valid `refresh_token` to generate a new `access_token` and `refresh_token` pair.

### HTTP Request

`POST https://plangrade.com/oauth/v1/token`

Parameter | Description
--------- | -----------
client_id | Application key
client_secret | Application secret
refresh_token | A valid refresh token
grant_type | This must be set to `refresh_token`

### Response Parameters

Parameter | Description
--------- | -----------
access_token | A new access token with requested scopes
expires_in | The lifetime of the access token, in seconds. Default is 7200
refresh_token | New refresh token
token_type | Always  `bearer`

# Users

## Get Account Info

Retrieve information about the authorized user.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`GET https://plangrade.com/api/v1/users/{id}`

### Response

Parameter | Description
--------- | -----------
id | User's plangrade ID
name | User's name
email | User's email address
companies | The id's for all companies the user is authorized to manage

## Update Account Info

## Create New Account

# Companies

## Get Company Info

## Update Company Info

## Get All User's Companies

## Create New Company

## Delete Company

# Participants

## Get Participant Info

## Update Participant Info

## Get All Company's Participants

## Create New Participant

## Delete Participant