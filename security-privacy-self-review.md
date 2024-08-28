# Questions to Consider

## 2.1 What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

This feature allows the exchange of data from a third-party to a first-party and itself, across partition boundaries. This is necessary to allow the third-party to behave as an Identity Provider for the first-party.

## 2.2 Do features in your specification expose the minimum amount of information necessary to enable their intended uses?

Yes.

## 2.3 How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?

The only personal information in this feature is in the UI Hints, which may be the name and photo of the user. These are stored locally in the browser and are only used to alter native browser UI to help the user select an account (see 2.11).

## 2.4 How do the features in your specification deal with sensitive information?

The sensitive information here what sites the user has an account on or attempts to log in to. This is especially challenging because we have two sites the user has an account on in a single interaction: the identity provider (i.e. foo in a "Log in with Foo" use case) and the relying party (i.e. the site the user is on that displays a "Log in with Foo" button) This spec protects these from each other by

1. not sending any unpartitioned requests to potential identity providers from the relying party
2. not disclosing to the relying party what identity providers are available

UNTIL the user selects a particular identity provider from a "credential chooser" UI, at which point the parties learn of each other and unpartitioned requests to the identity provider may be made.

## 2.5 Do the features in your specification introduce new state for an origin that persists across browsing sessions?

This specification makes further use of the Credential Management API's Credential Store to persist state across browsing sessions.

## 2.6 Do the features in your specification expose information about the underlying platform to origins?

No

## 2.7 Does this specification allow an origin to send data to the underlying platform?

No

## 2.8 Do features in this specification enable access to device sensors?

No

## 2.9 Do features in this specification enable new script execution/loading mechanisms?

No

## 2.10 Do features in this specification allow an origin to access other devices?

No

## 2.11 Do features in this specification allow an origin some measure of control over a user agent’s native UI?

Yes, this specification extends FedCM to more natively integrate with the Credential Management API. As such, the [Credential Management API's "Credential Chooser"](https://www.w3.org/TR/credential-management-1/#credential-chooser) can be shown with this spec, and the contents of that chooser can be controlled via the UI hints provided by the identity provider in much the same way as other Credential subtypes can.

## 2.12 What temporary identifiers do the features in this specification create or expose to the web?

None

## 2.13 How does this specification distinguish between behavior in first-party and third-party contexts?

The specification would use FedCM's Permission Policy to restrict use to first-party and third-party contexts.

## 2.14 How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

Normally, but with an independent Credential Store that should persist the lifetime of the Private Browsing session.

## 2.15 Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

The explainer does not, but a spec will.

## 2.16 Do features in your specification enable origins to downgrade default security protections?

Yes. With third party cookies deprecated and storage partitioned, the top-level site becomes a boundary preventing the exchange of information. This weakens that, and in combination with [`storage-access` autogrants for FedCM](https://github.com/explainers-by-googlers/storage-access-for-fedcm), actually re-enables third-party cookie deprecation for that origin on that top-level site. 

However, this is done after the context of a use-case-driven, ignorable, native browser credential chooser is presented to the user. This enables an off-ramp for the [`storage-access` heuristics](https://github.com/whatwg/compat/pull/253) which were buit for this use-case.

## 2.17 How does your feature handle non-"fully active" documents?

The specification would use the Credentials Manager API's handling of non-"fully active" documents.

## 2.18 What should this questionnaire have asked?

None!