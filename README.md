This technology is paused for development.This repository will be archived and will no longer be updated.
See our [Update on Plans for Privacy Sandbox Technologies](https://privacysandbox.com/news/update-on-plans-for-privacy-sandbox-technologies/).
[Privacy Sandbox feature status](https://privacysandbox.google.com/overview/status) provides more information about the status of individual APIs and platform features.

----------

# Related Website Partition API Explainer

### Authors

-   [Dylan Cutler](https://github.com/DCtheTall)

### Contributors

-   [Ari Chivukula](https://github.com/arichiv)

### Participate

-   https://github.com/explainers-by-googlers/related-website-partition-api/issues/

## Motivation

As web browsers make changes to restrict third party cookies and introduce new privacy preserving alternatives, one change that some major browsers have shipped is [third-party storage partitioning](https://github.com/privacycg/storage-partitioning).
This effort, along with [Cookies Having Independent Partitioned State](https://github.com/privacycg/CHIPS) for cookies, partitions third-party embeds' network and JavaScript state across top-level sites in order to prevent cross-site tracking.
[Related Website Sets](https://github.com/WICG/first-party-sets/) (RWS) is another effort which aims to introduce a new privacy boundary on the web which allows sites to have access to their unpartitioned cross-site state when they are embedded on other related sites.

Some third-party SaaS developers embed their content on sites which are part of a Related Website Set and wish to maintain a continuous session across multiple sites within that set (see this [issue](https://github.com/WICG/first-party-sets/issues/94) for more information).
For example, consider a SaaS company, `support.chat`, which embeds a chat widget on multiple websites in a Related Website Set.
The chat session starts on `setmember1.com`, but the user needs to also navigate to `setmember2.com` in order to complete their task.
We would like to support an API which allows the chat session to continue seamlessly between both sites.

Some of these sites may be able to leverage the [Storage Access API](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/StorageAccessAPI/explainer.md) in order to meet this use case, but this API has several limitations for this particular scenario:

-   If `support.chat` is embedded in multiple Related WebSite Sets, then the Storage Access API would grant unpartitioned storage access to state which is available to `support.chat` instead of establishing access to partitioned storage specific to the top-level site’s related website set.

-   The Storage Access API would require the browser to prompt the user on both `setmember1.com` and `setmember2.com` separately in order to maintain a continuous session between them.

-   The user would need to interact with the `support.chat` frames on both `setmember1.com` and `setmember2.com` separately.
    This provides friction, since users may not expect to need to accept a prompt on two separate sites to maintain a single chat session.

-   After the user navigates to `setmember2.com`, `support.chat` would need to invoke the Storage Access API just to see if there is a chat session to continue at all.

-   The user would need to interact with `support.chat` in a top-level context.
    Since this domain is used for hosting third-party chat widgets that are not intended to be loaded in a top-level context, this requirement would make it difficult for Storage Access API to meet the use case for `support.chat`.


While it’s possible that a particular Related Website Set includes the chat widget in this example, the mutual exclusivity requirement – the technical check that ensures that an eTLD+1 can only appear in a single set – makes this untenable for third parties.
The mutual exclusivity check is in place to prevent joining Related Website Sets across the web, since the underlying use of the Storage Access API grants set members access to unpartitioned storage.

For these reasons, we wish to propose a new API specifically to allow third-party embeds to request access to a storage partition that is accessible across sites in a single Related Website Set.

### Goals

The goals of this proposal are to:

-   Introduce an API which allows third-party embeds to maintain continuous sessions across websites in the same related website set

-   Only grant access to non-HTTP storage that is strictly limited to activity within the current related website set

-   Design this API to require no prompt, otherwise the Storage Access API meets this use case

### Non-goals

This proposal does not aim to:

-   Introduce a replacement for partitioned storage or Storage Access API

-   Introduce a mechanism that allows embeds to link a user's activity within a Related Website Set to activity on any site outside that set

-   Give the ability for embeds to change which partition their network state or JavaScript storage uses by default

## Proposed Solution

We propose introducing a new asynchronous API to the Document object which provides access to a [storage handle](https://github.com/privacycg/saa-non-cookie-storage).
This storage handle provides access to a partition for JavaScript storage APIs that is shared across all sites within a related website set.
The handle would have access to the embed's partition when the RWS primary site is the top-level site.
Below is an example of code that accesses the handle, `support.chat` could choose to use this API to store a session ID in LocalStorage on `setmember1.com`:

```javascript
// Executed on support.chat's <iframe> embedded in setmember1.com

const handle = await document.requestRelatedWebsitePartition({localStorage: true});

// Can be used to copy state...
const id = window.localStorage.get('sessionId');
handle.localStorage.set('sessionId', id);
```

Then when the user is directed to visit `sitemember2.com`, `support.chat` can access that shared session ID:

```javascript
// Executed on support.chat's <iframe> embedded in setmember2.com

const handle = await document.requestRelatedWebsitePartition({localStorage: true});

// ...to re-establish user ID on the other member site.
const id = handle.localStorage.get('sessionId');
window.localStorage.set('sitemember2_id', id);
```

The new function would accept a plain JavaScript object as input of the following type:

```typescript
interface RequestRelatedWebsitePartitionOptions {
  all?: boolean;
  BroadcastChannel?: boolean;
  createObjectURL?: boolean;
  estimate?: boolean;
  getDirectory?: boolean;
  indexedDB?: boolean;
  localStorage?: boolean;
  locks?: boolean;
  revokeObjectURL?: boolean;
  sessionStorage?: boolean;
}
```

Each field of this interface represents a type of JavaScript storage which would be made available by calling `requestRelatedWebsitePartition`.
If the API is called when the top-level site is not in a related website set then the Promise will reject.
Alternatively, this API could fall back on the Storage Access API.
This is discussed in more detail in the [Handling when RWS is disabled or not supported](#handling-when-rws-is-disabled-or-not-supported) section below.

We have opted to prevent the storage handle from accessing workers to only allow sharing of non-HTTP state across related sites.
We are open to allowing workers if there is demand from the web development community.

### RWS sites must provide a Permissions-Policy or allow attribute

To protect the RWS sites from any malevolent third-parties abusing this API, we require the RWS site to provide a [`Permissions-Policy`](https://w3c.github.io/webappsec-permissions-policy/#permissions-policy-http-header-field) header or iframe [`allow`](https://html.spec.whatwg.org/multipage/iframe-embed-object.html#attr-iframe-allow) attribute to allow third-party origins to use this API.
The default [allowlist](https://developer.mozilla.org/en-US/docs/Web/HTTP/Permissions_Policy#allowlists) for the policy will be `self`.

### RWS sites must opt in with their JSON

RWS owners will be required to explicitly enable use of this API within their set [in the RWS JSON file](https://github.com/GoogleChrome/related-website-sets/blob/main/related_website_sets.JSON) and the [`.well-known` file](https://github.com/GoogleChrome/related-website-sets/blob/main/Well-Known-Specification.md) file in a new `relatedWebsitePartitionEnabled` boolean field.
Setting this value to `true` will allow embeds to use `requestRelatedWebsitePartition` in that set.

Even when this field is set, embeds will still need to use the Storage Access API with a prompt in order to access their unpartitioned state.
This field does not exempt sites from the `Permissions-Policy` or `allow` attribute requirements in the section above this one.

Below is an example of how a JSON entry with this new field may look:

```javascript
{
   "sets": [
      {
        "primary": "https://example.com",
        "associatedSites": [...],
        "relatedWebsitePartitionEnabled": true,
        ...
      },
      ...
  ],
}
```

### Handling when RWS is disabled or not supported

User agents may provide users the option to disable RWS.
Since this new API is closely coupled with RWS, there is a need for us to define its behavior when RWS is disabled.

In the above section, we suggested this API return a Promise that rejects if RWS is enabled and the current top-level site is not in one.
We could also return a Promise that rejects when RWS is disabled or not supported as well.

Another option is having this API fall back on the [Storage Access API for non-cookie storage](https://github.com/privacycg/saa-non-cookie-storage).
When this occurs, the Storage Access API would still require a prompt and user interaction.
This would provide a way for user experiences to remain relatively consistent across browsers that do and do not have RWS enabled.

We look forward to hearing feedback from the developer community with regard to those options.

### Alternatives Considered

#### Extending Storage Access API

One possible alternative design is to propose another extension to the Storage Access API to accept another parameter indicating the embed wishes to access the Related Website Set's shared partition.


We chose to offer a new JavaScript API for this use case to keep the interface for the Storage Access API, a more widely adopted API, unchanged and simple.
Using a new function name would also allow us to mirror the argument interface of the Storage Access API.

However, if the community would rather this proposal be refactored to an extension of the Storage Access API we would be open to that discussion.

#### Extending the Partitioned cookie attribute

Another potential solution would be to allow support.chat to set session cookies using an extension of the [`Partitioned` attribute (CHIPS)](https://github.com/privacycg/CHIPS).
Below is an example of what the Set-Cookie header would look like with this extension:

<pre><code>
Set-Cookie: name=value; Secure; SameSite=None; HttpOnly; <b>Partitioned=RelatedSites;</b>
</code></pre>

We are concerned that overlapping cookie partitions without a way to detect which cookie partition key is being used by a request or frame would add to additional confusion as browsers and publishers continue to more widely adopt CHIPS.

CHIPS originally was designed to use the primary site as the cookie partition key in sites in an RWS by default, but [this integration was deleted](https://github.com/privacycg/CHIPS/pull/44) due to negative external feedback.

#### Storage Buckets

We could also propose an extension to the [Storage Buckets API](https://github.com/WICG/storage-buckets/blob/main/explainer.md) where we introduce a special bucket with either a reserved name or behind a novel method on the StorageBucketManager, like the examples below.

```javascript
async function openBucketExamples() {
  // Primary site
  const bucket = await navigator.storageBuckets.open('https://primary.com');
  
  // Special name
  const bucket = await navigator.storageBuckets.open('related_sites');
  
  // Symbol
  const bucket = await navigator.storageBuckets.open(Symbol.RelatedWebsitePartition);

  // New StorageBucketManager method
  const bucket = await navigator.storageBuckets.openRelatedWebsitePartition();
}
```

We chose to start by proposing a novel API with this as an alternative for simplicity and because Storage Buckets are still in incubation.
Also, there is more ambiguity behind the correct design choice to reserve a special bucket name for a specific use case.
Moreover, Storage Buckets do not have as broad support for storage APIs, for example, it does not include IndexedDB.

#### Collaboration with the Related Website Set

Another option is to have the third-party embed collude with the members of the Related Website Set to pass down their identifier to the embed to use as a common identifier.
While this would avoid having to introduce a new API, this approach would have a few drawbacks:

-   It requires that the RWS members make changes to their code to support a third party, rather than giving the third party the capability to support their use cases independently.

-   In order for `setmember1.co`m to access its common identifier when the user navigates to `setmember2.com`, the Storage Access API would need to be invoked by the RWS member, which cannot be observed by the relying third-party embed.

-   It removes the modularity of `support.chat` and requires them to deeply integrate their software with each individual RWS architecture they are embedded on.

-   `setmember[1|2].com` may not want to share their identifiers with `support.chat`.

#### Vanity domains for third-party services

We could require that a third-party like support.chat use a vanity domain (e.g., `support.chat.customer.com`) to be added to customer.com’s RWS.
We chose to design a novel API so that embeds can build this capability modularly without needing additional coordination with the RWS sites' architecture.

#### Use of an ephemeral partition

One alternative design we considered is to have the handle use a partition that will not be persisted to disk, meaning that the partition will only last for a single browsing session.
We chose not to do this because it would make the API difficult for developers to adopt and would provide no tangible privacy or security benefit.
We also received [feedback](https://github.com/WICG/first-party-sets/issues/94#issuecomment-1998064308) that an ephemeral partition may not meet embeds' use cases.

#### User interaction requirement

We considered requiring that the user interact with a frame belonging to the third-party embed in *at least one* of the RWS member sites.
However we decided against doing so, after receiving [feedback](https://github.com/WICG/first-party-sets/issues/94#issuecomment-1998064308) that it would make adoption more difficult.
Also, the Storage Access API only has this requirement to avoid prompt fatigue, but this proposal's API does not prompt the user.

## Privacy and Security Considerations

### Privacy

#### Comparison to current RWS privacy boundaries

The capabilities introduced by the Related Website Partition API are not completely novel to the browsers that already support RWS.
There are a couple of ways that sites can already effectively implement the Related Website Partition API with current functionality:

-   Third parties who embed themselves on multiple websites in the same RWS can already share information across those sites, provided that  RWS members collaborate with the third party.
    For example, `setmember1.com` and `setmember2.com` could share their own first-party identifier with `support.chat`, allowing `support.chat` to join a user's activity across the two member sites.
    The Related Website Partition API provides a more ergonomic interface for this type of collaboration for both the set member sites and the embedded third-party sites.

-   Third parties with first-party script access on a member site may also replicate this capability without explicit collusion with all RWS members.
    For example, `support.chat` could embed their script on `setmember1.com` and set a cookie.
    The user navigates to `setmember2.com`, which embeds a `setmember1.com` frame which also contains `support.chat`'s script.
    In this case, `support.chat`'s script would still be able to access the cookie set on `setmember1.com` after they are granted storage access.

The requirements that the member sites provide a [`Permissions-Policy`](#rws-sites-must-provide-a-permissions-policy-or-allow-attribute) and that the RWS [add a boolean to its JSON entry](#rws-sites-must-opt-in-with-their-json) both mean that the RWS members must permit the third party to join sessions across member sites; thus preserving the need for first-party collaboration.

#### Comparison to Storage Access API

Third-party sites embedded on RWS sites could rely on the [Storage Access API](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/StorageAccessAPI/explainer.md) to access their unpartitioned `SameSite=None` cookies or their first-party storage partition.
Despite the fact that this approach would prompt the user, this type of storage still has more potential for pervasive tracking since it would allow the embed to link a user's activity within an RWS to their activity outside the set.
With the Related Website Partition API, this type of joining is not possible.

This rationale also applies to set member sites, which can use the Storage Access API or [`requestStorageAccessFor()`](https://github.com/privacycg/requestStorageAccessFor) to access unpartitioned cookies or first-party storage without needing to prompt the user.
We encourage RWS sites to instead rely on the Related Website Partition API as well, since the partition it makes available cannot be used to link the user's activity in and outside of the set.

#### Rationale for RWS JSON boolean requirement

A third party with first-party script access may give itself permission to use this API if the RWS site embeds the third-party site's script in their top-level document.
This would allow the script to inject its own `<iframe>` and the third party could grant its own frame permission to use the Related Website Partition API by adding its own `allow` attribute.
The requirement for the RWS to add the [boolean field to their JSON](#rws-sites-must-opt-in-with-their-json) to enable this capability adds another layer of opt-in to protect set owners from third parties having unexpected access to user activity across the set.

#### Impact on Per-Site Privacy Limitations

Some Privacy Sandbox APIs such as [Private State Tokens](https://developers.google.com/privacy-sandbox/protections/private-state-tokens) and the [Attribution Reporting API](https://developers.google.com/privacy-sandbox/private-advertising/attribution-reporting) use the top-level site as a boundary for rate limiting or budgeting.
The Related Website Partition API allows third parties to join activity across different sites in the same RWS, allowing them to join information from these limited APIs.
To mitigate this, we should consider whether it is feasible to key these APIs' limits on the RWS primary site when they are being used by embeds within an RWS.

#### Preventing Potential for Cross-Site Tracking

It would be problematic if an embed could use `requestRelatedWebsitePartition` to link a user's activity outside a single Related Website Set with their activity in that set (or at least doing so must require the Storage Access API separately).
For set members leveraging the Storage Access API to support cross-set use cases, we have a mutual exclusivity requirement in place to prevent cross-site tracking.
Since third-party widgets can be embedded in multiple sets, we cannot enforce mutual exclusivity for these third parties across individual RWS when they use the Storage Access API.
In order to prevent cross-site tracking without mutual exclusivity, we have the browser restrict the storage partition so that it is only available to the embed within the current RWS.
The outcome we want to avoid by introducing this API is to introduce a means for third-party embeds to link activity between sites that are not in the same Related Website Set.

### Security

Since each sibling frame has its own separate JavaScript execution context, each will need to call `requestRelatedWebsitePartition` independently in order to access the handle.

Accessing the handle requires the execution of JavaScript, so only third-party `<iframe>` can access this API.
The handle can only write to its own distinct partition.
This design prevents any state stored in this handle from leaking in cross-site subresource requests outside of a frame that does not invoke this API, which makes using CSRF to access this storage impossible.

## References

-   [Attribution Reporting for Web overview | Privacy Sandbox | Google for Developers](https://developers.google.com/privacy-sandbox/private-advertising/attribution-reporting)

-   [CHIPS integration with First-Party Sets · Issue #94 · WICG/first-party-sets](https://github.com/WICG/first-party-sets/issues/94)

-   [GoogleChrome/related-website-sets](https://github.com/GoogleChrome/related-website-sets/)

-   [HTML Standard](https://html.spec.whatwg.org/)

-   [MicrosoftEdge/MSEdgeExplainers: Home for explainer documents originated by the Microsoft Edge team](https://github.com/MicrosoftEdge/MSEdgeExplainers/)

-   [Permissions Policy](https://w3c.github.io/webappsec-permissions-policy/#permissions-policy-http-header-field)

-   [Private State Tokens | Privacy Sandbox | Google for Developers](https://developers.google.com/privacy-sandbox/protections/private-state-tokens)

-   [privacycg/CHIPS: A proposal for a cookie attribute to partition cross-site cookies by top-level site](https://github.com/privacycg/CHIPS)

-   [privacycg/requestStorageAccessFor: A proposed extension to the Storage Access API and discussion of how it may be integrated with First-Party Sets](https://github.com/privacycg/requestStorageAccessFor)

-   [privacycg/saa-non-cookie-storage: Extending Storage Access API (SAA) to non-cookie storage](https://github.com/privacycg/saa-non-cookie-storage)

-   [privacycg/storage-partitioning: Client-Side Storage Partitioning](https://github.com/privacycg/storage-partitioning)

-   [WICG/first-party-sets](https://github.com/WICG/first-party-sets/)

-   [WICG/storage-buckets: API proposal for managing multiple storage buckets](https://github.com/WICG/storage-buckets/)
