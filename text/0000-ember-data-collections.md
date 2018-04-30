- Start Date: 2018-02-02
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Ember Data Documents

## Summary

Improve the ergonomics of handling document-level information and of 
fetching and operating on collection endpoints in ember-data.

## Motivation

Users of `ember-data` struggle with proxies and promise-proxies: they don’t understand them and they don’t
 know how to work with them.  Array proxies especially have a great cost on the mental model required to
 use `ember-data`, and give the impression that `Ember` is too complex and not `Just Javascript™`.

In addition to mental overhead, proxies have significant performance and complexity costs. The proxy 
 mechanism is expensive and `ember-data` proxy classes are constructed via a complex chain of inheritance.
 
Patterns used to work around proxies such as `toArray` and `.content` often introduce significant additional
 mental and performance penalties.

This RFC seeks to remove this painful ergonomic and performance experience by providing a lightweight solution that
 doesn't involve Ember's object model or proxies as a primitive upon which application and addon developers can design
 more complex features.

### A simpler primitive for the future

`ember-data` seeks to provide a flexible, efficient, and powerful core experience up which addons and
  apps can ergonomically implement complex requirements. While `ember-data` will not require users to
  adopt the `json-api` standard for their API, `json-api` is a well developed standard upon which this
  core experience can be built. By aligning core `ember-data` primitives to `json-api` concepts,
  we gain a readily understood mental model of how various concepts and primitives fit together, and
  documentation to support it.
  
In practice, this is a formalization of what `ember-data` already provides: an interface through which
  to work with data from any source in a unified format.  Already today, that format is often close to
  `json-api` internally.  By standardizing this format, we can more confidently expose more behaviors
  and information to app and addon developers, empowering them more than ever before.

This RFC works to align just one `ember-data` concept with `json-api`, by replacing the `RecordArray` and
  `AdapterPopulatedRecordArray` proxies with `Collection`, and unlocking the power of the `url` as a
  cache-key. By doing so, we solve real-world problems with accessing `meta`, `links`, `errors` and
  `Collections` that don't map directly to `Record`s.  We also allow app engineers to more easily solve
  `querying`, `caching` and `updating` edge cases on their own, while providing better ergonomics
  for manging collection membership, `pagination`, and accessing `meta`.

## Detailed design

### The Collection Class

```ts

abstract class Link {
  href: String;
  meta?: Object;
}

abstract class Links {
  self: String|Link
}

abstract class PaginationLinks extends Links {
  first?: String|Link
  last?: String|Link
  prev?: String|Link
  next?: String|Link
}

class Document {
  public meta?: Object;
  public links: Links;
  public data?: Object;
  public errors?: Object;

  private store;

  fetch(options: { requestOptions: Object, params: Object }): Promise<Document> {}
  
  next(): Promise<Collection> {}

  prev(): Promise<Collection> {}
  
  first(): Promise<Collection> {}
  
  last(): Promise<Collection> {}
}

class Collection extends Document {
  public links: PaginationLinks;
  public data?: Array<Object>;
 
  next(): Promise<Collection> {}

  prev(): Promise<Collection> {}
  
  first(): Promise<Collection> {}
  
  last(): Promise<Collection> {}
}
```

### Fetching Documents and Collections

A new "finder" method would be introduced to `DS.Store` for requesting a json-api document.
Requests that return a `data` member containing an `Array` would return the `Collection` sub-class.

```ts
class Store {
  fetchDocument(query: { url: String, cacheDocument: Boolean }, options: Object): Promise<Document> {}
}
```

// TODO the "query" aspect of this needs cleaned up, as does an explantion of `cacheDocument`.

`fetchDocument` represents the foundation for a re-imagining and simplification of the `finder` story
  in `ember-data`. Today, each new scenario for finding requires new methods on the store, on adapters,
  and significant internal plumbing. By relying on the `url`, and moving `url` to the forefront we can
  trim the fat and avoid this bloat, while still retaining the flexibility to pass queries.
  
  This focus on the url as well as the desire to remain true to the `Just Javascript™` story is why
  we've chosen `fetchDocument` instead of `findDocument` for the name of this new API.

```ts
class Adapter {
  fetchDocument(query: { url: String, cacheDocument: Boolean }, options: Object): Promise<jsonApiDocument> {}
}
```

Available `options` would include support for `shouldReload` and `shouldBackgroundReload`. Query params
for the request should already be included on the provided `url`.

### Pushing Documents into the store

Similarly, a new `push` method would be introduced to `DS.Store` for pushing a document
into the store.

