# Lightweight FedCM


A [Work Item](https://fedidcg.github.io/charter#work-items)
of the [Federated Identity Community Group](https://fedidcg.github.io/).

## Authors:

- Benjamin VanderSloot (Mozilla)
- Johann Hofmann (Google Chrome)

## Participate
- https://github.com/fedidcg/LightweightFedCM/issues

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table of Contents

- [Introduction](#introduction)
- [Goals and Non-Goals](#goals-and-non-goals)
- [Incremental Deployment of FedCM](#incremental-deployment-of-fedcm)
  - [Passive Mode Or Active Mode Without Fallback](#passive-mode-or-active-mode-without-fallback)
  - [Active Mode: Handling the Signed-Out Case](#active-mode-handling-the-signed-out-case)
- [Considerations for Supplying IdP Configuration in setStatus](#considerations-for-supplying-idp-configuration-in-setstatus)
  - [`client_metadata_endpoint` ❗](#client_metadata_endpoint-)
  - [`accounts_endpoint` ✔️](#accounts_endpoint-)
  - [`id_assertion_endpoint` ✔️](#id_assertion_endpoint-)
  - [`disconnect_endpoint` ✔️](#disconnect_endpoint-)
  - [`branding` ✔️](#branding-)
  - [`login_url` ❗](#login_url-)
- [Interaction with other FedCM Features](#interaction-with-other-fedcm-features)
  - ["Login-Status" Headers](#login-status-headers)
  - [Distinguishing between `sign-up` and `sign-in` without an `accounts_endpoint`](#distinguishing-between-sign-up-and-sign-in-without-an-accounts_endpoint)
  - [Handling Selective Disclosure with the Field Selector](#handling-selective-disclosure-with-the-field-selector)
  - [Handling Multiple IdPs](#handling-multiple-idps)
  - [IdP Registration: Handling "any" IdP](#idp-registration-handling-any-idp)
- [Open Questions](#open-questions)
  - [Allowing the relying party to control credentials that appear in the credential chooser](#allowing-the-relying-party-to-control-credentials-that-appear-in-the-credential-chooser)
- [Detailed design discussion](#detailed-design-discussion)
  - [A light touch from the browser](#a-light-touch-from-the-browser)
  - [Using the Credential Manager and Login Status API](#using-the-credential-manager-and-login-status-api)
  - [Identity provider opt-in per relying party](#identity-provider-opt-in-per-relying-party)
  - [Scope of the credential's effectiveness and storage access](#scope-of-the-credentials-effectiveness-and-storage-access)
  - [UI Considerations and identity provider origin](#ui-considerations-and-identity-provider-origin)
  - [Multiple identity providers](#multiple-identity-providers)
  - [The NASCAR problem](#the-nascar-problem)
- [Considered alternatives](#considered-alternatives)
  - [Independent Credential type](#independent-credential-type)
  - [requestStorageAccessFor, top-level-storage-access, Forward Declared Storage Access, the old Storage Access API](#requeststorageaccessfor-top-level-storage-access-forward-declared-storage-access-the-old-storage-access-api)
  - [Names](#names)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [Acknowledgements](#acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

The goal of this project is to provide a purpose-built API for enabling secure and user-mediated access to cross-site top-level unpartitioned cookies. 
This is accomplished with integration with the [Credential Management API](https://w3c.github.io/webappsec-credential-management/) and the [Login Status API](https://w3c-fedid.github.io/login-status/#introduction) to enable easy integration with alternative authentication mechanisms.
A site that wants a user to log in calls the `navigator.credentials.get()` function with arguments defined in this spec. The browser ensures there is appropriate user mediation and identity provider opt-in. With those assurances, the browser may also decide there is no additional privacy loss associated with access to unpartitioned state, and choose to automatically grant access to Storage Access requests.

It is possible to look at this proposal as an “axiomatic base” of FedCM; simplifying FedCM down to its barest essentials that will allow it to still function in a way that provides value to both users and identity providers. One benefit of this approach is it allows for incremental adoption of full FedCM by existing identity providers. It defines useful user-agent behaviors for FedCM when some of the specified endpoints are not defined. This also has some implementation benefits for user-agent developers, as it does not require that an entirely parallel codepath be developed, tested, and maintained.


## Goals and Non-Goals

The following use cases are all motivating to this work and it is our goal to provide an easy-to-integrate solution for them that can be integrated into the Credential Manager as a unified browser-mediated login mechanism.

This feature is intended to:
* Allow IdPs to request Storage Access with a UI that gives users more context for their choices.
* Provide the benefits of the FedCM UX to IdPs whose system architecture or engineering budget does not permit adoption of full FedCM.
* Improve performance and privacy properties of FedCM by moving credentialed calls to IdPs to a later point in the sign-in user journey.
* Act as a supplement to FedCM, not an entirely parallel API surface; ideally, an understanding of FedCM should confer some understanding of Lightweight FedCM, and vice versa.

This feature explicitly must *not*:
* Make invisible/silent timing attacks possible for a colluding IdP and RP.
* Create incentives for companies engaged in tracking to present themselves as a fake IdP.

This feature is *not* intended to:
* Entirely eliminate the need for some serving changes on the IdP; a limited number of new endpoints are acceptable in order to support some functionality.
* Create an entirely new type of credential or identity system.

## Incremental Deployment of FedCM

### Passive Mode Or Active Mode Without Fallback

If you are an IdP with existing infrastructure that relies on third-party cookie access to allow streamlined sign-in, ideally you will need to make minimal changes for your services to continue functioning in a world where a significant proportion of browsers no longer support unpartitioned third party cookie access. While you could have your embedded iframe on the RP call requestStorageAccess, that would require that the user interacts with the iframe. Additionally, the requestStorageAccess prompt in that case would not provide context about why you are requesting access; the user may well decline the request without understanding that it is necessary for sign-in to work smoothly.

```js
// On the IdP's sign-in page after successful auth, or whenever a
// signed-in user visits the IdP.
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

Even with just this client-side change, the browser will be able to give a meaningful prompt to the user when the following code is executed on the RP side.

```js
try {
  const {profile: {id, name, email, picture}} = await navigator.credentials.get({
  	identity: {providers: [{url: "https://example.com"}]}
  });
  //  ... Post a message to the IdP iframe with the account ID
catch (e) {
  // User either manually declined the prompt or the user was logged-out;
  // importantly, these two outcomes are indistinguishable to the RP.
}

// Browser checks the login status for https://example.com; if there's a logged-in status with account information (that isn't expired), prompt the user with that account information. If not, issue a non-credentialled request to "https://example.com/.well-known/web-identity"; since this doesn't return the correct status+MIME type in this case, the browser is able to degrade gracefully: in passive mode, without a logged in account, the prompt is simply dismissed and an exception is thrown.
...

// In the IdP iframe, after receiving a postMessage from the toplevel document with the account ID:
await navigator.requestStorageAccess(); // Since the user selected an account already, this will be autogranted.
const token = (await fetch("https://example.com/login?rp=rp.example&id=" + id)).text();
// Do whatever you would usually do with your token here.
```

This approach works well with the Passive mode: if the user is not already signed in to the IdP, the browser doesn’t need to display anything. In Active Mode, the RP may wish to provide some guidance or trigger some action to get the user to refresh their IdP session, like navigating to the login page if it is known.

> [!NOTE]
> We refer to a `url` field on the RP-side provider configurations instead of `configURL`; this is to align with future work deprecating the 
> `configURL` parameter in favor of `url` in FedCM.

### Active Mode: Handling the Signed-Out Case

For Active mode, if you want to be able to gracefully handle the signed-out case, IdPs have an incremental path: allow users to login using the “url” passed by the RP or (optionally) the IdP providing a custom one.

So, if an RP called:

```
let cred = await navigator.credentials.get({
	identity: {
         mode: "active",
         providers: [{url: "https://example.com"}]
       },
});
```

The browser would check the sign-in status list, 
first by trying to resolve “https://example.com/.well-known/web-identity”. If the well-known file doesn’t exist (i.e., the request returns a status 404), the browser would use `“login_url”` as the value of `“https://” + eltd+1(“url”)` (i.e., in this example, `“https://example.com”`). If an IdP requires a custom `login_url`, they can have a very simple `.well-known/identity` file, such as the following:

```json
{
   "login_url": "https://example.com/login"
}
```

Alternatively, the IdP can specify the `login_url` as part of their `navigator.login.setStatus` call:

```js
navigator.login.setStatus("logged-in", {
	accounts: [{
		id: "1234",
		name: "John Doe",
		email: "foobar@example.com",
    picture: "https://example.com/users/foobar.jpg",
  }],
  apiConfig: {
    login_url: "/login.html";
  }
  expiration: 86_400_000 // 24 hours
});
```

On the other hand, if the user doesn’t have cached account info from the login status API (or if their cached info has expired), the user agent can present the user with the option to sign in with that IdP.


## Considerations for Supplying IdP Configuration in setStatus

Most features of "Full" FedCM should still be available if the `navigator.login.setStatus` implementation route is chosen.

### `client_metadata_endpoint` ❌

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

Unfortunately, not supporting the `client_metadata_endpoint` removes the ability of IdPs to avoid reputational attacks that involve visually associating the IdP with an unapproved and undesired RP. IdPs that are sensitive to this kind of attack could supply a `.well_known/web-identity` and `config.json` and configure the `client_metadata_endpoint` to return an error if the origin does not match their allow-list, as in "full" FedCM.

### `accounts_endpoint` ✔️

Since the `accounts_endpoint` is invoked without referrer details, it shouldn't cause an IdP to become aware of the user's visit to an RP, even if the IdP is allowed to define it via `setStatus` and not require a matching `.well_known/web-identity` and JSON config URL. Since credentials are already being sent with the request, link decoration by the IdP would not provide them with any new information. Because the `setStatus` is occurring on the IdP's origin ahead of time, there's no way to smuggle the RP's identity to the IdP before the user has linked their identity.

### `id_assertion_endpoint` ✔️

There are no special considerations for this, since this is only invoked after the user has confirmed that they want to link their identity between the RP and IdP.

### `disconnect_endpoint` ✔️

Supplying this in `n.l.setStatus` should be fine, since this will only be invoked after a user has linked their account in the first place.

### `branding` ✔️

Supplying this in `n.l.setStatus` should be fine, with the caveat that the icon URLs provided must be retrieved when `setStatus` is called, *not* when the RP calls `n.c.get`.

### `login_url` ❗

The primary caveat is that if a `login_url` is supplied in `setStatus`, it could include identifying information for the user before the user has allowed linking between the RP and IdP. This can be partially alleviated by not supporting an RP-supplied `loginHint` or `domainHint` that would allow the IdP to create that linkage too early in the user journey for user permission to be gathered. The user agent can also choose to require additional confirmation before navigating to the loginUrl.

## Interaction with other FedCM Features

### "Login-Status" Headers

An IdP sending `Login-Status: logged-in` refreshes the expiration timer on the stored profile information by resetting the browser's last-modified timestamp for that site's Login Status information.

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

### Handling Selective Disclosure with the Field Selector

An RP can also select which attributes of the user profile they would like the IdP or user agent to disclose. Currently, the following values are allowed: `name`, `email`, `picture`, `phoneNumber`, and empty list. This aligns with [the current custom requests proposal](https://github.com/w3c-fedid/custom-requests/issues/4).

This would also work client-side only, with the returned credential object only containing the specific attributes that were requested if they were stored in the `setStatus` call.

```js
const {profile: {email, name}} = await navigator.credentials.get({
  identity: {
        providers: [{
          url: "https://example.com",
          fields: ["email", "name"] // doesn't request the profile picture
        }],
      }
});
```

### Handling Multiple IdPs

This would extend well if the user was "logged-in" in multiple IdPs, when an RP call could be as follows:

```js
const {profile: {id, name, email, picture}} = await navigator.credentials.get({
  identity: {
        providers: [{
          url: "https://idp1.example"
        }, {
          url: "https://idp2.example"
        }]
      }
});
```

### IdP Registration: Handling "any" IdP

In passive mode, an IdP can still use the IdP Registration API to expose itself for RPs that don’t want to enumerate IdPs:

```js
// Prompts the user to accept this top-level origin as an IdP
await IdentityProvider.register("indie-auth");
```

Allows an RP to call:

```js
await navigator.credentials.get({
  identity: {
    providers: [{type: "indie-auth"}]
  }
});
```

## Open Questions

* Should we allow defining the profile information and API config in the `Login-Status` header?
* Does expiration also expire the IdP-supplied `apiConfig` or just the account information?
* Does `Login-Status: logged-out` clear the IdP-supplied `apiConfig?`
* Should the API config be supplied using a call to the IdP registration API instead?
* Given the lower barrier to entry and risk of abuse, should we support passive mode for Lightweight FedCM at all?
* Under what conditions is presenting IdP picker mandatory vs UA-implementation-defined?
* For `approved_clients`, should we support matching on the RP origin, not just the clientId?

### Allowing the relying party to control credentials that appear in the credential chooser

Since any site can claim to be an identity provider with any `"effectiveType"`, we may want to allow websites further control over the elements in the UI.
However this carries a risk of information leak to the relying party of all of the origins of a given type.
Currently the relying party may mitigate this by validating the origin of the returned credential, or by attempting to use the credential, and by repeating the authentication process if it is unacceptable.
Here is an example of such behavior in some abstracted Javascript:

```js
while (true) {
  let cred = navigator.credentials.get(options);
  if (allowedOrigin(cred.origin) && credentialWorks(cred)) {
    break;
  }
}
useCredential(cred);
```


## Detailed design discussion

### A light touch from the browser

One core principal of this design is to get out of the identity provider's way as quickly and as much as possible. The purpose of UI when using this API should be to gather user consent to the linking of information between sites and then doing no more. Account selection, account data storage, policy presentation, and capability selection are all things we do not want to do as a browser as they are difficult and there is already an industry dedicated to solving these challenges. As such, each credential represents a connection to an identity provider, not an identity.

### Using the Credential Manager and Login Status API

We chose to use the credential manager on the RP side here because we want this to be login-focused. It also provides a good deal of infrastructure in its design around mediation and allows us to potentially seamlessly integrate with all other login methods.

Using the Login Status API for the IdP side reflects that what is being stored is not, in and of itself, a credential. It is a declaration of the availability of a credential, with information to make it clear to the user what that credential is. It also definitively answers questions about how this functionality should behave if the user has been `Login-Status: logged-out` by an IdP.

### Identity provider opt-in per relying party

A natural question is: why can these credentials only be created via this weird dance that involves an identity provider page visit? 

The answer lies in a constraint that the identity provider needs to pick and choose where it allows itself to use cross-site unpartitioned cookies carefully in order to mitigate CSRF attacks. So we have to allow the identity provider a say, and this is done via the `IdentityCredential.requests` interface.

### Scope of the credential's effectiveness and storage access

The credential provides cookie access to just the identity provider's origin. The security benefits of this are discussed elsewhere. We relax constraints on the relying party to site-scoping because login pages can reasonably be on different subdomains than the rest of the site. Because of the natural site-scoping of cookies, this has no privacy impact.

### UI Considerations and identity provider origin

The credential chooser element for this credential and its discovery should show the identity provider's origin clearly so that the user can make a reasonable decision to link their informaiton between the identity provider and the site that they are on.

### Multiple identity providers

We permit the collection from several identity providers, however only one identity provider may be used when a redirect may occur. Because we do not have a good answer of how to solve the NASCAR problem, we don't want to re-create it in browser UI. So we only permit one IDP as an option when linking.

### The NASCAR problem

We make this a bit better by enabling discovery of a user-selected identity provider that has already been visited. The problem is not fully solved because users must visit the identity provider already to make use of this. Further improvements are welcome directions of future work.

## Considered alternatives

### Independent Credential type

Making this a distinct credential type from FedCM is a reasonable alternative, but was eventually decided against because of the semantics of this are so similar to that of an `identity` Credential. It makes more sense to be a different operating mode of FedCM, with different arguments.

### requestStorageAccessFor, top-level-storage-access, Forward Declared Storage Access, the old Storage Access API

Several proposals have been made to allow top-level storage access in a generic way. All of them are not use-case specific so their messaging to the user is not clear, making consent more difficult to gather. The flows of this API are nearly identical to that of [top-level-storage-access](https://github.com/bvandersloot-mozilla/top-level-storage-access), however this proposal gains all of the beneifts of integration with the credential manager.

### Names

All names and strings are welcome to be bikeshed. Ideally, names should be chosen to align as closely as possible with their equivalent concepts in "full" FedCM.

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
- Erica Kovac
