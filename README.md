

# Identity Credential

## Authors

- Sam Goto
- Adam Langley


## Introduction

This document contains a specification of `IdentityCredential`—a credential type in the [Credential Management](https://www.w3.org/TR/credential-management-1/) framework for all types of identity verification. Types of identity credentials would extend `IdentityCredential` in the same way that different authentication methods extend [Credential Management](https://www.w3.org/TR/credential-management-1/) today.

<img src="./structure.svg" alt="Diagram of API structure" width="300px">

In this proposal, we lift `IdentityCredential` [out of](https://www.w3.org/2023/04/21-webauthn-minutes.html) the Federated Credential Management (FedCM). FedCM would be one extension of `IdentityCredential` but user agents would not need to implement FedCM in order to support other types of identity credentials. 

This proposal is informed by two design choices:

1. identity credentials are going to come from a plurality of schemes 
1. identity verification is not authentication

First, `IdentityCredential` supports multiple identity verification schemes because we envision that sites may want to support multiple identity types in a single identity credential picker. 

For example, student university affiliations are often asserted via [SAML](https://www.oasis-open.org/committees/download.php/56776/sstc-saml-core-errata-2.0-wd-07.pdf) today, but student IDs are going to be increasingly made available in wallet apps ([example](https://www.purdue.edu/newsroom/releases/2023/Q2/purdue-launches-purdue-mobile-id-for-students-allowing-them-to-get-around-campus-with-just-a-simple-tap-of-their-mobile-device.html)) and so could be verifiable credentials (e.g. [mdocs](https://www.ul.com/resources/new-isoiec-standard-electronic-credentials) or [VCs](https://www.w3.org/TR/vc-data-model/)) in the future. An online library (e.g. nature.com) that needs users to prove their student affiliations may need to support both mdocs and federation in order to support students from multiple institutions. 

Second, we think it is key to make requests for identity verification different from requests for authentication. The former is (potentially) a much more significant operation because it may disclose an irrevocable, cross-site identity. Because of this we considered building a `navigator.identity` namespace but ultimately decided to use [Credential Management](https://www.w3.org/TR/credential-management-1/), and thus `navigator.credentials`. This was because FedCM already uses `navigator.credentials.get({identity: …})` and thus the term `identity` would appear twice in the Web Platform if `navigator.identity` was added. But the use of `identity` for `IdentityCredential` was not a mistake: federation, and thus FedCM, provides identity verification and over time we would like authentication to be handled by [`PublicKeyCredential`](https://www.w3.org/TR/webauthn-2/#iface-pkcredential) (e.g. &ldquo;passkeys&rdquo;). However, today, FedCM is used for authentication too and so it lives in `navigator.credentials`. 

Thus we want to support identity requests that can be satisfied by either mdocs or federation, and authentication requests that can be satisfied by either passwords, passkeys, or federation. This implies that all those mechanisms end up in `navigator.credentials` and that has informed this design.

All identity requests are grouped together, rather than made top-level types in Credential Management, to communicate that they are separate from the authentication mechanisms. In general, user agents that receive a request that combines authentication and identity verification, e.g. a request for either an mdoc or a passkey, SHOULD immediately reject such requests with a `NotSupportedError` exception.

## The IdentityCredential Interface

`IdentityCredential` is a new type of [Credential](https://w3c.github.io/webappsec-credential-management/#credential):

```webidl
[Exposed=Window, SecureContext]
interface IdentityCredential : Credential {
  readonly attribute USVString scheme;
  readonly attribute USVString? response;
};
```

The `scheme` specifies the type of identity credential returned and schemes will be defined in other, scheme-specific documents. The `response` is a value with semantics defined by that scheme. If this structure is insufficient, schemes may subclass `IdentityCredential` to add more members.

(This differs from what's specified in FedCM. The intent is that FedCM will adjust to conform.)

[CredentialRequestOptions](https://w3c.github.io/webappsec-credential-management/#dictdef-credentialrequestoptions) is extended with a new member for identity requests:

```webidl
partial dictionary CredentialRequestOptions {
  IdentityCredentialRequestOptions identity;
};
```

`IdentityCredentialRequestOptions` mirrors the pattern of Credential Management and is extended with a list of identity verification providers:

```webidl
dictionary IdentityCredentialRequestOptions {
  // Common request fields, e.g. the context API.

  sequence<IdentityProviderConfig> providers;
}
```

`IdentityProviderConfig` represents a scheme-specific source of identity credentials, and it expected to be extended by other specifications (e.g. the federated or mdocs scheme).

```
dictionary IdentityProviderConfig {
  // More common fields across all identity provider types:
  // ...
};
```

As an illustrative example, the definition of an identity scheme would add members to that dictionary:

```webidl
partial dictionary IdentityProviderConfig {
  MDocsSpecificParameters mdoc;
}
```

When [DiscoverFromExternalSource](https://w3c.github.io/webappsec-credential-management/#dom-credential-discoverfromexternalsource-slot) is invoked, the `DiscoverFromExternalSource` function of each of the different identity providers  specified in the `IdentityCredentialRequestOptions.providers` is run in parallel. If any throws an error that that error is the result of `IdentityCredential`'s DiscoverFromExternalSource. If one or more return an `IdentityCredential` then the user agents picks one of the values to the result, as its discretion. Otherwise the result is `null`.

## Examples

### MDocs

The [MDocs API](https://github.com/agl/mobile-document-request-api/tree/identityapi#examples) extends the `IdentityCredential` API to allow mdocs to be requested:

```js
// I want specific fields out of mobile driver's license as an mdoc
const cbor = await navigator.credentials.get({
  identity: {
    providers: [{
      mdoc: {
        nonce: "gf69kepV+m5tGxUIsFtLi6pwg=",
        readerPublicKey: "ftl+VEHPB17r2 ... Nioc9QZ7X/6w...",
        retention: {
          days: 90,
        },
        documentType: "org.iso.18013.5.1.mDL",
        requestedElements: [
          { namespace: "org.iso.18013.5.1", name: "document_number" },
          { namespace: "org.iso.18013.5.1", name: "portrait" },
          { namespace: "org.iso.18013.5.1", name: "driving_privileges" },
          { namespace: "org.iso.18013.5.1.aamva", name: "organ_donor" },
        ],
      }
    }],
  }
});
```

### Federation

The [FedCM](https://fedidcg.github.io/FedCM/) also extends the `IdentityCredential` API to allow federated assertions to be requested: 

```js
const jwt = await navigator.credentials.get({
  identity: {
    providers: [{
      federated: {
        configURL: "https://university.edu/students",
        clientId: "123",
        nonce: "m5tGxUIsFtLi6pwg"
      }
    }]
  }
}
```

### Reconcilation

User agents can support multiple identity schemes, and they can be requested at the same time:

```js
const credential = await navigator.credentials.get({
  identity: {
    // Requests the user's university affiliation from either 
    // a VC, an MDoc or a JWT.
    providers: [{
      // The university may have a device-bound certificate ...
      mdoc: {
        nonce: "1234",
        retention: { days: 90 },
        documentType: "org.iso.18013.5.1.UniversityDegree",
        readerPublicKey: "...ftl+VEHpdNioc9QZ7X/6w...",
        requestedElements: [
          { namespace: "org.iso.18013.5.1", name: "affiliation" },
        ],
      }
    }, {
      // ... or a SAML assertion ...
      federated: {
        clientId: "myapp",
        nonce: "1234",
        configURL: "https://idp.university.edu",
      }
    }]
  }
});

```