* Prior Art exists [here](https://github.com/emberjs/rfcs/pull/161). and [here](https://github.com/emberjs/rfcs/pull/160)*

```ts
class Store {
  pushDocument(jsonApiDocument: Object): Promise<Document> {}
}
```

#### Document Caching

Documents would be cached by `url` when `cacheDocument` is `true` (the default), but with the same
ability to bust the cache when fetching a specific collection as when finding a single `Record`, 
either by a flag when using `fetch` or by implementing the appropriate adapter hook
 (`shouldBackgroundRefresh` or `shouldRefresh`).

As an important note, a `Document` need not have any `data`, and the available properties
on a `Document` MUST observe the rules of a [`json-api` Document](http://jsonapi.org/format/#document-top-level).

```ts
abstract class DocumentCache  {
  [url: String]: Document
}
```

#### Pagination Support

**Simple Case**

Because documents are cached by `URL`, and because `Links` are `URL`s, not only are pagination
 results for collections easy to associate with each other, but are even cached as individual
 collections. Async iteration becomes possible by calling `collection.next()`, which returns a 
 promise that resolves to the next collection in the list, or `null` if no `next` link is present
 in `Links`.

**Complex Case**

While query should always be serializable to a `url` (at least for cache key purposes) `links` may
not be present for `next()` and `prev()` and similar methods, but enough data is likely present in
the original `query` and `meta` for an app developer to formulate the correct request. We will want
a story for generating a cacheKey to check for the presence of and a way to properly build the request
to pass to `fetchDocument`.

// TODO cache-key-for-request and fetch extension point

##### Managing the combined Pagination results of multiple Collections

At times an Application may want to display the combined results of paginated requests. While the
specifics of this feature is outside the purview of this RFC, achieving such with the `Collection`
feature would be relatively simple given the ease of iterating pagination links, with the flexibility
to handle any number of UX and Performance strategies.

### peeking a Collection

This RFC considers changes to the behavior of `peekAll` to return a `Collection` instead of a `RecordArray`
 to be outside the scope of this RFC. It also declines to proffer a `peekCollection` API for checking
 for the existence of a collection locally.  For the time being, `fetchCollection` gives users the
 granular ability to decide whether to hit the network or not. Changes to the `peek` API ought to be
 scoped to a new RFC.
 
### url-building

- show url-builder helper pattern

## How we teach this

**TODO**

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

- `hasMany` Relationships are also essentially `collections`. However, their plumbing today is significantly different
   from that of `RecordArray` and `AdapterPopulatedRecordArray`, and while a mirror API is desired, the specifics have
   been omitted in favor of a separate RFC. This will lead to a short-term divergence in API design between the two.
- We would no longer lazily materialize the records contained in a `Collection`. While this might have averse performance
   ramifications for a few apps, it is arguable that unlike `hasMany` relationships, a fetched `Collection` is more
   probably to be immediately consumed.  Additionally, the cost of materializing a record is something we are working
   to pay-down, and should no longer be considered a deciding factor in API design. Lastly, because we would be creating
   and managing fewer Array proxies there is a significant probability that any performance benefits lost are recovered.
- We would no longer wrap return values in a proxy ala `PromiseArray` or `PromiseObject`. Some applications utilize
   these promise-proxies to eagerly return available data from model hooks or fetch data from computed properties. Should
   some users still require or desire this extra wrapper, they could easily wrap the `fetchCollection` promise on their own.
   Everyone else would gain the benefit of clearer APIs.
- Not everyone wants to learn json-api, and while it makes sense internally to ember-data aligning public APIs to mirror
   it's structure could encounter some resistance. However, the issues this data structure resolves also helps make the 
   case for why `json-api` is so well thought out. Building around a standard will be easier to teach than explaining an
   ad-hoc new one.

#### Perceived Churn

The primary drawback to this RFC is perceived "deprecation churn". While we believe this RFC presents early steps
towards a simpler and more powerful core `ember-data` experience, it also introduces a series of concepts that
over time and in conjunction with other RFCs would change the expected experience including:

- Treating the url as an `id` for the cache for some `Documents`
- Separating out url-building from the adapter
- Separating and clearly defining the boundary between adapter and store 
- Replacing the explosion of find methods with a unified url-based fetch method
- Conceptualizing api responses as `Documents` instead of single resources or arrays of resources.
- reducing and eliminating proxies as possible

## Alternatives

- Don't do this: Continued maintenance headaches and support requests around cache problems,
   pagination, manipulation, and proxy issues.
- Land Collection as a private feature replacing parts of the internals, bring public over time.

## Unresolved questions

- ResourceIdentifier vs Resource membership in a Collection, json-api allows for collections
   of identifiers
- RFC for a simpler adapter/serializer model, reducing the method bloat seen today and separating
   core store concerns from the network layer.
- Is Collection membership immutable? Or do we allow untracked mutation? This RFC provides no
  mechanism for tracking changes or initiating a `save`. Should it? What would that even mean?
  Is it enough to let users figure this out on their own? Adding a save API to collections 
  could be done via RFC or addon.