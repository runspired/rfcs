- Start Date: 2018-02-02
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# [SUPER WIP] Ember Data Collections

**This RFC should be considered the roughest of rough drafts**

## Summary

Improve the ergonomics of fetching and operating on collection endpoints in ember-data.

## Motivation

In `ember-data` today, arrays of records are typically fetched from an API via either `store.query`
or `store.findAll`.  These methods both return a `PromiseArray` wrapping either a `RecordArray` in
the case of `store.findAll` or an `AdapterPopulatedRecordArray` in the case of `query`.

In the case of `store.query`, `meta` and `links` from the response are available on the `RecordArray`,
but only `meta` is proxied to the `PromiseArray`.  Folks reach into `PromiseArray.content` to get access
to the `RecordArray` improperly, because they do not understand the async nature of this proxy but notice
 that it "seems to just work in templates".

`AdapterPopulatedRecordArray` and `RecordArray`, which it extends, both extend `Ember.ArrayProxy`.
Their content, instead of being an array of records, is an array of `InternalModel`s. A private
construct. Ember-data uses the `objectAt` method to lazily materialize the records for these internal-models
on access.  This is another very confusing thing, even if good for perf.

Adding to the confusion, these classes contain many public-looking-but-actually-private properties and
methods in addition to `meta`, `links` and `toArray()`.  The sad reality is that `toArray()` is the only
 endorsed semi-sane way of interacting with the record-array, although it forks the array at the call-point
 leading developers to lose the ability to respond to or make updates appropriately. This negates the
 benefits of `lazy-materialization` when relied upon, as it often is.
 
`RecordArray.toArray()` creates a divergence in the API, as developers must now reason about whether they
  are interacting with a `RecordArray` or the result of calling `toArray()` and similarly must devine 
  whether this means `meta` and `links` are available to them or not.

One of the more frustrating aspects of `store.query` is all calls to it bypass the cache, yet it is the
only method by which to make a request for a specific collection today via a store that touts caching as
 its selling point.  `store.query` is also the mandatory method to use if you want access to `links` and
`meta`.

The alternative to `store.query`, `store.findAll` comes with its own set of caveats.

For starters, it returns the `live-array` result of `store.peekAll`, meaning that all known records of a
 type are included, in random order, regardless of state.  Effectively, this makes the result of a request
  via `store.findAll` useless without an array computed and using `RecordArray.toArray().filter(() => {})`.
  This negates the benefits of the `live-array` as well as of `lazy-materialization` when relied upon,
  as it often is.

`store.findAll` also cannot support `meta` and `links` and poorly supports `pagination`, because it is the agglomeration
 of all requests for records of a specific `type` via any find method or local creation.

**TODO issues with updating arrays and/or manipulating array order**

Altogether, these issues make managing arrays of records one of the more painful ergonomic and performance experiences 
in `ember-data` today.

So why this particular solution?

- aligning usage more closely with json-api principles enables us to provide more robust primitives
- findAll meta / pagination / integrity problems
- query pagination and cache problems
- RecordArray, ManyArray, and their various proxies
  are difficult to reason about and debug, and surprise
  folks by not being "just arrays".
- separate the fetch from the current data
- enable more advanced patterns, improve flexibility (membership, pagination, caching, meta)
- pave a path for improvements and simplifications across ember-data in single-resource and
  relationship layer as well

## Detailed design

### Querying Data

`store.queryURL(cacheKey, url)`

`queryURL` would return a `Document` (potentially a document presenting as a `Collection`).

### Associated Classes

`Document`

For a single resource, data would point at a single resource. For a collection, it
would be an array.

```js
class Document {
  fetch() {}
  meta;
  links;
  data;
}
```

`Collection`

Collection would extend document to add access to individual pages. Pages would be 
 `Document`s thus exposing the appropriate nested meta, links, and data subset.
 
The `data` member for a collection would contain “all results” from all pages, and
 the top level meta and links would effectively reflect the meta and links of “page 0”.

```js
class Collection extends Document {
  pages {
     '1': <Document>
  };
}
```

### Caching Data and Mechanics

Documents would be stored in a cache, similar to the identity map, in a manner that allows
them to retain cache consistency but also store additional information such as `meta` `links`
and `errors`.

Collections would be stored in a two-level cache. The first level being "as a document" with
an internal cache for documents by page.

```js
collections {
   '/users/1/foos/2/bars' {
     1 { meta links data }
     2 { meta links data }
     3 { meta links data }
   }
}
```

`cacheKey` would be an identifier surfacing information about `type` and `id` and for collections
`page`.

`CacheKey`

```js
{
  type,
  id,
  page
}
```

### Busting the Cache

**TODO**

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

Deprecation Churn.

Deprecate-able store methods:

 - `store.findByIds`
 - `store.findMany`
 - `store.findAll`
 - `store.query`
 
Potentially deprecate-able store methods (once follow up RFCs have landed
for aligning single-resources and relationships with the `Document` and `Collection`
API)

 - `store.findHasMany`
 - `store.find`
 - `store.findBelongsTo`
 - `store.findRecord`
 - `store.queryRecord`
 - `store.filter` (moved to addon, we should finish off this method, svelte!)
 
 Intimate classes we would remove or be closer to removing
  (some may need hang around for the deprecation lifecycle or
   for other RFCs to land)
 
 - `RecordArray`
 - `AdapterPopulatedRecordArray`
 - `FilteredRecordArray`
 - `PromiseArray`
 - `RecordArrayManager`
 
 Internally, `CollectionManager` would replace `RecordArrayManager` to fulfill new requirements. Instead of `RecordArray`
 we would ideally just use arrays; however, questions regarding lazy-materialization must be answered first.
 
#### JSON-API

Not everyone wants to learn json-api, and while it makes sense internally to ember-data
aligning public APIs to mirror it's structure could encounter some resistance. That said,
the issues this data structure resolves also helps make the case for why json-api is
so well thought out!


## Alternatives

- Don't do this: Continued maintenance headaches and support requests around cache problems, pagination,
  manipulation, and proxy issues.
- No top-level collection management, end users must handle combination of paginated data
  on their own, cache-key becomes simpler because URL is always the key.

## Unresolved questions

- References vs Resources and mixing them for operations
- Record materialization (currently lazy). Can we make it not-lazy now that there are paths for pushing data into the
  store without requiring materialization? Should we explicitly make a public thing for the array (`Identifier`) instead?
- should we cleanup the relationship layer first (ala https://github.com/emberjs/data/pull/4882)
- should `collection()` descriptor come as a separate RFC?
- RFC for `buildURL()` potentially as `store.buildURL`, helper and how to use it to provide a url to `store.queryURL`
- should we fast follow with an RFC for similarly cleaning up `belongsTo` and single
  resources to ensure that these APIs move to a nice future together in step?
- should `queryURL` only work for collections initially to avoid questions about
  the previous bullet point?
- Should we have separate `queryResource` and `queryCollection` methods that allow us to differentiate
  collection cache-keys more clearly?
