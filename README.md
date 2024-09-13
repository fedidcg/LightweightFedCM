# CACHIMAN FedCM


A [Work Item](https://fedidcg.github.io/charter#work-items)
of the [Federated Identity Community Group](https://fedidcg.github.io/).

## Authors:

- DJENANE VIRGELIN (CACHIMAN)
- ˆLUCIEN VIRGELIN (CACHIMAN)

## Participate
- https://github.com/fedidcg/LightweightFedCM/issues

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table of Contents

- [Introduction](#introduction)
- [TL;DR](#tldr)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [Key scenarios](#key-scenarios)
  - [Scenario 1: User intends to link to an identity provider they are not logged into](#scenario-1-user-intends-to-link-to-an-identity-provider-they-are-not-logged-into)
  - [Scenario 2: User logs in with one of many identity providers, or other types of credentials](#scenario-2-user-logs-in-with-one-of-many-identity-providers-or-other-types-of-credentials)
  - [Scenario 3: User intends to link to an identity provider they are already logged in to](#scenario-3-user-intends-to-link-to-an-identity-provider-they-are-already-logged-in-to)
  - [Scenario 4: User intends to link to an identity provider they are already logged in to, but the relying party cannot provide the origin of](#scenario-4-user-intends-to-link-to-an-identity-provider-they-are-already-logged-in-to-but-the-relying-party-cannot-provide-the-origin-of)
- [Relying Party API, Getting a Credential](#relying-party-api-getting-a-credential)
  - [FedCM Integration](#fedcm-integration)
- [Relying Party API, Using a Credential](#relying-party-api-using-a-credential)
- [Identity Provider API, Creating a Credential](#identity-provider-api-creating-a-credential)
- [Identity Provider API, Attaching Account Information to a Credential](#identity-provider-api-attaching-account-information-to-a-credential)
- [Open Questions](#open-questions)
  - [Requiring loginURL in a site level well-known resource](#requiring-loginurl-in-a-site-level-well-known-resource)
  - [Allowing the relying party to control credentials that appear in the credential chooser](#allowing-the-relying-party-to-control-credentials-that-appear-in-the-credential-chooser)
- [Detailed design discussion](#detailed-design-discussion)
  - [A light touch from the browser](#a-light-touch-from-the-browser)
  - [Using the Credential Manager](#using-the-credential-manager)
  - [Identity provider opt-in per relying party](#identity-provider-opt-in-per-relying-party)
  - [Scope of the credential's effectiveness and storage access](#scope-of-the-credentials-effectiveness-and-storage-access)
  - [UI Considerations and identity provider origin](#ui-considerations-and-identity-provider-origin)
  - [Multiple identity providers](#multiple-identity-providers)
  - [The NASCAR problem](#the-nascar-problem)
- [Considered alternatives](#considered-alternatives)
  - [Independent Credential type](#independent-credential-type)
  - [requestStorageAccessFor, top-level-storage-access, Forward Declared Storage Access, the old Storage Access API](#requeststorageaccessfor-top-level-storage-access-forward-declared-storage-access-the-old-storage-access-api)
  - [Login Status API](#login-status-api)
  - [Browser dialog before navigation to the `loginURL`](#browser-dialog-before-navigation-to-the-loginurl)
  - [Names](#names)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [Acknowledgements](#acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

The goal of this project is to provide a purpose-built API for enabling secure and user-mediated access to cross-site top-level unpartitioned cookies. 
This is accomplished with integration with the [Credential Management API](https://w3c.github.io/webappsec-credential-management/) to enable easy integration with alternative authentication mechanisms.
A site that wants a user to log in calls the `navigator.credentials.get()` function with arguments defined in this spec. The browser ensures there is appropriate user mediation and identity provider opt-in. With those assurances, the browser may also decide there is no additional privacy loss associated with access to unpartitioned state, and choose to automatically grant access to Storage Access requests.

## TL;DR

Do you want to share data across origins for identity purposes?
This API gives you a way to do that.
You can share a data string or use third party cookies via the storage access API without a prompt.
All you have to do is store some data from the identity provider and get it from the relying party with a browser-mediated prompt.

Put this code in your identity provider's page, to be run when the user is logged in, replacing the list of RP origins with your own and any data you want to share:

```js
navigator.credentials.store({
    identity: {
      id: "foo",
      effectiveQueryURL: ["https://www.known-rp.com"],
    }
  });
```

Have the relying parties place this HTML in their page to get a login button, replacing your login URL:

```html
<button id="cscac" hidden>Login via IDP.com</button>
<script>
  let button = document.getElementById("cscac");
  async function do() {
    let identityInit = {
      providers: [{
	      loginURL: "https://auth.idp.com/login"
      }]
    };
    let credential = await navigator.credentials.get({
      identity: identityInit,
      mediation: "silent",
    });
    if (!credential) {
      button.onclick = () => {
        credential = await navigator.credentials.get({
          identity: identityInit,
        });
        console.log("logged in with", credential.origin);
      };
      button.hidden = false;    
    }	
  }();
</script>
```


If you don't want to declare your list of relying parties in advance, you can provide a HTTP endpoint that replies with success only to Origin headers that correspond to your relying parties.
You may have such an endpoint already!
This requires two changes to the code above.

First, you provide the endpoint instead of the list of origins on the IDP site:

```js
navigator.credentials.store({
    identity: {
      id: "preloaded",
      effectiveQueryURL: "https://auth.idp.com/api/v1/anyCORS", // updated this line
    }
  });
```

Second, provide that same URI to the relying parties to be used in the `identityInit` object:

```js
let identityInit = {
      providers: [{
        effectiveQueryURL: "https://auth.idp.com/api/v1/anyCORS", // added this line
	      loginURL: "https://auth.idp.com/login",
      }]
    };
```


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


## Key scenarios

These APIs together enable login and linking scenarios that I have put into a few categories.
For all of these, imagine that an identity provider would store a credential on the user's browser when they log in.


### Scenario 1: User intends to link to an identity provider they are not logged into

In this case, our user is not even logged into this this identity provider (`idp.net`), just having navigated to this site (`example.com`).
They first interact with some UI in the page that is clearly associated with the identity provider and the following is called.

```js
let credential = await navigator.credentials.get({
  identity : {
    providers : [
      {
        loginURL : "https://login.idp.net/login.html",
        origin: "https://login.idp.net", // may be omitted, inferred from loginURL
      },
    ]
  }
});
```

The browser sees there is no credential in the credential store that would work for `example.com`.
So it falls back and opens the `loginURL`. The RP can choose whether to open this URL in a pop-up or via a redirect.
The API defaults to redirecting the user to the `loginURL`.
There, the user goes through some authentication and/or authorization flow entirely of the identity provider's choosing, after which the identity provider stores a credential with some code like this:

```js
let cred = await navigator.credentials.create({
  identity : {
    effectiveOrigins: ["https://example.com"],
  }
});
await navigator.credentials.store(cred);
```

Once this is done, the identity provider navigates the user back to the relying party.
Upon return to `example.com`, the page may run the following, showing the user UI to link the identities as in Scenario 3.

```js
let credential = await navigator.credentials.get({
  identity : {
    providers : [
      {
        origin: "https://login.idp.net",
      },
    ]
  }
});
```

Note that in the case of a popup, the credential chooser should show once the identity provider stores a credential that is effective for the pending credential request on the relying party, removing the need for the relying party to call `navigator.credentials.get` a second time.

### Scenario 2: User logs in with one of many identity providers, or other types of credentials

In this scenario the user has made some indication to the site that they want to log in.
The specifics of that interaction dictate what Credential types are appropriate.
For sake of discussion, let's say the identity providers defined here and a PasswordCredential would be good.
The page then calls the following:

```js
let credential = await navigator.credentials.get({
  password: true,
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

Then the user is presented any account information from identity providers they have visited and stored themselves in the credential store, and password manager entries as options in the browser UI.
Whichever is selected is returned.

Note also that depending on the credential manager state, request details, and if only one credential is collected from the store, the UI may be elided.
Or if the browser simply wants to poll for the presence of such a credential without any UI they can do that as well.
See the [mediation requirements in the Credential Manager API](https://w3c.github.io/webappsec-credential-management/#mediation-requirements) for more details.

### Scenario 3: User intends to link to an identity provider they are already logged in to

In this case, our user has not used this identity provider (`login.idp.net`) on this site (`example.com`). They first interact with some UI in the page that is (hopefully) clearly associated with the identity provider. This calls the following code:

```js
let credential = await navigator.credentials.get({
  'identity' : {
    'providers' : [
      {
        origin : "https://login.idp.net",
      },
    ]
  }
});
```

The browser looks into the credential store and sees that there is a credential this is effective for `example.com` from `login.idp.net`.
The browser give the user a "credential chooser" UI that allows them to share their account at `login.idp.net` with `example.com`.
Once the user consents, a link is made and the Promise is resolved with a Credential.

### Scenario 4: User intends to link to an identity provider they are already logged in to, but the relying party cannot provide the origin of

In this scenario, the user is already logged into an identity provider that the relying party is willing to accept, but may not be willing or able to provide the origin of.
This may be because the relying party trusts a class of identity providers with voluntary membership (e.g., IndieAuth), or because they do not wish to provide a list of acceptable identity providers to the browser (e.g., a consortium with anonymous membership).
To request a credential in this way, the relying party needs to specify a provider with a given "type", like so:

```js
let credential = await navigator.credentials.get({
  'identity' : {
    'providers' : [
      {
        type : "example-string-to-match",
      },
    ]
  }
});
```

## Relying Party API, Getting a Credential

The site that the user wants to log into needs to call the already existing method `navigator.credentials.get()`. We put our arguments under the `identity` key in the options argument, as does FedCM.
While not shown here, this can be combined with arbitrary other credential arguments.

```js
let credential = await navigator.credentials.get({
  identity : {
    providers : [
      {
        origin: "https://login.idp.net",
        loginURL: "https://bounce.example.com/?u=https://login.idp.net/login.html?r=https://rp.net/",
        loginTarget: "redirect",
      },
    ]
  }
});
```

This example shows the use perfect for a "Log in with Foo" button, where one identity provider is presented, and if the user has not already logged in, they may be redirected to that provider's login page. This redirect behavior is only permitted when there is only one provider in the list. A provider with `loginURL` field indicates that this is the expected mode. If `loginURL` is present, but `origin` is not, its value can be inferred as the origin of the link.

Another use example, provided below, shows how to request a credential from one of many IDPs the user may have already linked to this page.

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

There are two main points that need to be integrated with FedCM. First is the provider list. The approach we take is to restrict each provider entry to either the `loginURL` member or the `configURL` member. Second is the interaction of this proposal's login with "button mode" FedCM. We allow them to coexist by saying that a FedCM request with `mode: 'button'` implies a `loginTarget: "popup"`. This is for developer convenience.

```js
navigator.credentials.get({
  identity: {
    mode: "button", // loginTarget: "redirect" would cause an error now.
    providers : [
      {
        configURL : "https://example.com/FEDCM.json",
      },
      {
        origin : "https://login.idp.net", // Actually fine!
      },
      {
        loginURL : "https://auth.example.biz/login" // Invalid combination, can't have loginURL and other providers
      },
      {
        configURL : "https://example.com/FEDCM.json", // This provider entry will never be valid,
        loginURL : "https://auth.example.biz/login"   // even if it is the only one in the list.
      },
    ]
  }
})
```

## Relying Party API, Using a Credential

The RP can use the Credential as an object once it is obtained, as it would with FedCM. This will, for now, only be used to verify that the user has selected an account with a given IdP, providing an `origin` field on the credential by analogy to the `configUrl` from the [multi IdP proposal.](https://github.com/w3c-fedid/multi-idp).

```js
let credential = await navigator.credentials.get({
  identity: {providers: {origin: "https://login.idp.net"}}});
if (credential) {
  let idpConfigSelected = credential.origin;
} else {
  // User did not select an account.
}

```

To use cross site cookies, if the credential can be silently accessed by the RP, then a browser may decide there is no additional privacy loss associated with access to unpartitioned state and choose to automatically grant access to Storage Access requests, as [proposed already for FedCM](https://github.com/explainers-by-googlers/storage-access-for-fedcm).

```js
// Inside of an idp.net iframe
await document.requestStorageAccess();
```

## Identity Provider API, Creating a Credential

The identity provider needs to specify at least one of three arguments when creating the credential (`effectiveOrigins`, `effectiveType`, or `effectiveQueryURL`) to tell the browser the origins for which the credential is [effective](https://w3c.github.io/webappsec-credential-management/#credential-effective). A list of origins may be provided to `effectiveOrigins` if the list of relying parties may be made public and is known ahead of time. If the list of relying parties is dynamic or private, the identity provider may provide an HTTP-endpoint with `effectiveQueryURL` that will respond successfully to a CORS request from the relying party with `Sec-Fetch-Dest: web-identity` if the relying party can use the credential at that time. Also, a string may be provided as the `effectiveType` to allow a relying party to enable out-of-band negotiation with one or a consortium of identity providers.

```js
let cred = await navigator.credentials.create({
  identity : {
    effectiveOrigins: ["https://rp1.biz", "https://rp2.info"], // optional
    effectiveQueryURL: "https://api.login.idp.net/v1/foo", // optional
    effectiveType: "example-string-to-match", // optional
  }
});
await navigator.credentials.store(cred);
```

This allows the identity provider to be used without a redirect flow if the user has already logged in to that provider. Because of this, the credential can be one of several of this type in the credential chooser, rather than the only cross-origin credential. If the allowlist is provided, a credential will only appear in the chooser if the relying party is on its allowlist. If the allowlist is not provided, then the credential will appear in the chooser if the same link is provided by the IDP and a CORS request with `Sec-Fetch-Dest: webidentity` is successful. This is because we can only use the dynamic test endpoint after the user has agreed to use the given identity provider or if the link is identical when provided by the identity provider and relying party for privacy reasons. However, these failures should only result when the relying party or identity provider are misconfigured and can be detected dynamically.

This reduces the need for NASCAR pages. Since we allow identity providers to declare themselves and several that are unlinked to be included in the same credential chooser, we remove the need for NASCAR pages where a user has visited the identity provider before. In those cases where there are no registered identity providers or there are none that are acceptable to a user, the relying party can show fallback content that presents a set of candidate identity providers. Because the choice is not shown to users until obtaining a credential is unsuccessful, the added complexity of the interface might be easier for sites to manage.

## Identity Provider API, Attaching Account Information to a Credential

We add optional fields to facilitate the user's selection of the credential from the credential chooser. These match the fields in the `CredentialDataMixin` from the `Credential Management Level 1` spec.

```js
let cred = await navigator.credentials.create({
  identity : {
    effectiveQueryURL: "https://api.login.idp.net/v1/foo",
    uiHint: {
      name: "example human readable",
      iconURL: "https://api.login.idp.net/v1/photos/exampleUser",
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
    effectiveQueryURL: "https://api.login.idp.net/v1/foo",
    uiHint: {
      name: "example human readable",
      iconURL: "https://api.login.idp.net/v1/photos/exampleUser",
      expiresAfter: 30*24*60*60*1000, // ms after this call that is the last time the name and iconURL can be used. After this they are "empty"
    }
  }
});
await navigator.credentials.store(cred);
```

## Open Questions

### Requiring loginURL in a site level well-known resource

One solution to preventing navigational tracking on the `loginURL` is to make the url be constant across the IDP's site.
This restricts white label SSO use cases and is a challenge for smaller deployments.
Instead we currently accept the navigational tracking since there is no clear path to removing `window.open` from the platform.
Whether or not this is acceptable will depend on further analysis and discussion.

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

### Using the Credential Manager

We chose to use the credential manager here because we want this to be login-focused. It also provides a good deal of infrastructure in its design around mediation and allows us to potentially seamlessly integrate with all other login methods.

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

### Login Status API

The identity provider's use of `IdentityCredential.requests` to allow future requests looks a lot like the Login Status API in FedCM. That would be a reasonable place to re-locate this function when the Login Status API sees multi-browser-adoption. However, for now, making future requests a variation on the `allow()` call is simpler to explain and creates no external dependencies.

### Browser dialog before navigation to the `loginURL`

Allowing a navigation to the identity provider before any dialog does incur the potential for navigational tracking.
However, this is no worse than permitting calls to `window.open`, especially since our use requires user activation.
This also makes presence in the credential chooser entirely opt-in and makes it trivial to obtain an icon to show in place of UI hints, making a well-known resource unneccessary and cleaning up the architecture of the design.

### Names

All names and strings are welcome to be bikeshed.

## Stakeholder Feedback / Opposition

- LUCIEN VIRGELIN : Positive


## Acknowledgements

Many thanks for valuable feedback and advice from:

- 
- 
- 
- 
- 
- 
- 
- 
