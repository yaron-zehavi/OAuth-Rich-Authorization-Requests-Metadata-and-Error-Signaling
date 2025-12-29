---
title: "OAuth 2.0 Rich Authorization Requests Metadata and Error Signaling"
abbrev: "OAuth RAR Metadata and Error Signaling"
category: std

docname: draft-zehavi-oauth-rar-metadata-and-error-signaling-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - RAR
 - Step-up
 - oauth
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "https://github.com/yaron-zehavi/OAuth-Rich-Authorization-Requests-Metadata-and-Error-Signaling"
  latest: "https://yaron-zehavi.github.io/OAuth-Rich-Authorization-Requests-Metadata-and-Error-Signaling/draft-zehavi-oauth-rar-metadata-and-error-signaling.html"

author:
 -
    fullname: Yaron Zehavi
    organization: Raiffeisen Bank International
    email: yaron.zehavi@rbinternational.com
 -
    fullname: Aaron Parecki
    organization: Okta
    email: aaron@parecki.com

normative:
  RFC6749:
  RFC8414:
  RFC9396:
  RFC9728:
  IANA.oauth-parameters:
  JSON.Schema:
    title: JSON Schema: A Media Type for Describing JSON Documents
    target: https://json-schema.org/draft/2020-12/json-schema-core
    date: June 16, 2022
    author:
      - ins: A. Wright, Ed.
      - ins: H. Andrews, Ed.
      - ins: B. Hutton, Ed. Postman
      - ins: G. Dennis

--- abstract

OAuth 2.0 Rich Authorization Requests (RAR), as defined in {{RFC9396}}, enables clients to request fine-grained authorization using structured JSON objects. While RAR {{RFC9396}} standardizes the exchange and handling of authorization details, it does not define a mechanism for clients to discover how to construct valid authorization details types.

This document defines a machine-readable metadata structure for advertising authorization details type documentation and JSON Schema {{JSON.Schema}} definitions via OAuth Authorization Server Metadata {{RFC8414}} and OAuth Resource Server Metadata {{RFC9728}}. In addition, this document defines a new OAuth error code, `insufficient_authorization_details`, enabling resource servers to return actionable authorization details information to clients.

--- middle

# Introduction

OAuth 2.0 Rich Authorization Requests (RAR) {{RFC9396}} allow OAuth clients to request structured, fine-grained authorization beyond simple scopes. This has enabled advanced authorization models across domains such as open banking & API marketplaces, and is well positioned to be used for authorizing AI agent state-changing actions.

However, RAR {{RFC9396}} does not specify how a client discovers the structure of supported authorization_detail types and how to construct syntactically valid authorization details.

As a result, clients must rely on out-of-band documentation or static ecosystem profiles, limiting interoperability and preventing dynamic client behavior.

This document addresses this gap by defining:

* A metadata structure for authorization details types, containing both human-readable documentation as well as embedded JSON Schema {{JSON.Schema}} definitions.
* Discovery through Authorization Server Metadata {{RFC8414}}, as well as via OAuth 2.0 Protected Resource Metadata {{RFC9728}}.
* A standardized error signaling mechanism allowing resource servers to return an authorization details object, to be included in a new Auth request, in order to accomplish a specific request.

It is up to implementers to decide if their clients need to learn to construct valid authorization details objects, and if so whether authorization details types metadata should be provided by authorization servers or by resource servers.

Alternatively they may decide clients do not need to learn to construct valid authorization details objects, and interoperability shall be achieved by resource servers providing required authorization_details in their error signals, which clients then include in subsequent OAuth requests.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Protocol Overview

There are two main proposed flows, which may be combined:

* Client learns to construct valid authorization details objects from authorization details types metadata provided by authorization servers, resource servers or both.
* Resource servers provide clients as part of an error response, the precise authorization details object, whose inclusion in subsequent OAuth requests is required to accomplish a specific request.

## Client obtains authorization details types metadata

~~~ ascii-art
                                                +--------------------+
                                                |   Authorization    |
                          (B)Native             |      Server 1      |
             +----------+ Authorization Request |+------------------+|
