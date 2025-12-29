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
  RFC3986:
  RFC6749:
  RFC8414:
  RFC9396:
  RFC9728:
  IANA.oauth-parameters:
  JSON.Schema:
    title: "JSON Schema: A Media Type for Describing JSON Documents"
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

## Client obtains authorization details object

Here comes another drawing.

# Authorization Details Type Metadata

## Overview

This document defines a new metadata attribute, `authorization_details_types_metadata`, which provides documentation and validation information for authorization details types used with Rich Authorization Requests {{RFC9396}}.

The metadata MAY be published by OAuth Authorization Servers using Authorization Server Metadata {{RFC8414}} as well as by OAuth Resource Servers using Resource Server Metadata {{RFC9728}}.

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

# Document History

-00

* Document creation


# Acknowledgments
{:numbered="false"}
