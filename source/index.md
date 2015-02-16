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

> Use the tabs above to select the code that suits your needs. A [sample client app](https://github.com/topherreynoso/plangrade-ruby-client) is available in the ruby environment. It demonstrates the authentication flow and accessing the API endpoints.

This is the plangrade API documentation. You can use our API to access plangrade endpoints, allowing clients to retrieve, create, update, and destroy information on companies, plans, participants, compliance documents, and more.

Our API is organized around REST with JSON responses. Our API is designed to use HTTP response codes to indicate API success/errors. We allow you to interact with our API from a client-side web application. 

<aside class="notice">
JSON will be returned in all responses from the API.
</aside>

## API v1

Currently this is the only public version.

> You'll find sample code in this pane throughout these docs.

**Base API Endpoint:** `https://plangrade.com/api/v1`

<aside class="success">
This entire API is *https only*.
</aside>

# Authentication

> There is a gem available to facilitate authentication in a ruby environment: [omniauth-plangrade](https://github.com/topherreynoso/omniauth-plangrade). Please refer to the gem's documentation for full instructions.

> Provide your Application key and secret as the following variables when you initialize your app.

```ruby
# set API keys
ENV['PLANGRADE_CLIENT_ID'] = "jhsafdiurwe989734kjo3485uihw5io"
ENV['PLANGRADE_CLIENT_SECRET'] = "uysadsir5wt6837i9kjo3485u46swfd"
```

To interact with the API, you will need API keys. [Create a consumer application](https://plangrade.com/oauth/applications) in order to get an Application `key` and `secret`.

Some API endpoints only require your application `key` and `secret` (for instance, [creating a user](#create-new-user-account)), but most require an OAuth `access_token`.

For endpoints that require an OAuth access token, it should be included in the Authorization HTTP header like so:

`Authorization: Bearer <T0K3N_H3R3>`

<aside class="warning">
Be sure to replace `<T0K3N_H3R3>` with an OAuth `access_token`.
</aside> 

## OAuth2

> Set up your application to properly support callback routes.

```ruby
# For omniauth-plangrade users, here's an example
# Add this to your config/routes.rb
get   '/auth/:provider/callback' => 'static_pages#redirect'

# Add this to config/initializers/omniauth.rb
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :plangrade, ENV['PLANGRADE_CLIENT_ID'], ENV['PLANGRADE_CLIENT_SECRET']
end
```

> Now just make sure that your [plangrade client application](https://plangrade.com/oauth/applications) has the proper Redirect uri specified (e.g. https://mysite.com/auth/plangrade/callback)

plangrade's API lets you interact with a user's plangrade account and act on their behalf to view, create, update, and destroy things like companies, plans, and participants. To do so, your application first needs to request authorization from users.

plangrade implements the [OAuth 2.0 standard](http://oauth.net/2/) to facilitate this authorization. Similar to Facebook and Twitter's authentication flow, the user is first presented with a permission dialog for your application, at which point the user can either approve the permissions requested or reject them. Once the user aproves, an `authorization_code` is sent to your application which will then be [exchanged](#finish-authorization) for an `access_token` and a `refresh_token` pair.

The `access_token` can then be used to make API calls which require user authentication like [Participants](#participants) or [Companies](#companies).

### Token Lifetimes

**Authorization codes** expire *very quickly*: 10 minutes.

**Access tokens** are *short lived*: 2 hours.

**Refresh tokens** are *good forever* (or until revoked by the user).

A refresh token can be used to generate a new `access_token` and `refresh_token` pair. You can [refresh your authorization](#refresh-authorization) to regain active access to a user's account but so long as you store the `refresh_token` you can maintain authorization indefinitely without requiring the user to re-authorize.

## Request Authorization

> Request authorization with a properly generated link

```ruby
# For omniauth-plangrade users, this is trivial
# Add a link or button to start the authentication
link_to "Authorize", "/auth/plangrade"

# As an alternative, use OAuth2
redirect_uri = "https://www.myredirect.com/redirect"
client = OAuth2::Client.new(app_id, secret, :site => "https://plangrade.com")
@auth_path = client.auth_code.authorize_url(:redirect_uri => redirect_uri)

# Then in your corresponding view you can add a link or button
link_to "Authorize", @auth_path
```

To start the OAuth process, construct the initiation URL which the user will visit in order to grant permission to your application. It describes the permissions your application requires (which grants read/write access to companies, plans, and participants), and who the client application is (your application's name).

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

> Once authorized, you need to obtain your `access_token` and `refresh_token` pair.

```ruby
# For omniauth-plangrade, just interpret the omniauth response
auth = request.env["omniauth.auth"]
token = auth["credentials"]["token"]
refresh_token = auth["credentials"]["refresh_token"]

# Alternatively, you can use OAuth2
client = OAuth2::Client.new(ENV['PLANGRADE_CLIENT_ID', 'PLANGRADE_CLIENT_SECRET', :site => "https://plangrade.com")
token = client.auth_code.get_token(params[:code], :redirect_uri => redirect_uri)
```

Once the user returns to your application via the `redirect_uri` you specified, there will be a `code` querystring parameter appended to that URL. Exchange the authorization `code` for an `access_token` and `refresh_token` pair.

### HTTP Request

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

> You can refresh the token in a ruby environment with the second gem available: [plangrade-ruby](https://github.com/topherreynoso/plangrade-ruby). See the plangrade-ruby website for setup instructions.

```ruby
# The example below demonstrates using the plangrade-ruby gem
# Create a plangrade oauth client and refresh the token
client = Plangrade::OAuth2Client.new(ENV['PLANGRADE_CLIENT_ID'], ENV['PLANGRADE_CLIENT_SECRET'])
token_pair = JSON.parse client.refresh!(saved_refresh_token).body
new_token = token_pair["access_token"]
new_refresh_token = token_pair["refresh_token"]

# Alternatively, if you used the OAuth2 client, call this on your token pair
new_token_pair = JSON.parse token_pair.refresh!.body
new_token = token_pair["access_token"]
new_refresh_token = token_pair["refresh_token"]
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

> In ruby environments, we recommend using the [plangrade-ruby](https://github.com/topherreynoso/plangrade-ruby) gem. See the plangrade-ruby website for setup instructions.

### Attributes

Parameter | Type | Optional? | Read Only? | Description
--------- | ---- | --------- | ---------- | -----------
`id` | Integer| No | Yes | User's plangrade ID
`name` | String | No | No | User's name
`email` | String | No | No | User's email address

## Current User Information

```ruby
# Retrieve the user's information
@user = Plangrade.current_user
```

> If successful, you'll receive either a response similar to this one or a list of errors:

```ruby
{
  "id" => 1
  "name" => "Compliance Man"
  "email" => "compliance@plangrade.com"
}
```

Retrieve the current authorized user's information.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`GET https://plangrade.com/api/v1/me`



## Create New User Account

```ruby
# Create a new user account with `name` and `email`
new_user = Plangrade.create_user(params)
```

> If successful, you'll receive either a response like this one or a list of errors:

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

## Update User Account Info

```ruby
# Update account information (only for the authorized user)
updated_user = Plangrade.update_user(id, params)
```

> If successful, you'll receive either a response similar to this one or a list of errors:

```ruby
{
  "id" => 1
  "name" => "Compliance Man"
  "email" => "compliance@plangrade.com"
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
# Delete a user account (authorized user only)
Plangrade.delete_user(id)
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

> In ruby environments, we recommend using the [plangrade-ruby](https://github.com/topherreynoso/plangrade-ruby) gem. See the plangrade-ruby website for setup instructions.

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
company_id = Plangrade.create_company(params)
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
company = Plangrade.get_company(id)
```

> If successful, you'll receive this response:

```ruby
{
  "id" => 1
  "ein" => "123456789"
  "name" => "plangrade"
  "grade" => "A+"
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
companies = Plangrade.all_companies(params)
```

> If successful, you'll receive this response:

```ruby
{
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
updated_company = Plangrade.update_company(id, params)
```

> If successful, you'll receive either this response or a list of errors:

```ruby
{
  "id" => 1
  "ein" => "123456789"
  "name" => "plangrade, LLC"
  "grade" => "A+"
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
Plangrade.delete_company(id)
```

> If successful, you'll recieve a "204 No Content" response.

Delete one of the authenticated user's companies.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`DELETE https://plangrade.com/api/v1/companies/{id}`

<aside class="warning">
Deleting a company will result in deleting all of its information as well. This includes plans, participants, documents, its history and any distributions. This action cannot be reversed.
</aside>

# Participants

> In ruby environments, we recommend using the [plangrade-ruby](https://github.com/topherreynoso/plangrade-ruby) gem. See the plangrade-ruby website for setup instructions.

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
participant_id = Plangrade.create_participant(params)
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

You may optionally include a valid `email` address and/or an SMS enabled `phone` number (10 numbers only). This enables plangrade to send the participant electronic authorizations and notices. Additionally, a `street2` may be included, if necessary.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`POST https://plangrade.com/api/v1/participants`

## Get Participant Info

```ruby
# Fetch a participant's information
participant = Plangrade.get_participant(id)
```

> If successful, you'll receive this response:

```ruby
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
participants = Plangrade.all_participants(:company_id => 1, params)
```

> If successful, you'll receive this response:

```ruby
{
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
}
```

Retrieve information about all of the participants in an authorized user's company.

### Mandatory

You must include a `company_id` parameter in order to allow plangrade to identify which company you want to query and to verify that the authorized user has access to this company.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`GET https://plangrade.com/api/v1/participants?company_id={id}`

## Update Participant Info

```ruby
# Update participant information
updated_participant = Plangrade.update_participant(id, params)
```

> If successful, you'll receive either a response like this one or a list of errors:

```ruby
{
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
```

Update a participant's information. You must include an `employee_id` for every participant update request, even if the `employee_id` is not changing.

### Employees

Remember to make the `employee_id` equivalent to the participant's `id`.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`PUT https://plangrade.com/api/v1/participants/{id}`

## Archive Participant

```ruby
archived_participant_id = Plangrade.archive_participant(id)
```

> If successful, you'll receive either a response like this one or a list of errors:

```ruby
{
  "id" => 1
}
```

Non-destructively archive a participant from one of the authorized user's companies.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`GET https://plangrade.com/api/v1/participants/archive`

<aside class="warning">
Archiving a participant will remove the participant and any dependents from regular reports and distributions. The participant and their dependents (if applicable) will still be accessible through the archive along with their history of authorizations, distributions, and notices. This action can be reversed.
</aside>

## Delete Participant

```ruby
# Delete a company's participant
Plangrade.delete_participant(id)
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

# Activities

> In ruby environments, we recommend using the [plangrade-ruby](https://github.com/topherreynoso/plangrade-ruby) gem. See the plangrade-ruby website for setup instructions.

### Attributes

&nbsp;&nbsp;&nbsp;Parameter&nbsp;&nbsp;&nbsp; | Type | Optional? | Read Only? | Description
--------------- | ---- | --------- | ---------- | -----------
`id` | Integer| No | Yes | Activity's plangrade ID
`name` | String | No | Yes | Name of the activity
`start_time` | DateTime | No | Yes | Date and time that the activity is scheduled to occur

## Get All Company's Activities

```ruby
# Fetch all activities associated with a company
activities = Plangrade.all_activities(:company_id => 1, params)
```

> If successful, you'll receive this response:

```ruby
{
  {
    "id" => 1
    "name" => "Distribute Summary Plan Description"
    "start_time" => "1984-12-30"
  },
  {
    "id" => 2
    "name" => "Distribute Open Enrollment"
    "start_time" => "1986-04-18"
  },
  {
    "id" => 3
    "name" => "Distribute Creditable Coverage"
    "start_time" => "2006-11-22"
  }
}
```

Retrieve information about all of the activities in an authorized user's company.

### Mandatory

You must include a `company_id` parameter in order to allow plangrade to identify which company you want to query and to verify that the authorized user has access to this company.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`GET https://plangrade.com/api/v1/activities?company_id={id}`

# Notices

> In ruby environments, we recommend using the [plangrade-ruby](https://github.com/topherreynoso/plangrade-ruby) gem. See the plangrade-ruby website for setup instructions.

### Attributes

&nbsp;&nbsp;&nbsp;Parameter&nbsp;&nbsp;&nbsp; | Type | Optional? | Read Only? | Description
--------------- | ---- | --------- | ---------- | -----------
`id` | Integer| No | Yes | Notice's plangrade ID
`name` | String | No | Yes | Name of the notice
`plan_name` | String | No | Yes | Name of the plan associated with this notice
`link` | String | No | Yes | URL for the PDF associated with this notice
`created_at` | DateTime | No | Yes | Date and time that the notice's latest version was created

## Get All Company's Notices

```ruby
# Fetch all notices associated with a company
notices = Plangrade.all_notices(:company_id => 1, params)
```

> If successful, you'll receive this response:

```ruby
{
  {
    "id" => 1
    "name" => "Summary Plan Description"
    "plan_name" => "ABC Corp Health and Welfare Benefits Plan"
    "link" => "https://plangrade.com/iterations/1234.pdf"
    "created_at" => "1984-12-30"
  },
  {
    "id" => 2
    "name" => "Plan Documents"
    "plan_name" => "ABC Corp Health and Welfare Benefits Plan"
    "link" => "https://plangrade.com/iterations/2345.pdf"
    "created_at" => "1984-12-30"
  },
  {
    "id" => 3
    "name" => "Hiring Notice"
    "plan_name" => "ABC Corp Health and Welfare Benefits Plan"
    "link" => "https://plangrade.com/iterations/3456.pdf"
    "created_at" => "1984-12-30"
  }
}
```

Retrieve information about all of the notices in an authorized user's company.

### Mandatory

You must include a `company_id` parameter in order to allow plangrade to identify which company you want to query and to verify that the authorized user has access to this company.

<aside class="notice">
This endpoint requires an OAuth access token.
</aside>

### HTTP Request

`GET https://plangrade.com/api/v1/notices?company_id={id}`