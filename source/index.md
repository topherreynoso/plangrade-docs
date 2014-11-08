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

Some API endpoints only require your application `key` and `secret` (for instance, [creating a user](#create-new-user-account)), but most require an OAuth `access_token`.

For endpoints that require an OAuth access token, it should be included in the Authorization HTTP header like so:

`Authorization: Bearer <T0K3N_H3R3>`

<aside class="warning">
Be sure to replace `<T0K3N_H3R3>` with an OAuth `access_token`.
</aside> 

## OAuth2

plangrade's API lets you interact with a user's plangrade account and act on their behalf to view, create, update, and destroy things like companies, plans, and participants. To do so, your application first needs to request authorization from users.

plangrade implements the [OAuth 2.0 standard](http://oauth.net/2/) to facilitate this authorization. Similar to Facebook and Twitter's authentication flow, the user is first presented with a permission dialog for your application, at which point the user can either approve the permissions requested or reject them. Once the user aproves, an `authorization_code` is sent to your application which will then be [exchanged](#finish-authorization) for an `access_token` and a `refresh_token` pair.

The `access_token` can then be used to make API calls which require user authentication like [Participants](#participants) or [Companies](#companies).

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

### Attributes

Parameter | Type | Optional? | Read Only? | Description
--------- | ---- | --------- | ---------- | -----------
`id` | Integer| No | Yes | User's plangrade ID
`name` | String | No | No | User's name
`email` | String | No | No | User's email address

## Create New User Account

```ruby
# Create a new user account
token.post('api/v1/users')
```

> If successful, you'll receive either this response or a list of errors:

```ruby
{
  "id" => 1
}
```

Create a new user account with a `name` and valid `email` address.

<aside class="success">
This endpoint does not require an OAuth access token.
</aside>

### HTTP Request

`POST https://plangrade.com/api/v1/users`

## Get User Info

```ruby
# Fetch account information for the authorized user
token.get('/api/v1/users/1')
```

> If successful, you'll receive this response:

```ruby
{
  "user" => {
    "id" => 1
    "name" => "Test User"
    "email" => "compliance@plangrade.com"
  }
}
```

Retrieve information about the authorized user.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`GET https://plangrade.com/api/v1/users/{id}`

## Update User Account Info

```ruby
# Update account information for the authorized user
token.put('/api/v1/users/1')
```

> If successful, you'll receive either this response or a list of errors:

```ruby
{
  "user" => {
    "id" => 1
    "name" => "Test User"
    "email" => "support@plangrade.com"
  }
}
```

Update an authorized user's account information (must include `name` and `email`).

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`PUT https://plangrade.com/api/v1/users/{id}`

## Delete User Account

```ruby
# Delete an authorized user's own account
token.delete('/api/v1/users/1')
```

> If successful, you'll recieve a "204 No Content" response.

Delete the authenticated user's plangrade account.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`DELETE https://plangrade.com/api/v1/users/{id}`

<aside class="warning">
Deleting a user will result in deleting all of its associations with any companies as well. While the companies will not be affected, you should ensure that there is at least one other manager still associated with the company or else only plangrade administrators will have access to it. Deleted users cannot be restored.
</aside>

# Companies

### Attributes

Parameter | Type | Optional? | Read Only? | Description
--------- | ---- | --------- | ---------- | -----------
`id` | Integer| No | Yes | Company's plangrade ID
`ein` | String | No | No | Company's Employer Identification Number (numbers only)
`name` | String | No | No | Company's name
`grade` | String | Yes | Yes | Company's compliance letter grade, ranging from A+ to F

## Create New Company

```ruby
# Create a new company
token.post('api/v1/companies')
```

> If successful, you'll receive either this response or a list of errors:

```ruby
{
  "id" => 1
}
```

Create a new company with a `name` and `ein` for the authorized user to manage.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`POST https://plangrade.com/api/v1/companies`

## Get Company Info

```ruby
# Fetch company information for an authorized user
token.get('/api/v1/companies/1')
```

> If successful, you'll receive this response:

```ruby
{
  "company" => {
    "id" => 1
    "ein" => "123456789"
    "name" => "plangrade"
    "grade" => "A+"
  }
}
```

Retrieve information about an authorized user's company.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`GET https://plangrade.com/api/v1/companies/{id}`

## Get All Companies

```ruby
# Fetch all companies associated with the authorized user
token.get('/api/v1/companies')
```

> If successful, you'll receive this response:

```ruby
{
  "companies" => [
    {
      "id" => 1
      "ein" => "123456789"
      "name" => "plangrade"
      "grade" => "A+"
    },
    {
      "id" => 2
      "ein" => "123456788"
      "name" => "plangrade, LLC"
      "grade" => "A"
    },
    {
      "id" => 3
      "ein" => "123456787"
      "name" => "plangrade, Inc."
      "grade" => "A+"
    }
  ]
}
```

Retrieve information about all of the authorized user's companies.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`GET https://plangrade.com/api/v1/companies`

## Update Company Info

```ruby
# Update company information for an authorized user
token.put('/api/v1/companies/1')
```

> If successful, you'll receive either this response or a list of errors:

```ruby
{
  "company" => {
    "id" => 1
    "ein" => "123456789"
    "name" => "plangrade, LLC"
    "grade" => "A+"
  }
}
```

Update an authorized user's company information (must include `name` and `ein`).

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`PUT https://plangrade.com/api/v1/companies/{id}`

## Delete Company

```ruby
# Delete an authorized user's company
token.delete('/api/v1/companies/1')
```

> If successful, you'll recieve a "204 No Content" response.

Delete one of the authenticated user's companies.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`DELETE https://plangrade.com/api/v1/companies/{id}`

<aside class="warning">
Deleting a company will result in deleting all of its information as well. This includes plans, participants, documents, its history and any future distributions. This action cannot be reversed.
</aside>

# Participants

### Attributes

&nbsp;&nbsp;&nbsp;Parameter&nbsp;&nbsp;&nbsp; | Type | Optional? | Read Only? | Description
--------------- | ---- | --------- | ---------- | -----------
`id` | Integer| No | Yes | Participant's plangrade ID
`company_id` | Integer | No | Yes | Plangrade ID of the company the participant is associated with
`employee_id` | Integer | No | No | Plangrade ID of the employee the participant is associated with; if the `id` and `employee_id` are equivalent it signifies that the participant is an employee, otherwise the participant is a dependent
`first_name` | String | No | No | Participant's first name
`last_name` | String | No | No | Participant's last name
`email` | String | Yes | No | Participant's email address
`phone` | Integer | Yes | No | Participant's SMS enabled phone number (10 numbers only)
`street1` | String | No | No | The first line of the participant's address
`street2` | String | Yes | No | The second line of the participant's address
`city` | String | No | No | The city of the participant's address
`state` | String | No | No | The state of the participant's address
`zip` | String | No | No | The zip code of the participant's address (5 numbers only)
`dob` | Date | No | No | Participant's date of birth
`ssn` | String | See below | Yes | The last four digits of the participant's Social Security Number; while this is required in order to create a new participant, it is not visible in any other endpoint

## Create New Participant

```ruby
# Create a new participant
token.post('api/v1/participants')
```

> If successful, you'll receive either this response or a list of errors:

```ruby
{
  "id" => 1
}
```

Create a new participant with a `company_id`, `first_name`, `last_name`, `street1`, `city`, `state`, `zip`, and `dob`.

### Employee Participants

When adding an employee, you must include the last four of their `ssn` and an `employee_id` with a value of `0` (plangrade will detect that the record is that of an employee and add the employee's own `id` to the `employee_id` field after the record is created).

### Dependent Participants
The last four of their `ssn` is optional for a dependent, however an `employee_id` is necessary. As such, employees must have a participant record created before their dependents.

### Optional

You may optionally include a valid `email` address and/or an SMS enabled `phone` number (10 numbers only). This enables plangrade to send the participant electronic authorizations and notices. Additionally, a `street2` may also be included, if necessary.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`POST https://plangrade.com/api/v1/participants`

## Get Participant Info

```ruby
# Fetch a participant's information
token.get('/api/v1/participants/1')
```

> If successful, you'll receive this response:

```ruby
{
  "participant" => {
    "id" => 1
    "company_id" => 1
    "employee_id" => 1
    "first_name" => "Comply"
    "last_name" => "Ants"
    "email" => "compliance@plangrade.com"
    "phone" => "1234567890"
    "street1" => "1234 Fake St."
    "street2" => ""
    "city" => "Orem"
    "state" => "UT"
    "zip" => "84606"
    "dob" => "1984-12-30"
  }
}
```

Retrieve information about a participant.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`GET https://plangrade.com/api/v1/participants/{id}`

## Get All Company's Participants

```ruby
# Fetch all participants associated with a company
token.get('/api/v1/companies')
```

> If successful, you'll receive this response:

```ruby
{
  "participants" => [
    {
      "id" => 1
      "company_id" => 1
      "employee_id" => 1
      "first_name" => "Comply"
      "last_name" => "Ants"
      "email" => "compliance@plangrade.com"
      "phone" => "1234567890"
      "street1" => "1234 Fake St."
      "street2" => ""
      "city" => "Orem"
      "state" => "UT"
      "zip" => "84606"
      "dob" => "1984-12-30"
    },
    {
      "id" => 2
      "company_id" => 1
      "employee_id" => 1
      "first_name" => "Fire"
      "last_name" => "Ants"
      "email" => "support@plangrade.com"
      "phone" => "1234567891"
      "street1" => "1234 Fake St."
      "street2" => ""
      "city" => "Orem"
      "state" => "UT"
      "zip" => "84606"
      "dob" => "1986-04-18"
    },
    {
      "id" => 3
      "company_id" => 1
      "employee_id" => 1
      "first_name" => "Red"
      "last_name" => "Ants"
      "email" => ""
      "phone" => "1234567892"
      "street1" => "1234 Fake St."
      "street2" => ""
      "city" => "Orem"
      "state" => "UT"
      "zip" => "84606"
      "dob" => "2006-11-22"
    }
  ]
}
```

Retrieve information about all of the participants in an authorized user's company.

### Mandatory

You must include a `company_id` parameter in order to allow plangrade to identify which company you want to query and to verify that the authorized user has access to this company.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`GET https://plangrade.com/api/v1/participants?company_id={1}`

## Update Participant Info

```ruby
# Update participant information
token.put('/api/v1/participants/1')
```

> If successful, you'll receive either this response or a list of errors:

```ruby
{
  "participant" => {
    "id" => 1
    "company_id" => 1
    "employee_id" => 1
    "first_name" => "Comp"
    "last_name" => "Lance"
    "email" => "compliance@plangrade.com"
    "phone" => "1234567890"
    "street1" => "1234 Fake St."
    "street2" => ""
    "city" => "Orem"
    "state" => "UT"
    "zip" => "84606"
    "dob" => "1984-12-30"
  }
}
```

Update a participant's information. You must include an `employee_id` for every participant update request.

### Employees

Remember to make the `employee_id` equivalent to the participant's `id`.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`PUT https://plangrade.com/api/v1/participants/{id}`

## Delete Participant

```ruby
# Delete a company's participant
token.delete('/api/v1/participants/1')
```

> If successful, you'll recieve a "204 No Content" response.

Delete a participant from one of the authorized user's companies.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`DELETE https://plangrade.com/api/v1/participants/{id}`

<aside class="warning">
Deleting a participant will result in deleting all of their information as well as any associated dependents. Make sure that you actually want to delete and not just archive the participant. This includes deleting any distributions, history, and documents specific to this participant and all of the same information associated with any dependents. This action cannot be reversed.
</aside>