(A)User  +---|          |---------------------->||     Native       ||
   Starts|   |          |                       ||  Authorization   ||
   Flow  +-->|  Client  |<----------------------||    Endpoint      ||
             |          | (C)Federate Error     |+------------------+|
             |          |        Response       +--------------------+
             |          |         :
             |          |         :             +--------------------+
             |          |         :             |   Authorization    |
             |          | (D)Native             |      Server 2      |
             |          | Authorization Request |+------------------+|
             |          |---------------------->||     Native       ||
             |          |                       ||  Authorization   ||
             |          |<----------------------||    Endpoint      ||
             |          | (E) Redirect to       |+------------------+|
             |          |     App Response      +--------------------+
             |          |         :
             |          |         :             +--------------------+
             |          | (F) Invoke App        |                    |
             |          |---------------------->|   Native App of    |
             |          |                       |   Auth Server 2    |
             |          |<----------------------|                    |
             |          | (G)Authorization code +--------------------+
             |          |   For Auth Server 1
             |          |         :             +--------------------+
             |          |         :             |   Authorization    |
             |          | (H)Authorization Code |      Server 1      |
             |          |    For Auth Server 1  |+------------------+|
             |          |---------------------->||    Response      ||
             |          |                       ||       Uri        ||
             |          |<----------------------||    Endpoint      ||
             |          | (I) Authorization     |+------------------+|
             |          |     Code Response     |                    |
             |          |         :             |                    |
             |          |         :             |                    |
             |          | (J) Token Request     |+------------------+|
             |          |---------------------->||      Token       ||
             |          |                       ||     Endpoint     ||
             |          |<----------------------||                  ||
             |          | (K) Access Token      |+------------------+|
             |          |                       +--------------------+
             |          |
             +----------+
~~~
Figure: Native client federated, then redirected to app

- (A) The client starts the flow.
- (B) The client initiates the authorization request by making a POST request to the Native Authorization Endpoint of Authorization Server 1.
- (C) Authorization Server 1 decides to federate the user to Authorization Server 2. To do so it contacts Authorization Server 2's PAR endpoint, then returns the `federate` error code together with the *federation_uri*, *federation_body*, *response_uri* and *auth_session* response attributes.
- (D) The client calls Authorization Server 2's Native Authorization Endpoint, as instructed by Authorization Server 1.
- (E) Authorization Server 2 decides to use a native app, and therefore responds with the `redirect_to_app` error code together with the *deep_link* response attribute.
- (F) The client invokes the app using the deep_link.
- (G) The app interacts with user and if satisifed, returns an authorization code, regarded as Authorization Server 2's response to Authorization Server 1's federation to it.
- (H) The client provides the authorization code to Authorization Server 1.
- (I) Authorization Server 1 returns an authorization code.
- (J) The client sends the authorization code received in step (I) to obtain a token from the Token Endpoint.
- (K) Authorization Server 1 returns an Access Token from the Token Endpoint.

## Client obtains authorization details object

Here comes another drawing.

# Authorization Details Type Metadata

## Overview

This document defines a new metadata attribute, `authorization_details_types_metadata`, which provides documentation and validation information for authorization details types used with Rich Authorization Requests {{RFC9396}}.

The metadata MAY be published by OAuth Authorization Servers using Authorization Server Metadata {{RFC9126}} as well as by OAuth Resource Servers using Resource Server Metadata {{RFC9728}}.

Clients MAY use this metadata to dynamically construct valid `authorization_details` objects.

## Metadata Location

The `authorization_details_types_metadata` attribute may be included in:

* Authorization Server Metadata {{RFC8414}}, to describe authorization details types supported by the authorization server. If present, its keys MUST be a subset of the values listed in `authorization_details_types_supported` as defined in {{RFC9396}}.
* OAuth 2.0 Protected Resource Metadata {{RFC9728}}, to describe authorization details types accepted by the protecred resource.

## Metadata Structure

The `authorization_details_types_metadata` attribute is a JSON object whose keys are authorization details type identifiers. Each value is an object describing a single authorization details type.

