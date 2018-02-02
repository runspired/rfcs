- Start Date: 2018-02-02
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# [SUPER WIP] Ember Data Collections

**This RFC should be considered the roughest of rough drafts**

## Summary

New apis that replace the existing `store.findAll` and `store.query` methods to 
 enable better fetch, update, pagination and cache management of collection
 API endpoints in ember-data.

## Motivation

- aligning usage more closely with json-api principles enables us to provide more robust primitives
- findAll meta / pagination / integrity problems
- query pagination and cache problems
- RecordArray, ManyArray, and their various proxies
  are difficult to reason about and debug, and surprise
  folks by not being "just arrays".
- separate the fetch from the current data
- enable more advanced patterns, improve flexibility
- pave a path for improvements and simplifications across ember-data in single-resource and
  relationship layer as well

## Detailed design

### Querying Data

`store.queryURL(cacheKey, url)`

`queryURL` would return a `Document` (potentially a document presenting as a `Collection`).

`store.buildURL()`

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
 
 Internally, `CollectionManager` would replace `RecordArrayManager` to fulfill new requirements.
 
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

- should we cleanup the relationship layer first (ala https://github.com/emberjs/data/pull/4882)
- should `collection()` descriptor come as a separate RFC?
- should `buildURL()` helper come as a separate RFC, potentially as `store.buildURL`? 
- should we fast follow with an RFC for similarly cleaning up `belongsTo` and single
  resources to ensure that these APIs move to a nice future together in step?
- should `queryURL` only work for collections initially to avoid questions about
  the previous bullet point?
- Should we have separate `queryResource` and `queryCollection` methods that allow us to differentiate
  collection cache-keys more clearly?
