---
title: Provisioning Fly.io Extensions
layout: docs
sitemap: false
nav: firecracker
---

*This document is a mere proposal - not set in stone. Please give us feedback!*

Read our [Extensions Program overview](https://fly.io/docs/about/extensions) before digging into this doc.

## Provisioning Flow

Fly.io is a CLI-first platform. So provisioning extensions starts there. Either when launching an app, or when a customer explicitly asks to provision an extension.

We'll forward this request on to you, along with details about the provisioning Fly.io organization. You should reply synchronously with details about your provisioned extension. We'll attach these details to a target application, where applicable, as secrets exposed as environment variables in VMs.

## Provisioning Accounts and Organizations

We'd like customers to have full access to your platform features through their Fly.io-provisioned account. And you should be able to communicate with customers via email. To that end, all provisioning requests are acccompanied by:

* A unique user ID and working email alias
* A unique organization ID and working email alias that emails org admins

Our SSO flow allows you to fetch user and organization data, to determine whether the user has access to a given resource.

You should provision, at least:

* An account to own extensions provisioned through Fly.io
* An organization/team, where applicable, to match the Fly.io organization
* One or more administrative users as account owners and SSO targets

We recommend that for each resource provisioning or SSO request, you provision the above inline, idempotently. This avoid the need for a separate account provisioning API.

## Resource Provisioning API

This section covers describes a REST-based API for provisioning individual extension resources.

### Authentication and Request Signing

Given the high privilege afforded to us by your platform, we sign all requests with a shared secret and a SHA256 HMAC. We require that providers verify these signatures.

All requests are accompanied by an `X-Signature` header containing the HMAC of the request contents: the query string for `GET` and `DELETE` requests, and the request body for `POST` and `PATCH` requests.

We add the following parameters to help you verify the authenticity of requests. We recommend at least verifying that the signed URL matches the request URL.

```
timestamp: A UNIX epoch timestamp value
nonce: A random unique string identifying the individual request
url: The full request target URL
```

### Request Base URL

We recommend that your partner API builds upon base URL, like `https://logjam.io/flyio`. You can append then the paths shown below as REST resources.

### Resource provisioning

Here's an example provisioning request.

```javascript
POST https://logjam.io/flyio/extensions

{
  name: "test",
  id: "test",
  organization_id: "04La2mblTaz",
  organization_name: "High Flyers",
  organization_email: "04La2mblTaz@customer.fly.io",
  user_email: "v9WvKokd@customer.fly.io",
  user_id: "NeBO2G0l0yJ6",
  primary_region: "mad",
  ip_address: "fdaa:0:47fb:0:1::1d",
  read_regions: ["syd", "scl"],
  url: "https://logjam.io/flyio/extensions",
  nonce: "e3f7b4c8f5ea016d0d2e92048ca0d856",
  timestamp: 1685641159
}

```

**Parameters**

These parameters are sent with every provisioning request.

| Name | Type | Description | Example |
| --- | --- | --- | --- |
| **name** | string | Name of the extension, unique across the Fly.io platform | `collider-prod` |
| **id** | string | Unique ID representing the extension | `30bqn49y2jh` |
| **organization_id** | string | Unique ID representing an organization | `M03FclA4m` |
| **organization_name** | string | Display name for an organization | `Supercollider Inc` |
| **organization_email** | string | Obfuscated email that routes to all organization admins (does not change) | `n1l330mao@customer.fly.io` |
| **user_id** | string | Unique ID representing an organization | `M03FclA4m` |
| **user_email** | string | Obfuscated email that routes to the provisioning user (does not change) | `n1l330mao@customer.fly.io` |

**Optional parameters**

These parameters would only be sent for extensions that are region-aware or that deploy on Fly.io.

These parameters are sent with every provisioning request.

| Name | Type | Description | Example |
| --- | --- | --- | --- |
| **primary_region** | string | The three-letter, primary Fly.io region where the target app intends to write from | `mad` |
| **read_regions** | array | An array of Fly.io region codes where read replicas should be provisioned | `["mad", "ord"]` |
| **ip_address** | string | An IPv6 address on the customer network assigned to this extension | `fdaa:0:47fb:0:1::1d` |


Your response should contain, at least, a list of key/value pairs of secrets that that should be set on the associated Fly.io application. These secrets are available as environment variables. They aren't visible outside of an application VM.

```javascript
{
  "LOGJAM_URL": "https://user:password@test.logjam.io",
}

```

### Fetching extension data

Im some cases, we'll want to be able to fetch data about your extension. For example, when displaying credentials to users directly via the CLI, we won't store these credentials in our database. We'll pass this request directly to your API.

This endpoint should be discussed on a case-by-base basis.

### Updating Extensions

Customers should be able to update some extensions directly from `flyctl`. For example, adding/removing read replica regions, or switching payment plans.

Your API should support updates using the same parameters as the `POST` request above at:

```
PATCH {base_url}/extensions/{extension_id}
```

### Single Sign On Flow

We want to get users into your platform with as little friction as possible, directly from `flyctl`. Example:

`flyctl ext logjam dashboard -a myapp`

This command should log the customer in to your UI, on the extension detail screen.

The CLI will open the user's browser and send a `GET` request.

`GET {base_url}/extensions/{extension_id}/sso`

If the user is already logged in, you should redirect them to the extension detail screen. If not, you should start an OAuth authorization request to Fly.io, by redirecting to:

```
GET https://api.fly.io/oauth/authorize?client_id=123&response_type=code&redirect_uri=https://logjam.io/flyio/callback&scope=read
```

You should pass the organization ID and desired permissions scope. Currently, only the 'read' scope is supported.

Once we authenticate the user, we'll redirect to your OAUth `redirect_uri` with an authorization code you may exchange for an access token via a POST request.

```
POST https://api.fly.io/oauth/token

client_id=logjam
client_secret=123
grant_type=authorization_code
code=myauthcode
redirect_uri=https://logjam.io/flyio/callback
```

The JSON response:

```
{
  "access_token"=>"fo1__034hk03k4mhjea0l4224hk",
  "token_type"=>"Bearer",
  "expires_in"=>7200,
  "refresh_token"=>"j-elry40hpy05m2qbaptr",
  "scope"=>"read",
  "created_at"=>1683733170
}
 ```

Finally, use this access token to issue an inline request to our [token introspection endpoint](https://www.oauth.com/oauth2-servers/token-introspection-endpoint/). The response gives you enough information, for example, to authorize the user if they belong the correct parent organization in your system, or to provision the user and add them to these organizations.

```
GET https://api.fly.io/oauth/token/info
Authorization: Bearer fo1__034hk03k4mhjea0l4224hk
```

The JSON response:

```
{
  "resource_owner_id"=>"rzjkdw3g0ypx061q",
  "user_id"=>"rzjkdw3g0ypx061q",
  "organization_ids"=>["zd3e5wvkjel6pgqw", "zgyep87w1m4q06d4"],
  "scope"=>["read"],
  "expires_in"=>7200,
  "application"=>{"uid"=>"logjam"},
  "created_at"=>1683740928
}
```

## Deploying your service on Fly.io

Contact us at [extensions@fly.io](mailto:extensions@fly.io) about deploying your service on the Fly.io platform.
