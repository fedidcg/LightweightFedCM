# Lightweight FedCM


A [Work Item](https://fedidcg.github.io/charter#work-items)
of the [Federated Identity Community Group](https://fedidcg.github.io/).

## Authors:

- Benjamin VanderSloot (Mozilla)
- Erica Kovac (Google Chrome)

## Participate
- https://github.com/fedidcg/LightweightFedCM/issues

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table of Contents

- [Introduction](#introduction)
- [Goals and Non-Goals](#goals-and-non-goals)
- [FedCM Accounts Push](#fedcm-accounts-push)
- [FedCM Optional Endpoints](#fedcm-optional-endpoints)
  - [Accounts Endpoint](#accounts-endpoint)
  - [Client Metadata Endpoint](#client-metadata-endpoint)
  - [ID Assertion Endpoint](#id-assertion-endpoint)
  - [Login URL](#login-url)
- [FedCM Config Push](#fedcm-config-push)
  - [`client_metadata_endpoint`: Not Supported](#client_metadata_endpoint-not-supported)
  - [`accounts_endpoint`: Supported](#accounts_endpoint-supported)
  - [`id_assertion_endpoint`: Supported](#id_assertion_endpoint-supported)
  - [`disconnect_endpoint`: Supported](#disconnect_endpoint-supported)
  - [`branding`: Supported](#branding-supported)
  - [`login_url`: Partially Supported](#login_url-partially-supported)
- [Interaction with other FedCM Features](#interaction-with-other-fedcm-features)
  - ["Login-Status" Headers](#login-status-headers)
  - [Distinguishing between `sign-up` and `sign-in` without an `accounts_endpoint`](#distinguishing-between-sign-up-and-sign-in-without-an-accounts_endpoint)
- [Open Questions](#open-questions)
- [Detailed design discussion](#detailed-design-discussion)
  - [Using the Credential Manager and Login Status API](#using-the-credential-manager-and-login-status-api)
- [Considered alternatives](#considered-alternatives)
  - [A distinct "mode" for FedCM](#a-distinct-mode-for-fedcm)
  - [Independent Credential type](#independent-credential-type)
  - [requestStorageAccessFor, top-level-storage-access, Forward Declared Storage Access, the old Storage Access API](#requeststorageaccessfor-top-level-storage-access-forward-declared-storage-access-the-old-storage-access-api)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [Acknowledgements](#acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

The goal of this proposal is to create more flexibility for Identity Providers (IdPs) who wish to integrate with Federated Credential Management and gain the benefits of more contextually-relevant user-mediated access to cross-site cookies.

This is accomplished with three proposed changes to the [FedCM specification](https://w3c-fedid.github.io/FedCM/):

* *FedCM Accounts Push*: Allow IdPs to store display information about user accounts up-front via the [Login Status API](https://w3c-fedid.github.io/login-status/#introduction).
* *FedCM Optional Endpoints*: Allow IdPs to skip implementation of the `client_metadata_endpoint`, `id_assertion_endpoint`, `accounts_endpoint`, and `login_url` by defining user agent semantics when any of these are not present.
* *FedCM Config Push*: Allow IdPs to store endpoint configuration information up-front via the Login Status API.

It is possible to look at this proposal as an “axiomatic base” of FedCM; simplifying FedCM down to its barest essentials that will allow it to still function in a way that provides value to both users and identity providers. One benefit of this approach is it allows for incremental adoption of full FedCM by existing identity providers. It defines useful user-agent behaviors for FedCM when some of the specified endpoints are not defined.

In particular, making the `id_assertion_endpoint` optional would pave the way for Identity Providers that only require unpartitioned third-party cookie access in an iframe on a Relying Party in order to function. 

## Goals and Non-Goals

The following use cases are all motivating to this work and it is our goal to provide an easy-to-integrate solution for them that can be integrated into the Credential Manager as a unified browser-mediated login mechanism.

These changes are intended to:
* Allow IdPs to request Storage Access with a UI that gives users more context for their choices.
* Provide the benefits of the FedCM UX to IdPs whose system architecture or engineering budget does not permit adoption of full FedCM.
* Improve performance and privacy properties of FedCM by optionally moving credentialed calls to IdPs to a later point in the sign-in user journey.

These changes explicitly must *not*:
* Enable invisible or silent timing attacks for a colluding IdP and RP.
* Create incentives for companies engaged in tracking to present themselves as a fake IdP.

These changes are *not* intended to:
* Provide full drop-in compatibility with existing SSO architectures; 
  some engineering work by IdPs will still remain to adopt FedCM.
* Replace or supersede FedCM.
* Create an entirely new type of credential or identity system.

## FedCM Accounts Push

In addition to, or in lieu of, implementing an `accounts_endpoint` as defined in the FedCM specification, an IdP using FedCM Accounts Push can call the Login Status API with information about the user when the user logs in or visits their site:

```js
// On the IdP's sign-in page after successful auth, or whenever a
// signed-in user visits the IdP. ("example.com" here)
// The profile information on accounts is a dictionary of the form defined in
// https://w3c-fedid.github.io/FedCM/#dictdef-identityprovideraccount
navigator.login.setStatus("logged-in", {
	accounts: [{
		id: "1234",
		name: "John Doe",
		email: "foobar@example.com",
                picture: "https://example.com/users/foobar.jpg",
  }],
  expiration: 86_400_000 // 24 hours
});

// navigator.login.setStatus("logged-out"); undoes the prior operation
```

Then, when an RP calls `navigator.credentials.get()` with the appropriate configuration for `example.com` to begin the FedCM signin or signup flows, the user's browser will not have to make a credentialed request to the `accounts_endpoint` defined in `fedcm.json`; instead, the browser-controlled account selector will be displayed immediately, using the accounts list that was previously stored by the IdP. If the account information has expired, the browser will either invoke the `accounts_endpoint`, or, if the `accounts_endpoint` isn't defined in the IdP's configuration, navigate to the `login_url` (when `mode: active`) or terminate the flow and reject the returned promise (when `mode: passive`).

```
const credential = await navigator.credentials.get({
  identity: {providers: [{configURL: "https://example.com/fedcm.json"}]}
});
```

## FedCM Optional Endpoints

Some IdPs may not require all aspects of the FedCM flow in order to provide a useful experience to users and RPs. It might be useful for an IdP to not define some endpoints in the [IdentityProviderAPIConfig](https://w3c-fedid.github.io/FedCM/#dictdef-identityproviderapiconfig), rather than implementing stubs. If they define none of them, no IdP configuration should be required at all.

### Accounts Endpoint

If the IdP is making use of FedCM Accounts Push, then it should be permitted to not define an `accounts_endpoint` in their IdP config. If no `accounts_endpoint` is defined and there are no stored accounts from Accounts Push for the IdP, the user agent behavior for credential requests will be identical to the case where an empty list is returned from the `accounts_endpoint`.

### Client Metadata Endpoint

Some IdPs may not require all aspects of the FedCM flow to provide a useful experience to users and RPs. It might be useful for an IdP to not define some endpoints in the [IdentityProviderAPIConfig](https://w3c-fedid.github.io/FedCM/#dictdef-identityproviderapiconfig), rather than implementing stubs. If they define none of these endpoints, no IdP configuration should be required at all.

The `client_metadata_endpoint` is already [effectively optional in Chrome](https://github.com/w3c-fedid/FedCM/issues/701#issuecomment-2666184937), and the intention is to update the specification to match the implemented behavior.

### ID Assertion Endpoint

IdPs may not need to return any sort of token to the relying party, instead relying on unpartitioned third-party cookie access in an embedded IdP iframe on the RP site. If the IdP does not define an `id_assertion_endpoint`, the `IdentityCredential` that is returned by the RP's `navigator.credentials.get()` call will have an empty string for the `token` value.

### Login URL

If the `login_url` is not defined in the IdP API configuration, cases where the browser would have opened `login_url` will instead navigate to the origin of the URL specified in the provider configuration in the `navigator.credentials.get()`. 

```
const credential = await navigator.credentials.get({
  identity: {providers: [{configURL: "https://example.com/fedcm.json"}],
             mode: "active"}
});
```



## FedCM Config Push

An IdP may prefer to define the API configuration of their deployment by pushing the configuration to the user's browser when the user successfully logs in.

We propose doing this via the LoginStatus API. Example:

```
// On idp.example
navigator.login.setStatus("logged-in", {
	apiConfig: {
    accounts_endpoint: "https://idp.example/api/accounts",
    id_assertion_endpoint: "https://idp.example/api/id_assertion",
    login_url: "https://idp.example/signin",
    disconnect_endpoint: "https://idp.example/api/disconnect",
    branding: {
      icons: [{url: "https://idp.example/fedcm-icon-32.png", size: 32"}]
    }
  }
  expiration: 86_400_000 // 24 hours
});
```

Most features of "Full" FedCM should still be available if the `navigator.login.setStatus` implementation route is chosen. The exceptions are outlined below.

### `client_metadata_endpoint`: Not Supported

Since the `client_metadata_endpoint` is invoked with referrer details before the user has selected the IdP, an IdP that wished to track the user could store a decorated link that is unique to the user, and then join that unique ID with the request origin header. In order to prevent this, the `client_metadata_endpoint` parameter should only be used when read from a `configURL` supplied by an RP with the same `.well-known/web-identity` constraints as defined in the full FedCM specification, not supplied via a `setStatus` call. It would then be necessary for RPs to supply these details in the `navigator.credentials.get` call. A set of parameters is proposed in a [comment on the FedCM issue tracker.](https://github.com/w3c-fedid/FedCM/issues/665).

```js
const credential = await navigator.credentials.get({
  identity: {
    provider: [{
      configURL: "https://idp.example.com/",
    }],
    rp: {
      termsOfService: "https://rp.example/tos.html",
      privacyPolicy: "https://rp.example/tos.html",
    },
  }
});
```

### `accounts_endpoint`: Supported

Since the `accounts_endpoint` is invoked without referrer details, it shouldn't cause an IdP to become aware of the user's visit to an RP, even if the IdP is allowed to define it via `setStatus` and not require a matching `.well_known/web-identity` and JSON config URL. Since credentials are already being sent with this request, link decoration by the IdP would not provide them with any new information. Because the `setStatus` is called on the IdP's origin ahead of time, there's no way to smuggle the RP's identity to the IdP before the user has linked their identity.

### `id_assertion_endpoint`: Supported

There are no special considerations for this, since this is only invoked after the user has confirmed that they want to link their identity between the RP and IdP.

### `disconnect_endpoint`: Supported

Supplying this in `n.l.setStatus` should be fine, since this will only be invoked after a user has linked their account in the first place.

### `branding`: Supported

Supplying this in `n.l.setStatus` should be fine, with the caveat that the icon URLs provided must be retrieved when `setStatus` is called, *not* when the RP calls `n.c.get`.

### `login_url`: Partially Supported

The primary caveat is that if a `login_url` is supplied in `setStatus`, it could include identifying information for the user before the user has allowed linking between the RP and IdP. This can be partially alleviated by not supporting an RP-supplied `loginHint` or `domainHint` that would allow the IdP to create that linkage too early in the user journey for user permission to be gathered. The user agent can also choose to require additional confirmation before navigating to the loginUrl.

## Interaction with other FedCM Features

The FedCM features addressed here should not be considered exhaustive; instead, this outlines features where special consideration is
required. Unless otherwise stated, any FedCM functionality should be available to IdPs taking advantage of Accounts Push, Optional Endpoints, or Config Push. 

Features that should work with no modification include:

* Multiple identity providers in one `navigator.credentials.get()` call.
* Login, Domain, and Account Hints for accounts filtering.
* Fields and params for identity assertion retrieval.
* `IdentityProvider.getUserInfo()` (in an iframe with an origin matching the IdP origin)

### "Login-Status" Headers

An IdP sending the `Login-Status: logged-in` response header refreshes the expiration timer on the stored profile information by resetting the browser's last-modified timestamp for that site's Login Status information.

Sending `Login-Status: logged-out` clears the profile information along with the login status bit.

### Distinguishing between `sign-up` and `sign-in` without an `accounts_endpoint`

By default, FedCM makes a distinction between `sign-up` and `sign-in` via a property in the user profile information, called `approved_clients`, that is received in the [fetch the accounts](https://w3c-fedid.github.io/FedCM/#fetch-the-accounts) step: if the `clientId` passed in the `navigator.credentials.get()` call is not a member of `approved_clients`, it means that this client was never previously approved by the user.

This could also be accomplished client-side by passing a list of `approved_clients` in the login status API call:

```js
navigator.login.setStatus("logged-in", {
	accounts: [{
         // ... other fields
         approved_clients: ["https://rp.example"],
         // ... other fields
  }]
});
```

## Open Questions

* Should we allow defining the profile information and API config in the `Login-Status` header?
* Does expiration also expire the IdP-supplied `apiConfig` or just the account information?
* Should it be possible to specify expiration times on a per-account basis?
* Should it be possible to add or remove individual accounts from the list instead of replacing the entire account list?
* Does `Login-Status: logged-out` clear the IdP-supplied `apiConfig?`
* Should the API config be supplied using a call to the IdP registration API instead?
* For `approved_clients`, should we support matching on the RP origin, not just the clientId?

## Detailed design discussion

### Using the Credential Manager and Login Status API

We chose to use the credential manager on the RP side here because we want this to be login-focused. It also provides a good deal of infrastructure in its design around mediation and allows us to potentially seamlessly integrate with all other login methods.

Using the Login Status API for the IdP side reflects that what is being stored is not, in and of itself, a credential. It is a declaration of the availability of a credential, with information to make it clear to the user what that credential is. It also definitively answers questions about how this functionality should behave if the user has been `Login-Status: logged-out` by an IdP.

## Considered alternatives

### A distinct "mode" for FedCM

The initial proposal and some of its intermediate revisions implied that "Lightweight" would be all-or-nothing; either you get the minimal set of features, or you use "full" FedCM. By making this a set of proposals for changes to the existing FedCM spec instead, it makes sure the benefits of these features are usable by a wider range of implementing IdPs.

### Independent Credential type

Making this a distinct credential type from FedCM is a reasonable alternative, but was eventually decided against because of the semantics of this are so similar to that of an `identity` Credential. It makes more sense to be a different operating mode of FedCM, with different arguments.

### requestStorageAccessFor, top-level-storage-access, Forward Declared Storage Access, the old Storage Access API

Several proposals have been made to allow top-level storage access in a generic way. All of them are not use-case specific so their messaging to the user is not clear, making permission more difficult to gather. A minimal IdP implementation, made possible by the proposed changes above, can enable functionality nearly identical to that of [top-level-storage-access](https://github.com/bvandersloot-mozilla/top-level-storage-access) with minimal overhead, while still allowing the browser to provide needed context to the user.

## Stakeholder Feedback / Opposition

- Mozilla : Positive


## Acknowledgements

Many thanks for valuable feedback and advice from:

- Tim Cappalli
- George Fletcher
- Sam Goto
- Yi Gu
- Nicolas Peña Moreno
- Achim Schlosser
- Phil Smart
- Martin Thompson
- Christian Biesinger
- Johann Hofmann