```json
{
  "authorization_details_types_metadata": {
    "<type>": {
      "version": "...",
      "description": "...",
      "documentation_uri": "...",
      "schema": { },
      "schema_uri": "...",
      "examples": [ ]
    }
  }
}

## Metadata Attributes

"version":
: OPTIONAL. String identifying the version of the authorization details type definition. The value is informational and does not imply semantic version negotiation.

"description":
: OPTIONAL. String containing a human-readable description of the authorization details type. Clients MUST NOT rely on this value for authorization or validation decisions.

"documentation_uri":
: OPTIONAL. URI referencing external human-readable documentation describing the authorization details type.

"schema":
: The `schema` attribute is a JSON Schema document {{JSON.Schema}} describing a single authorization detail object. The schema MUST validate a single authorization detail object and MUST constrain the `type` attribute to the authorization detail type identifier. This attribute is REQUIRED unless `schema_uri` is specified. If this attribute is present, `schema_uri` MUST NOT be present.

"schema_uri":
: The `schema_uri` attribute is an absolute URI, as defined by RFC 3986 {{RFC3986}}, referencing a JSON Schema document describing a single authorization details object. The referenced schema MUST satisfy the same requirements as the `schema` attribute. This attribute is REQUIRED unless `schema` is specified. If this attribute is present, `schema` MUST NOT be present.

"examples":
: OPTIONAL. An array of example authorization details objects. Examples are non-normative.

## Metadata Examples

### Example Authorization Detail Type Metadata: Payment Initiation

```json
{
  "authorization_details_types_supported": ["payment_initiation"],
  "authorization_details_types_metadata": {
    "payment_initiation": {
      "version": "1.0",
      "description": "Authorization to initiate a single payment.",
      "documentation_uri": "https://example.com/docs/payment-initiation",
      "schema": {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "type": "object",
        "required": ["type", "instructed_amount", "creditor_account"],
        "properties": {
          "type": { "const": "payment_initiation" },
          "instructed_amount": {
            "type": "object",
            "required": ["currency", "amount"]
          },
          "creditor_account": {
            "type": "object",
            "required": ["iban"]
          }
        },
        "additionalProperties": false
      }
    }
  }
}

# Resource Server Error Signaling with insufficient_authorization_details

## Overview

This document defines a new OAuth error code, `insufficient_authorization_details`, for use when access is denied due to missing or insufficient authorization details.

## Error Definition

The error MUST be conveyed using the `WWW-Authenticate` header and MUST include an `authorization_details` parameter.

## authorization_details Error Parameter

The parameter MUST contain a JSON object or array, representing the required authorization details, whose inclusion in a subsequent OAuth request is required to satisfy the resource server's requirements for this specific request.
The value MUST be base64url-encoded.

