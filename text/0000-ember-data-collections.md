- Start Date: 2018-02-02
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# [SUPER WIP] Ember Data Collections

**This RFC should be considered the roughest of rough drafts**

Also note, it currently contains references MANY MANY more changes than it will
ultimately propose. Ultimately, we want relationships and find methods for both
single resources and collections to have consistent APIs. However, such a large
scope of changes is greater than one RFC, and it is most likely the smartest course
of action to start small with just introducing collections and then moving to 
enhance single-resources and finally updating relationships to have matching semantics.

## Summary

New apis that replace the existing `find` methods and `hasMany` descriptor to 
 enable better fetch, update, pagination and cache management of collection
 API endpoints in ember-data.

## Motivation

- aligning usage more closely with json-api principles enables us to provide more robust primitives
- findAll meta / pagination / integrity problems
- query pagination and cache problems
- hasMany pagination problems
- hasMany fetching problems
- hasMany / recordArray order manipulation struggles
- hasMany / recordArray membership struggles
- improve our ability to refactor and cleanup the relationship layer
- RecordArray, ManyArray, and their various proxies
  are difficult to reason about and debug, and surprise
  folks by not being "just arrays".
- separate the fetch from the current data
- enable more advanced patterns, improve flexibility
- fix several `hasMany` API constraints such as implicit inverses and auto-fetch

## Detailed design

### Querying Data

`store.queryURL(cacheKey, url)`

`queryURL` would return a `Document` (potentially a document presenting as a `Collection`).

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

### Allowing relationships to use this nice new ability as well

Currently `hasMany` uses ember arrays and array proxies in a similar manner to `RecordArray`.
We could use this opportunity to expose collections as a descriptor and give these new abilities
to relationships as well.

`collection()`



`store.buildURL()`

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

The sheer number of deprecations.

Deprecated store methods:

 - `store.findByIds`
 - `store.findMany`
 - `store.findHasMany`
 - `store.findAll`
 - `store.query`
 
 Additional store methods capable of being deprecated
 
 - `store.find`
 - `store.findBelongsTo`
 - `store.findRecord`
 - `store.queryRecord`
 - `store.filter` (moved to addon, we should finish off this method, svelte!)
 
 Other deprecations
 
 - `hasMany()` descriptor (and by proxy `ManyArray`)
 
 Other potential deprecations
 
 - `belongsTo()` descriptor
 
 Intimate classes we would remove (some may hang around for the deprecation lifecycle)
 
 - `ManyArray`
 - `RecordArray`
 - `AdapterPopulatedRecordArray`
 - `FilteredRecordArray`
 - `PromiseArray`
 
JSON-API

Not everyone wants to learn json-api, and while it makes sense internally to ember-data
aligning public APIs to mirror it's structure could encounter some resistance. That said,
the issues this data structure resolves also helps make the case for why json-api is
so well thought out!


## Alternatives

- Continued maintenance headaches in the relationship layer and support requests around
  cache problems, pagination, manipulation, and proxy issues.
- No top-level collection management, end users must handle combination of paginated data
  on their own, cache-key becomes simpler because URL is always the key.

## Unresolved questions

- should we cleanup the relationship layer first (ala https://github.com/emberjs/data/pull/4882)
- should `collection()` descriptor come as a separate RFC?
- should `buildURL()` helper come as a separate RFC, potentially as `store.buildURL`? 
- should we fast follow with an RFC for similarly cleaning up `belongsTo` and single
  resources to ensure that these APIs move to a nice future together in step?
- should `queryURL` only work for collections initially to avoid questions about
  the previous bullet point?
- Should we have separate `queryResource` and `queryCollection` methods that allow us to differentiate
  collection cache-keys more clearly?


> Optional, but suggested for first drafts. What parts of the design are still
TBD?
