- Start Date: 2018-02-02
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# [WIP] Ember Data Collections

**This RFC should be considered the roughest of rough drafts**

## Summary

Improve the ergonomics of fetching and operating on collection endpoints in ember-data.

## Motivation

The use of `RecordArray` and `PromiseArray` proxies have a great cost on the mental model
 required to use `ember-data`, and foment the impression that `Ember` is too complex and not 
 `Just Javascript™`.

In addition to the mental overhead, these proxies have a significant performance cost as the proxy
 mechanism is expensive and these ember-data's proxy classes are constructed via a complex chain
 of inheritance.
 
Patterns used to work around the limitations of the proxies often introduce significant additional
 mental and performance penalties.

This RFC seeks to remove this painful ergonomic and performance experience by providing a 
 `Just Javascript™` primitive upon which application and addon developers can design more complex
 features.

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
  `querying`, `caching` and `updating` edge cases on their own, while substantially improving the ergonomics
  of manging collection membership, `pagination`, and accessing `meta`.


## Detailed design

### The Collection Class

```ts

abstract class Link {
  href: String;
  meta?: Object;
}

abstract class Links {
  self: String|Link
  first?: String|Link
  last?: String|Link
  prev?: String|Link
  next?: String|Link
}

class Collection {
  public meta?: Object;
  public links: Links;
  public data?: Array<Object>;
  public errors?: Object;

  private store;

  fetch(options: { requestOptions: Object, params: Object }): Promise<Collection> {}
  
  next(): Promise<Collection> {}

  prev(): Promise<Collection> {}
  
  first(): Promise<Collection> {}
  
  last(): Promise<Collection> {}
}
```

### Fetching Collections

A new "finder" method would be introduced to `DS.Store` for requesting a collection.
With this method in place, we would deprecate the `findAll` and `query` methods on `DS.Store`.

*Note: This method would be eventually be deprecated in favor of a method with the same signature
but allowing for fetching of single-resource documents as well.*

```ts
class Store {
  fetchCollection(url: String, options: Object): Promise<Collection> {}
}
```

**TODO explain why fetchCollection and not findCollection, and details about shouldReload / shouldBackgroundReload support**

### Pushing Collections

Similarly, a new `push` method would be introduced to `DS.Store` for pushing a collection
into the store.

*Note: This method would similarly be eventually deprecated in favor of a method with the same
signature but allowing for the pushing of single-resource documents into the store as well. Prior
Art exists [here](https://github.com/emberjs/rfcs/pull/161). and [here](https://github.com/emberjs/rfcs/pull/160)*

```ts
class Store {
  pushCollection(jsonApiDocument: Object): Collection {}
}
```

**TODO How does this fit into the Adapter interface without leading to more code bloat?**

#### Collection Caching

Collections would be cached by `url`, but with the same ability to bust the cache when
fetching a specific collection as when finding a single `Record`, either by a flag when
using `fetch` or by implementing the appropriate adapter hook (`shouldBackgroundRefresh` or
`shouldRefresh`).

As an important note, a `Collection` need not have any `data`, and the available properties
on a `Collection` MUST observe the rules of a [`json-api` Document](http://jsonapi.org/format/#document-top-level).

```ts
abstract class CollectionCache  {
  [url: String]: Collection
}
```

#### Pagination Support

Because collections are cached by `URL`, and because `Links` are `URL`s, not only are pagination
results easy to associate with each other, but are even cached as individual collections. Async
iteration becomes possible by calling `collection.next()`, which returns a promise that resolves
to the next collection in the list, or `null` if no `next` link is present in `Links`.

##### Managing the combined Pagination results of multiple Collections

At times an Application may want to display the combined results of paginated requests. While the
specifics of this feature is outside the purview of this RFC, achieving such with the `Collection`
feature would be relatively simple given the ease of iterating pagination links, with the flexibility
to handle any number of UX and Performance strategies.

### No changes peekAll / No peekCollection

This RFC considers changes to the behavior of `peekAll` to return a `Collection` instead of a `RecordArray`
 to be outside the scope of this RFC. It also declines to proffer a `peekCollection` API for checking
 for the existence of a collection locally.  For the time being, `fetchCollection` gives users the
 granular ability to decide whether to hit the network or not. Changes to the `peek` API ought to be
 scoped to a new RFC.

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

#### Doesn't address relationships

**TODO a bit about this**

#### An End to Lazy Record Materialization

**TODO explain why this is actually a good thing, even for perf**

#### An End to promise-proxies

**TODO explain why this is actually a good thing, even for templates**

#### Perceived Churn

The primary drawback to this RFC is perceived "deprecation churn". We must make a
clear case for why this presents a better future, and how it fits into the comprehensive
for ember-data's future.

Deprecate-able methods:

 - `store.findAll`
 - `store.query`
 - `Adapter.findAll`
 - `Adapter.query`
 
Deprecate-able intimate classes:

 - `FilteredRecordArray`
 - `PromiseArray`
 - `RecordArrayManager`
 
Internally, `CollectionManager` would replace `RecordArrayManager` to fulfill new requirements.

Closer to being deprecate-able methods (once follow up RFCs have landed
for aligning single-resources and relationships with the `Document` and `Collection`
API, and for building URLs / fetching data)

 - `store.findByIds`
 - `store.findMany`
 - `store.findHasMany`
 - `Adapter.findByIds`
 - `Adapter.findMany`
 - `Adapter.findHasMany`
 - `store.find`
 - `store.filter` (already moved to addon and asserted, we should finish off removing this method, svelte!)
 
 Closer to being deprecate-able intimate classes (once follow up RFCs have landed proposing new `peek` APIs)

 - `RecordArray`
 - `AdapterPopulatedRecordArray`
 
#### JSON-API hesitation

Not everyone wants to learn json-api, and while it makes sense internally to ember-data
aligning public APIs to mirror it's structure could encounter some resistance. That said,
the issues this data structure resolves also helps make the case for why json-api is
so well thought out!

## Alternatives

- Don't do this: Continued maintenance headaches and support requests around cache problems,
   pagination, manipulation, and proxy issues.
- Land Collection as a private feature replacing parts of the internals, bring public over time.
- **TODO is an addon feasible**

## Unresolved questions

- ResourceIdentifier vs Resource membership in a Collection, json-api allows for collections
   of identifiers
- META RFC or RFC Issue for presenting the comprehensive vision for `ember-data` into which
   this RFC fits.
- RFC for `buildURL()` potentially as `store.buildURL`, helper and how to use it to provide
   a url to `store.fetchCollection`
- RFC for a simpler adapter/serializer model, reducing the method bloat seen today and separating
   core store concerns from the network layer.