```http
WWW-Authenticate: Bearer error="insufficient_authorization_details",
 authorization_details="{JSON or base64url-encoded JSON of required RAR}"

# Security Considerations {#security-considerations}

## Non First-Party applications of federated authorization servers {#first-party-applications}

A federated authorization server should consider end-user's privacy and security
to determine if it should present authorization challenges in federation scenarios.
For example, it can label **federating** clients as such and avoid serving them
authorization challenges, as user-serving clients receiving those challenges are
not first party clients.

# IANA Considerations

## OAuth Parameters Registration

IANA has (TBD) registered the following values in the IANA "OAuth Parameters" registry of {{IANA.oauth-parameters}} established by {{RFC6749}}.

**Parameter name**: `native_callback_uri`

**Parameter usage location**: Native Authorization Endpoint

**Change Controller**: IETF

**Specification Document**: Section 5.4 of this specification

--- back

# Example User Experiences

This section provides non-normative examples of how this specification may be used to support specific use cases.

## Native client federated and redirected to app

### Diagram

~~~ ascii-art
                                                +--------------------+
                                                |   Authorization    |
                          (B)Native             |      Server 1      |
             +----------+ Authorization Request |+------------------+|
(A)User  +---|          |---------------------->||     Native       ||
   Starts|   |          |                       ||  Authorization   ||
   Flow  +-->|  Client  |<----------------------||    Endpoint      ||
             |          | (C)Federate Error     |+------------------+|
             |          |        Response       +--------------------+
             |          |         :
             |          |         :             +--------------------+
             |          |         :             |   Authorization    |
             |          | (D)Native             |      Server 2      |
             |          | Authorization Request |+------------------+|
             |          |---------------------->||     Native       ||
             |          |                       ||  Authorization   ||
             |          |<----------------------||    Endpoint      ||
             |          | (E) Redirect to       |+------------------+|
             |          |     App Response      +--------------------+
             |          |         :
             |          |         :             +--------------------+
             |          | (F) Invoke App        |                    |
             |          |---------------------->|   Native App of    |
             |          |                       |   Auth Server 2    |
             |          |<----------------------|                    |
             |          | (G)Authorization code +--------------------+
             |          |   For Auth Server 1
             |          |         :             +--------------------+
             |          |         :             |   Authorization    |
             |          | (H)Authorization Code |      Server 1      |
             |          |    For Auth Server 1  |+------------------+|
             |          |---------------------->||    Response      ||
             |          |                       ||       Uri        ||
             |          |<----------------------||    Endpoint      ||
             |          | (I) Authorization     |+------------------+|
             |          |     Code Response     |                    |
             |          |         :             |                    |
             |          |         :             |                    |
             |          | (J) Token Request     |+------------------+|
             |          |---------------------->||      Token       ||
             |          |                       ||     Endpoint     ||
             |          |<----------------------||                  ||
             |          | (K) Access Token      |+------------------+|
             |          |                       +--------------------+
             |          |
             +----------+
~~~
Figure: Native client federated, then redirected to app

### Client makes initial request and receives "federate" error

Client calls the native authorization endpoint and includes the *native_callback_uri* parameter:

    POST /native-authorization HTTP/1.1
    Host: as-1.com
    Content-Type: application/x-www-form-urlencoded

    client_id=t7CieSlru4&native_callback_uri=
    https://client.example.com/cb

The first authorization server, as-1.com, decides to federate to as-2.com after validating
it supports the native authorization endpoint. If it does not, as-1.com returns:

    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
        "error": "native_authorization_federate_unsupported"
    }

If native authorization endpoint is supported by the federated authorization server,
as-1.com performs a PAR request to as-2.com's pushed authorization endpoint,
including the original *native_callback_uri*:

    POST /par HTTP/1.1
    Host: as-2.com
    Content-Type: application/x-www-form-urlencoded

    client_id=s6BhdRkqt3&native_callback_uri=
    https://client.example.com/cb

as-1.com receives a request_uri from as-2.com's PAR endpoint, which it
includes in its response to client, in the *federation_body* attribute:

    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
        "error": "federate",
        "auth_session": "ce6772f5e07bc8361572f",
        "response_uri": "https://as-1.com/native-authorization",
        "federation_uri": "https://as-2.com/native-authorization",
        "federation_body": "client_id=s6BhdRkqt3&request_uri=
          urn:ietf:params:oauth:request_uri:R3p_hzwsR7outNQSKfoX"
    }

See {{federating-response}} for more details.

### Client calls federated authorization server and is redirected to app

Client calls the *federation_uri* it got from as-1.com using HTTP POST with
*federation_body* as application/x-www-form-urlencoded request body:

    POST /native-authorization HTTP/1.1
    Host: as-2.com
    Content-Type: application/x-www-form-urlencoded

    client_id=s6BhdRkqt3&request_uri=
    urn:ietf:params:oauth:request_uri:R3p_hzwsR7outNQSKfoX

as-2.com decides to use its native app to interact with end-user and responds:

    HTTP/1.1 400 Bad Request
    Content-Type: application/json

    {
        "error": "redirect_to_app",
        "deep_link": "https://as-2.com/native-authorization?
          client_id=s6BhdRkqt3&request_uri=
          urn:ietf:params:oauth:request_uri:R3p_hzwsR7outNQSKfoX"
    }

Client locates an app claiming the obtained deep_link and invokes it.
See {{redirect-to-app-response}} for more details.
The invoked app handles the native authorization request and then natively invokes native_callback_uri:

    https://client.example.com/cb?authorization_code=uY29tL2F1dGhlbnRpY

### Client calls federating authorization server

Client invokes the response_uri of as-1.com as it is the authorization server which federated
it to as-2.com and the app's response is regarded as the response of as-2.com:

    POST /native-authorization HTTP/1.1
    Host: as-1.com
    Content-Type: application/x-www-form-urlencoded

    authorization_code=uY29tL2F1dGhlbnRpY
    &auth_session=ce6772f5e07bc8361572f

And receives in response an authorization code, which it is the audience of (no further federations) to resolve:

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "authorization_code": "vZ3:uM3G2eHimcoSqjZ"
    }

Client proceeds to exchange code for tokens.

# Document History

-00

* Document creation


# Acknowledgments
{:numbered="false"}

The authors would like to thank the attendees of IETFs 123 & 124 in which this was discussed, as well as the following individuals who contributed ideas, feedback, and wording that shaped and formed the final specification: George Fletcher, Arndt Schwenkshuster, Filip Skokan.
