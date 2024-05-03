# CrossSiteCookieAccessCredential


A [Work Item](https://fedidcg.github.io/charter#work-items)
of the [Federated Identity Community Group](https://fedidcg.github.io/).

## Authors:

- Benjamin VanderSloot (Mozilla)
- Johann Hofmann (Google Chrome)

## Participate
- https://github.com/fedidcg/CrossSiteCookieAccessCredential/issues

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table of Contents

- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [Relying Party API, Getting a Credential](#relying-party-api-getting-a-credential)
  - [FedCM Integration](#fedcm-integration)
- [Relying Party API, Finishing the Creation of a Credential](#relying-party-api-finishing-the-creation-of-a-credential)
- [Relying Party API, Using a Credential](#relying-party-api-using-a-credential)
- [Identity Provider API, Allowing a Credential's Creation During Redirect Flow](#identity-provider-api-allowing-a-credentials-creation-during-redirect-flow)
- [Identity Provider API, Creating a Credential Creation For Many Relying Parties In Advance](#identity-provider-api-creating-a-credential-creation-for-many-relying-parties-in-advance)
- [Identity Provider API, Attaching Account Information to a Credential](#identity-provider-api-attaching-account-information-to-a-credential)
- [Key scenarios](#key-scenarios)
  - [Scenario 1: User intends to link to an identity provider they are not logged into](#scenario-1-user-intends-to-link-to-an-identity-provider-they-are-not-logged-into)
  - [Scenario 2: User intends to link to an identity provider they are already logged in to](#scenario-2-user-intends-to-link-to-an-identity-provider-they-are-already-logged-in-to)
  - [Scenario 3: User intends to log into a site, and may have already linked an identity provider or an unlinked-but-logged-in identity provider](#scenario-3-user-intends-to-log-into-a-site-and-may-have-already-linked-an-identity-provider-or-an-unlinked-but-logged-in-identity-provider)
- [Detailed design discussion](#detailed-design-discussion)
  - [A light touch from the browser](#a-light-touch-from-the-browser)
  - [Using the Credential Manager](#using-the-credential-manager)
  - [Identity provider opt-in per relying party](#identity-provider-opt-in-per-relying-party)
  - [Scope of the credential's effectiveness and storage access](#scope-of-the-credentials-effectiveness-and-storage-access)
  - [Scope of the `crossSiteRequests` and lifetime of those requests](#scope-of-the-crosssiterequests-and-lifetime-of-those-requests)
  - [UI Considerations and identity provider origin](#ui-considerations-and-identity-provider-origin)
  - [Multiple identity providers](#multiple-identity-providers)
  - [The NASCAR problem](#the-nascar-problem)
- [Considered alternatives](#considered-alternatives)
  - [Lightweight Credentials that contain cross-site Identities](#lightweight-credentials-that-contain-cross-site-identities)
  - [Independent Credential type](#independent-credential-type)
  - [requestStorageAccessFor, top-level-storage-access, Forward Declared Storage Access, the old Storage Access API](#requeststorageaccessfor-top-level-storage-access-forward-declared-storage-access-the-old-storage-access-api)
  - [Login Status API](#login-status-api)
  - [Names](#names)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [Acknowledgements](#acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

The goal of this project is to provide a purpose-built API for enabling secure and user-mediated access to cross-site top-level unpartitioned cookies. 
This is accomplished with integration with the [Credential Management API](https://w3c.github.io/webappsec-credential-management/) to enable easy integration with alternative authentication mechanisms.
A site that wants a user to log in calls the `navigator.credentials.get()` function with arguments defined in this spec and after appropriate user mediation and identity provider opt-in an object is returned that gives the power to obtain unpartitioned cookies for the chosen identity provider. 

## Goals

The following use cases are all motivating to this work and it is our goal to provide an easy-to-integrate solution for them that can be integrated into the Credential Manager as a unified browser-mediated login mechanism.

1. Log in with Foo buttons
2. Single-Sign On for domains that are not same-site
3. Revisiting a page that has already been logged in with the API and presenting only the previously used identity provider
4. Identity providers with bounce proxies 
5. Upgrade to [FedCM](https://fedidcg.github.io/FedCM) in browsers that support it
6. IDP discovery, reducing the need for [NASCAR](https://indieweb.org/NASCAR_problem) pages.
7. Allowing account-specific details in the Credential to empower the UI to show that in the Credential Chooser dialog
8. Integrate with FedCM as a lightweight operating mode

## Non-goals

- Custom identity provider infrastructure
- Generic prompts to "allow foo.com to track you"
- Design of an identity protocol

## Relying Party API, Getting a Credential

The site that the user wants to log into needs to call the already existing method `navigator.credentials.get()`. We put our arguments under the `identity` key in the options argument, as does FedCM.
While not shown here, this can be combined with arbitrary other credential arguments.

```js
let credential = await navigator.credentials.get({
  identity : {
    providers : [
      {
        origin: "https://login.idp.net",
        login_url: "https://bounce.example.com/?u=https://login.idp.net/login.html",
        login_target: "redirect",
      },
    ]
  }
});
```

This example shows the use perfect for a "Log in with Foo" button (use case #1, and use case #2), where one identity provider is presented, and if the user has not already logged in, they may be redirected to that provider's login page. This redirect behavior is only permitted when there is only one provider in the list. A provider with `login_url` field indicates that this is the expected mode. If `login_url` is present, but `origin` is not, its value can be inferred as the origin of the link.

Another use example, provided below, shows how to request a credential from one of many IDPs the user may have already linked to this page (use case #3).

```js
let credential = await navigator.credentials.get({
  identity : {
    providers : [
      {
        origin: "https://login.idp.net",
      },
      // ... many allowed ...	
      {
        origin: "https://auth.example.biz",
      },
    ]
  }
});
```

### FedCM Integration

There are two main points that need to be integrated with FedCM. First is the provider list. The approach we take is to permit only either regular FedCM providers or a CrossSiteCookieAccessCredential provider with the `login_url` member. Second is the interaction of this proposal's login with "button mode" FedCM. We allow them to coexist by saying that a FedCM request with `mode: 'button'` implies a `login_target: "popup"`. This is for developer convenience.

```js
navigator.credentials.get({
  identity: {
    mode: "button", // login_target: "redirect" would cause an error now.
    providers : [
      {
        configURL : "https://example.com/FEDCM.json",
      },
      {
        origin : "https://login.idp.net", // Actually fine!
      },
      {
        login_url : "https://auth.example.biz/login" // Invalid combination!
      },
    ]
  }
})
```

## Relying Party API, Using a Credential

The RP can now access the credential as it would with FedCM:

```js
let credential = await navigator.credentials.get();
```

Instead of handling the credential, the RP can also delegate the capability for Storage Access to the IdP

```js
// Inside of an idp.net iframe
await document.requestStorageAccess();
```

## Identity Provider API, Creating a Credential

An identity provider may also provide either an allowlist of domains or an HTTP-endpoint that will reply with a success to a CORS requests from allowed relying parties to create and store a Credential that will be effective for several relying parties in advance. 

```js
let cred = await navigator.credentials.create({
  identity : {
    origin_allowlist: ["https://rp1.biz", "https://rp2.info"], // optional
    dynamic_via_cors: "https://api.login.idp.net/v1/foo", // optional
  }
});
await navigator.credentials.store(cred);
```

This allows the IDP to be used without a redirect flow if the user has already logged in. Because of this, the credential can be one of several of this type in the credential chooser, rather than the only cross-origin credential. If the allowlist is provided, a credential will only appear in the chooser if the relying party is on its allowlist. If the allowlist is not provided, then the credential will appear in the chooser if the same link is provided by the IDP and then a browser-initiated CORS request with `Sec-Fetch-Dest: webidentity` is successful. This is because we can only use the dynamic test endpoint after the user has agreed to use the given identity provider or if the link is identical when provided by the identity provider and relying party for privacy reasons. However, these failures should only result when the relying party or identity provider are misconfigured and can be detected dynamically.

This reduces the need for NASCAR pages. Since we allow identity providers to declare themselves and several that are unlinked to be included in the same credential chooser, we remove the need for NASCAR pages where a user has visited the identity provider before. However, if the user has not visited any of the supported identity providers, then the relying party will still have to present some direction to get the user to their identity provider, and a NASCAR page is a good option.


## Understanding which relying parties to store credentials for

If the user wants to link an IDP that did not already store a valid credential for that origin, the user will find themselves navigated to that `login_url`. In this case, the IDP will want to evaluate the origin of the relying party and then construct and store a credential for that relying party if it so chooses.

```js
for (let r in await IdentityCredential.pendingRequests()) {
  if (IDP_DEFINED_ALLOW_SITE(r.origin)) {
    let cred = await navigator.credentials.create({
      identity : {
        origin_allowlist: [r.origin],
      }
    });
    navigator.credentials.store(cred);
  }
}
```

Here the identity provider chooses which sites may be valid relying parties dynamically from its own page, via the function `IDP_DEFINED_ALLOW_SITE`, after enumerating all pending requests that exist for their use as an identity provider. Those pending requests should have a short lifetime, probably no longer than an hour.


## Identity Provider API, Attaching Account Information to a Credential

We add optional fields to facilitate the user's selection of the credential from the credential chooser. These match the fields in the `CredentialDataMixin` from the `Credential Management Level 1` spec. 

```js
let cred = await navigator.credentials.create({
  identity : {
    dynamic_via_cors: "https://api.login.idp.net/v1/foo",
    ui_hint: {
      name: "example human readable",
      icon: "https://api.login.idp.net/v1/photos/exampleUser",
    }
  }
});
await navigator.credentials.store(cred);
```

The browser should use this information, along with the Origin of the identity provider to construct an entry to the credential chooser that clearly communicates its meaning to the user. In the absence of these fields (or where this function has not yet been called) the identity provider's favicon and Site may be used.

We also add optional fields to allow the identity provider to restrict the lifetime of a Credential's user data, in case there are freshness requirements or deletion requirements on the identity provider. Storing a credential with a falsy value for username or iconURL should delete the previous value in the credential store.
The identity provider can also supply a time at which the account information should expire, as follows:

```js
let cred = await navigator.credentials.create({
	identity : {
    dynamic_via_cors: "https://api.login.idp.net/v1/foo",
    ui_hint: {
      name: "example human readable",
      icon: "https://api.login.idp.net/v1/photos/exampleUser",
      expires: "2025-01-01", // RFC 3339 date-time that is the last time the name and iconURL can be used. After this they are "empty"
    }
  }
});
await navigator.credentials.store(cred);
```


## Key scenarios

These APIs together enable login and linking scenarios that I have put into three  categories.

### Scenario 1: User intends to link to an identity provider they are not logged into

In this case, our user has not used this identity provider (idp.net) on this site (example.com). They first interact with some UI in the page that is clearly associated with the identity provider and the following is called.

```js
let credential = await navigator.credentials.get({
  identity : {
    providers : [
      {
        login_url : "https://login.idp.net/login.html",
      },
    ]
  }
});
```

Browser UI is shown to the user that lets them pick to link their account to the identity provider. On selection, the browser redirects the navigatable to `https://login.idp.net/login.html`.
There, the user may do some auth flow and on completion, the identity provider calls the following:

```js
let cred = await navigator.credentials.create({
  identity : {
    dynamic_via_cors: "https://api.login.idp.net/v1/foo",
    ui_hint: {
      name: "example human readable",
      icon: "https://api.login.idp.net/v1/photos/exampleUser",
    }
  }
});
navigator.credentials.store(cred);
location.href = RETURN_TO_PAGE; // example.com page
```

This stores a new Credential in the Credential Store and enables a silent access for the site and navigates the user back.
Upon return to the site to be logged into, the site runs the following:

```js
let credential = await navigator.credentials.get({
  identity : {
    providers : [
      {
        origin: "https://login.idp.net",
      },
  }
});
````

or embeds an IdP iframe or other resources that want to access unpartitioned cross-site storage.

This can be run on every page load as it is guaranteed to provide no browser UI and provides the cross-site unpartitioned storage access desired.

### Scenario 2: User intends to link to an identity provider they are already logged in to

As a prerequisite to this scenario, when the user logged into its identity provider, it called the following:

```js
let cred = await navigator.credentials.create({
  identity : {
    dynamic_via_cors: "https://api.login.idp.net/v1/foo", 
  }
});
await navigator.credentials.store(cred);
```

As before, in this case, our user has not used this identity provider (idp.net) on this site (example.com). They first interact with some UI in the page that is clearly associated with the identity provider. The same code is called, and the same browser UI is shown. 

```js
let credential = await navigator.credentials.get({
  'identity' : {
    'providers' : [
      {
        login_url : "https://login.idp.net/login.html",
        dynamic_via_cors: "https://api.login.idp.net/v1/foo",
      },
    ]
  }
});
```

However, upon selecting to link with idp.net, the browser notices that it has a way to test if this is a valid origin. Since there is no allowlist, it sends a GET request to `https://api.login.idp.net/v1/foo` with CORS header `Origin: https://www.example.com`, and observes the response. If it is a successful response, the credential is returned.

### Scenario 3: User intends to log into a site, and may have already linked an identity provider or an unlinked-but-logged-in identity provider

In this scenario the user has made some indication to the site that they want to log in. 
The specifics of that interaction dictate what Credential types are appropriate. For sake of discussion, let's say the credentials defined here and a PasswordCredential would be good.
The page then calls the following: 


```js
let credential = await navigator.credentials.get({
  password: true,
  identity : {
    providers : [
      {
        dynamic_via_cors: "https://api.login.idp.net/v1/foo",
        origin: "https://login.idp.net",
      },
      // ... many allowed ...	
      {
        origin: "https://auth.example.biz",
      },
    ]
  }
});
```

Then the user is given any identity provider from the list that they have already linked, any identity providers they have visited and stored themselves in the credential store, and password manager entries as options in the browser UI. Whichever is selected is returned. Note also that depending on the credential manager state, request details, and if only one credential is collected from the store, the UI may be elided. See the [mediation requirements in the Credential Manager API](https://w3c.github.io/webappsec-credential-management/#mediation-requirements).

## Detailed design discussion

### A light touch from the browser

One core principal of this design is to get out of the identity provider's way as quickly and as much as possible. The purpose of UI when using this API should be to gather user consent to the linking of information between sites and then doing no more. Account selection, account data storage, policy presentation, and capability selection are all things we do not want to do as a browser as they are difficult and there is already an industry dedicated to solving these challenges. As such, each credential represents a connection to an identity provider, not an identity.

### Using the Credential Manager

We chose to use the credential manager here because we want this to be login-focused. It also provides a good deal of infrastructure in its design around mediation and allows us to potentially seamlessly integrate with all other login methods.

### Identity provider opt-in per relying party

A natural question is: why can these credentials only be created via this weird dance that involves an identity provider page visit? 

The answer lies in a constraint that the identity provider needs to pick and choose where it allows itself to use cross-site unpartitioned cookies carefully in order to mitigate CSRF attacks. So we have to allow the identity provider a say, and this is done via the `IdentityCredential.requests` interface.

### Scope of the credential's effectiveness and storage access

The credential provides cookie access to just the identity provider's origin. The security benefits of this are discussed elsewhere. We relax constraints on the relying party to site-scoping because login pages can reasonably be on different subdomains than the rest of the site. Because of the natural site-scoping of cookies, this has no privacy impact.

### Scope of the `IdentityCredential.pendingRequests()` and lifetime of those requests

The pending requests of the `pendingRequests` interface is partitioned by top-level navigatable to preserve contextual integrity of the login flow. This means that popup flows are explicitly out of scope. We also dictate that the lifetime of a request should be at most an hour to prevent persistent tracking if a user backs out of an account linkage. Notably the pre-allowed identity providers are not partitioned by navigatable and are instead global.

### UI Considerations and identity provider origin

The credential chooser element for this credential and its discovery should show the identity provider's origin clearly so that the user can make a reasonable decision to link their informaiton between the identity provider and the site that they are on.

### Multiple identity providers

We permit the collection from several identity providers, however only one identity provider may be used when a redirect may occur. Because we do not have a good answer of how to solve the NASCAR problem, we don't want to re-create it in browser UI. So we only permit one IDP as an option when linking.

### The NASCAR problem

We make this a bit better by enabling discovery of a user-selected identity provider that has already been visited. The problem is not fully solved because users must visit the identity provider already to make use of this. Further improvements are welcome directions of future work.

## Considered alternatives

### Lightweight Credentials that contain cross-site Identities

This was decided against because storing identity information in the browser from an identity provider was a hard constraint for the development of FedCM, and we wish to be able to store our credentials in the browser. We also find that this reduces the complexity of privacy analysis.

### Independent Credential type

Making this a distinct credential type from FedCM is a reasonable alternative, but was eventually decided against because of the semantics of this are so similar to that of an `identity` Credential. It makes more sense to be a different operating mode of FedCM, with different arguments.

### requestStorageAccessFor, top-level-storage-access, Forward Declared Storage Access, the old Storage Access API

Several proposals have been made to allow top-level storage access in a generic way. All of them are not use-case specific so their messaging to the user is not clear, making consent more difficult to gather. The flows of this API are nearly identical to that of [top-level-storage-access](https://github.com/bvandersloot-mozilla/top-level-storage-access), however this proposal gains all of the beneifts of integration with the credential manager.

### Login Status API

The identity provider's use of `IdentityCredential.requests` to allow future requests looks a lot like the Login Status API in FedCM. That would be a reasonable place to re-locate this function when the Login Status API sees multi-browser-adoption. However, for now, making future requests a variation on the `allow()` call is simpler to explain and creates no external dependencies.

### Names

All names and strings are welcome to be bikeshed. Little care was put into picking the correct name for anything.

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
