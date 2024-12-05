# Frontify Service Handler

Frontify is a third party marketing document generation application. The frontify uses a GraphQL interface for its API, so call will look a little different.

The following query will get a list of Brands that are available on Frontify. GraphQL has 3 top level methods Query, Variables and Mutations.

* Query - as the name suggests allows you to build queries, where you specify the exact data you wish it to return.
* Variables - Allows you to perform key value substitution.
* Mutations - Are calls that are equivalent of Put or Post, and allow you to modify the data.

```
{
  "query": "query ListBrands { brands {id, name, avatar, slug} }",
  "variables": {}
}
```

## Security

All calls to Frontify are secured with OAuth2. As there is currently on ly one Frontify Environment

### Terms

|          Term         | Description                                                                                                                                                      |
| :-------------------: | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|          PKCE         | The Authorization Code Flow + PKCE is an OpenId Connect flow specifically designed to authenticate native or mobile application users.                           |
|        OAuth2.0       | OAuth2.0 is an industry-standard protocol for authorization                                                                                                      |
|          JSON         | JavaScript Object Notation (JSON), is a lightweight format for storing and transporting data.                                                                    |
| Bearer Authentication | Bearer authentication (also called token authentication) is an HTTP authentication scheme that involves a security token called bearer token.                    |
|      Bearer Token     | Bearer Tokens are the predominant type of access token used with OAuth 2.0. In our documentation we use the terms Bearer Token and Access Token interchangeable. |

Most requests to GraphQL require an authenticated user.

* Private developer token
* Oauth 2

The token MUST be included in the authorization header (Authorization: Bearer your-token) when making any requests to GraphQL.

All access tokens in Frontify are issued in the name of a user and give you access to at most the permissions assigned to that specific user.

### Developer Token

This is a long lived persistent token, that is generated and assigned to a user or application.

Use-Cases:

* Synchronize Assets to Frontify
* Private applications not requiring user-based authentication
* During development
* Usage in CI/CD, for example in a Github Action

#### Generation

To generate a Personal Developer token, navigate to https://domain/api/developer/token in your browser. Once generated, a developer can use the token until it is manually revoked

When creating a new token, give it a meaningful name. This is helpful if you later need to revoke a token and for you to keep track of where a given token is used.

Make sure to only select the needed scopes, to limit misuse of the token.

### OAuth2

OAuth 2.0 is an authorization framework widely used for delegated authorization and access control in API-based systems. It allows users to grant limited access to their resources on Frontify to another application without sharing their credentials. OAuth2 access tokens are shorter lived typically 1 week to 3 months.

#### Generation

Frontify uses a version of OAuth2 which include PKCE (Proof Key Code Exchange)

**Frontify URLs**

| Endpoint               | HTTP Method | URL                                  |
| ---------------------- | ----------- | ------------------------------------ |
| Authorization Endpoint | GET         | https://domain/api/oauth/authorize   |
| Access Token Endpoint  | POST        | https://domain/api/oauth/accesstoken |
| Refresh Endpoint       | POST        | https://domain/api/oauth/refresh     |
| Revoke Endpoint        | POST        | https://domain/api/oauth/revoke      |

**For detailed instructions please see** [Frontify Authentication](https://developer.frontify.com/d/xJoA5nhTq1AT/graphql-api#/access-control/authentication)

## Frontify API

For all Frontify API information, I recommend you look at the frontify developer site http://Developer.Frontify.com

It has a GraphQL playground where you can test your queries.

## List of Templates Available to Hub

```plantuml

start
    : Get List of Templates;
    : Filter list of Templates;
    : Create response object;
    repeat
        : Create Template object;
        : Add Description to Template Object;
        : Add Key and Type infomation to Template Object;
        : Add Template Object to response Object;
    repeat while (more templates) is (yes) not (no)

    : return Array of templates;
stop
```

Queries you will need to use include :-

1. List Templates By Brand This will give you a list of templates available

```
{
	"query": "query listTemplatesByBrand($id: ID!) 
        { 
            brand(id: $fred) 
            { 
                name creativeTemplates(limit: 50) 
                { 
                    items 
                    { 
                        id 
                        name 
                        description
                    } 
                } 
            }
        }",
	"variables": {
		"id": "Brand Id"
	}
}
```

1. Get Template By ID

```
{
    "query": "query getCreativeTemplateById($id: ID!) 
    { 
        creativeTemplate(id: $id) 
        { 
            id 
            name 
            variables 
            { 
                key 
                type 
                value 
            } 
        }
    }",
    "variables": {
        "id": "Id of template datails you need"
    }
}
```

## Upload Files and Create Assets

This applies to both the Image and CSV assets. Files can only be used once a asset has been created.

```plantuml
participant Service as svc
participant Frontify as fnt

group Asset Upload
    svc -> fnt : Initialize File Upload
    fnt --> svc : FileId
    svc -> fnt : Upload binary image
    svc -> fnt : Create Image Asset
    fnt --> svc : Asset Id
end
```

Please refer to this page :- https://developer.frontify.com/document/2570#/deep-dive/upload-file-create-asset

## Auto Generation Brochures

\*\*N.B. Currently this a BETA feature, Use of Webhooks needs to be investigated.

The ExportCreative operation passes all the text and image ids to a template and executes the brochure creation. Currently there is a polling interface to check when the brochure has been completed, But this will change to the webhook style.

```plantuml
participant Service as svc
participant Frontify as fnt

group Auto Generate Brochure
    svc -> fnt ++ #gold: Export Creative 
    fnt --> svc --
    loop
        svc -> fnt ++ #gold: Poll Status
        fnt -> svc --
    end

    alt Webhook
        svc -> fnt ++ #gold: Register Webhook
        fnt --> svc --
        ====
        fnt -> svc ++ #gold: Status Complete
        svc --> fnt --
        svc -> fnt ++ #gold: Get Brochure
        fnt --> svc --:
    end

end
```

Please Refer to this page :- https://developer.frontify.com/document/2570#/deep-dive/template-data-fetching-and-exporting

Access token : GNAccessToken - skJVZko3Eeyt2DYzTq29wuAaRToVHTm65mapnw9N Graphgl to Json formatter : https://datafetcher.com/graphql-json-body-converter
