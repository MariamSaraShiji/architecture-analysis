# Multi-Cloud AK/SK Management Design

This document serves as the design specification for the Multi-Cloud AK/SK Management in OpenSDS. This proposal was drafted [here](https://docs.google.com/document/d/1P9GEFHIjFyqTn8ZPoBN2bLdeCY98iUOqDGu4kTSWZ9o/edit?usp=sharing).

## Background

Multi-Cloud AK/SK Management supports authenticating requests. When you send requests to Multi-Cloud, you sign the requests so that multi-cloud can identify who sent them. For providing complete control over how a request is sent to the multi-cloud, you sign requests with your access key, which consists of an access key ID and signing key. Requests are then allowed or denied in part based on the identity of the requester.

AK/SK Management helps secure requests in the following ways:

* **Identity Verification of the requester:**
Signing makes sure that the request has been sent via a valid access key requester.

* **In-transit data protection:**
In-transit request is prevented from tempering, some of the request elements are used to calculate a digest i.e. hash  of the request, and the resulting hash value is included as signature within the request. When multi-cloud receives the request, it uses the same information to calculate a hash and matches it against the hash value in your request. If the values don't match, multi-cloud denies the request.


## Objectives

### Goals

* Goal 1: Support keystone application credential management mechanism, provides improved security by limiting access and not exposing the user credentials.

    * Goal 1 - sub goal 1: Support keystone application credential management for the set of authentication credentials i.e. AK/SK.

    * Goal 1 - sub goal 2: Support keystone APIs such as PUT/POST/GET/DELETE for AK/SK.

* Goal 2: Develop multi-cloud AK/SK management mechanism, provides signing and authenticating requests to manage multi-cloud APIs.

    * Goal 2 - sub goal 1: Support a protocol for authenticating inbound API request to multi-cloud. Implement a custom HTTP scheme based on a keyed-HMAC authentication scheme.

    * Goal 2 - sub goal 2: Support signature validations for the inbound API request to multi-cloud.

### Non Goals

* Goal 1: Support role-based authorization for the multi-cloud users for API requests to PUT/POST/GET/DELETE a backend. Assign the set of roles the User has on multi-cloud.


## Design Details

The Multi-Cloud AK/SK Management design is shown in the following diagram:

![Multi-Cloud AK-SK Management Diagram](resources/multicloud_ak-sk_management.PNG?raw=true "Multi-Cloud AK-SK Management Diagram")

Following are the steps for authenticating requests to Multi-Cloud API:

1. Construct a request to API handler.

2. Calculate the signature using signing key.

3. Send the request (include access key ID and signature) to API handler.

4. API Handler uses the access key ID to look up your secret access key from keystone.

5. API Handler calculates a signature from the request data and the signing key using the same algorithm that user used to calculate the signature sent in the request.

6. If the signature generated by API Handler matches with the user requested signature, the request is authentic. Else, the request is discarded and returns an error response.

### AK/SK Credential Management

User sends authentication credentials to keystone, the Identity service from keystone generates and returns a token. A token represents the authenticated identity of a user and, grants authorization. User can PUT/POST/GET/DELETE a credential as [here](https://developer.openstack.org/api-ref/identity/v3/index.html?expanded=list-credentials-detail,show-credential-details-detail,create-credential-detail#application-credentials).

### Signature Calculation and Validation

API requests to multi-cloud containing the authentication information must include a signature which is computed based on the request elements. To calculate a signature, concatenate request elements to form a string (string to sign) and then create a signing key using secret access key. Signing key is derived from the credential scope, which means that there is no need to include the key in the request. Then signing key is used to calculate the hash-based message authentication code (HMAC) of the string to sign.
When API handler receives the request, it performs the same steps to calculate the signature included in the request. API handler then compares its calculated signature to the one sent with the request. If the signatures match, the request is processed. If the signatures don't match, the request is denied.

Following is the illustration for the signature calculation and validation process.

![Multi-Cloud Signature Calculation Diagram](resources/multicloud_signature_calculation.PNG?raw=true "Multi-Cloud Signature Calculation Diagram")

![Multi-Cloud Signature Validation Diagram](resources/multicloud_signature_validation.PNG?raw=true "Multi-Cloud Signature Validation Diagram")

### API Impact on Current Implementation
#### S3 Interface

Add Signature Information to the Authorization Header. Include signature information by adding it to an HTTP Header i.e. Authorization.
```
Authorization: <authorization string>
Authorization: algorithm Credential=accessKeyID/credential scope, SignedHeaders=signedHeaders, Signature=signature
Credential scope request_date/region/service/sign_request
```

**Example Authorization:**
```
Authorization: OPENSDS-HMAC-SHA256 Credential=access_key/20190301/us-east-1/s3/sign_request, SignedHeaders=x-auth-date, Signature=9b1329c71ebc1923a3b7d801f764336790d98f9fc358b76d1ab90450b0daccd1
```

Following are the steps to calculate the Signature.


**Task 1: Create a Canonical Request**

Create a standardized (canonical) request.
```
 CanonicalRequest =
  HTTPRequestMethod + '\n' +
  CanonicalURI + '\n' +
  CanonicalQueryString + '\n' +
  CanonicalHeaders + '\n' +
  SignedHeaders + '\n' +
  HexEncode(Hash(RequestPayload))
```
*Note:* Hash(RequestPayload) produces a SHA-256 body digest. HexEncode returns the base-16 encoding of the digest in lowercase characters.

**Example request**
```
GET https://localhost:8089/v1/s3
Host: localhost:8089
X-Auth-Date:20190306T113400Z
```

Following are the steps to create a canonical request.

1. Add HTTP GET, PUT, POST, etc. request method.

**Example request method**
```
GET
```

2. Add the canonical URI which is the URI-encoded version of the absolute path of the URI. If the absolute path is empty, use a forward slash (/).

**Example canonical URI**
```
/v1/s3
```

3. Add the canonical query string. If the request does not include a query string, use an empty string.

Steps to construct the canonical query string:
Sort the query parameter names in ascending order.
Append the URI-encoded parameter name, followed by (=) and URI-encoded parameter value. Use an empty string for parameters that have no value.
Append (&) after each parameter value, except for the last value in the list.

4. Add the canonical header i.e. the list of all HTTP headers included with the signed request.

**Example canonical headers**
```
x-auth-date:20190306T113400Z\n
```

To create the canonical headers list, convert all header names to lowercase and remove leading spaces and trailing spaces and extra spaces if any.

```
CanonicalHeaders =CanonicalHeadersEntry0 + CanonicalHeadersEntry1 + ... + CanonicalHeadersEntryN
CanonicalHeadersEntry = Lowercase(HeaderName) + ':' + TrimExtraSpaces(HeaderValue) + '\n'
```

Lowercase represents a function that converts all characters to lowercase. The TrimExtraSpaces function removes excess white space before and after values.

5. Add the signed headers i.e. the list of headers included in the canonical headers.
To create the signed headers list, convert all header names to lowercase, sort them by character code, and use a semicolon to separate the header names.
```
SignedHeaders = Lowercase(HeaderName0) + ';' + Lowercase(HeaderName1) + ";" + ... + Lowercase(HeaderNameN)
```
**Example signed headers**
```
x-auth-date
```

6. Create SHA256 payload digest included in the body of the request.

**Example structure of payload**

```
HashedPayload = Lowercase(HexEncode(Hash(requestPayload)))
```
If the payload is empty, use an empty string as the input to the hash function.

**Example hashed payload (empty string)**
```
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

7. Combine all the above components with a newline character.

**Example canonical request**

```
GET
/v1/s3

x-auth-date:20190306T113400Z

x-auth-date
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

8. Create a hash of the canonical request.

**Example hashed canonical request**
```
c2db941c7e3495e327ec6c4bac66e2c318451c3768d2ae7fc5d2aae792143169
```

**Task 2: Create a String to Sign**

The string to sign includes concatenated string the algorithm, date and time, credential scope, and digest of the canonical request.

Structure of string to sign
```
StringToSign =
	Algorithm + \n +
	RequestDateTime + \n +
	CredentialScope + \n +
	HashedCanonicalRequest
```

To create the string to sign

1. Define an algorithm string.
```
OPENSDS-HMAC-SHA256
```
2. Append request date time value specified with ISO8601 format (YYYYMMDD'T'HHMMSS'Z') in the x-auth-date header.
```
20150830T123600Z
```

3. Append the credential scope value. The region and service name strings must be UTF-8 encoded.
```
request_date/region/service/sign_request
```
**Note:** The date must be in the YYYYMMDD format.

4. Append the hash of the canonical request that you created in Task 1.
```
c2db941c7e3495e327ec6c4bac66e2c318451c3768d2ae7fc5d2aae792143169
```
5. Combine elements to create canonical request

**Example string to sign**

```
OPENSDS-HMAC-SHA256
20190306T113400Z
20190306/ap-south-1/s3/sign_request
c2db941c7e3495e327ec6c4bac66e2c318451c3768d2ae7fc5d2aae792143169
```

**Task 3: Calculate the Signature**

Derive a signing key from your secret access key from keystone credential.

**Steps to create a signature**

1. Derive signing key.
```
kSecret = Secret Access Key
kDate = HMAC("OPENSDS" + kSecret, Date)
kRegion = HMAC(kDate, Region)
kService = HMAC(kRegion, Service)
signingKey = HMAC(kService, "sign_request")
```

Date is in the format YYYYMMDD (for example, 20150830), and does not include the time.

**Example inputs**
```
HMAC(HMAC(HMAC(HMAC("OPENSDS" + kSecret," 20190306"),"ap-south-1"),"s3"),"sign_request")
```
**Example signing key**
```
c4afb1cc5771d871763a393e44b703571b55cc28424d1a5e86da6ed3c154a4b9
```

2. Calculate the signature using signing key and the string to sign as inputs to the keyed hash function and get its hexadecimal representation.
```
signature = HexEncode(HMAC(signing key, string to sign))
```
**Example signature**
```
9b1329c71ebc1923a3b7d801f764336790d98f9fc358b76d1ab90450b0daccd1
```