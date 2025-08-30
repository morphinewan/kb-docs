---
title: "Personal Access Tokens \| App Store API · Appfigures"
url: "https\://docs.appfigures.com/api/authentication/personal-access-tokens"
description: "\<null\>"
author: "\<null\>"
site_name: "\<null\>"
language: "\<null\>"
image: "\<null\>"
generated_at: "2025-08-30T01:39:31.191Z"
---

Personal Access Tokens
======================

If you would like to access data only for your account and don’t want to expose your API client or integration to other Appfigures users, you can use a Personal Access Token to authenticate with the API.

A Personal Access Token is an OAuth 2.0 token that you issue yourself manually without having to implement a full OAuth 2.0 flow into your API client. After generating the token, you pass it along with your requests like you would an OAuth 2.0 Bearer token.

Note that this token can’t be directly used in OAuth 1.0a requests, but instead is used as the bearer in OAuth 2.0 requests.

Generating a Personal Access Token
----------------------------------

Each Personal Access Token is associated with an [API Client](https://docs.appfigures.com/api/authentication#api-client), so you will need to create one in your [Appfigures account](https://appfigures.com/developers/keys). Make sure to think about what scopes you’ll need, since you can’t modify the scopes of an existing client and your Personal Access Token will inherit its scopes.

After that, you can open the details of any key in your account and select “Create Personal Access Token”. The token you are given can be used to make API requests on behalf of you and the API Client. Make sure to treat it as you would a password and to save it somewhere safe, as there is no way to see the token again after it is issued.

Using a Personal Access Token
-----------------------------

Assuming we have generated a token `pat_1234` above, we can pass it in the `Authorization` header just as we would with an [OAuth 2.0](oauth-2) Bearer token.

    >> Request
    GET https://api.appfigures.com/v2/
    Authorization: Bearer pat\_1234
    
    << Response
    200 OK
    {
      "status": "200",
      "message": "OK",
      "see": "http://docs.appfigures.com/api",
      "version": "2.0",
      "user": {
        "id": 42,
        "name": "Test User",
        "email": "test@test.com",
        "avatar\_url": "https://secure.gravatar.com/avatar/dbe9de53f77958ee4ff7e19697a58990?d=mm"
      },
      . . . snip . . .
    }

#### Python Sample

import requests
token = "pat\_1234mytoken"
headers = {
  "user-agent": "myclient/1.0",
  "authorization": f"Bearer {token}"
}
resp = requests.get("https://api.appfigures.com/v2/", headers=headers)