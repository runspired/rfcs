- Start Date: 2018-02-02
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# [WIP] Ember Data Collections

**This RFC should be considered the roughest of rough drafts**

## Summary

Improve the ergonomics of fetching and operating on collection endpoints in ember-data.

## Motivation

Today, the use of `RecordArray` and `PromiseArray` proxies have a great cost on the mental model
 required to use `ember-data`, and foment the impression that `Ember` is too complex and not 
 `Just Javascript™`.

These proxies have a significant performance cost as well as these proxies are not cheap and
 the classes built with them are built on a complex inheritance chain. What's more, the patterns
 used to work around the limitations of the proxies more often than not are themselves significant
 bottlenecks in applications.

This RFC seeks to remove one of the more painful ergonomic and performance experiences in `ember-data`
 by providing a `Just Javascript™` primitive upon which application and addon developers can design
 more complex features.

#### Ergonomic and performance issues with the current state of things

In `ember-data` today, arrays of records are typically fetched from an API via either `store.query`
 or `store.findAll`.  These methods both return a `PromiseArray` wrapping either a `RecordArray` in
 the case of `store.findAll` or an `AdapterPopulatedRecordArray` in the case of `query`.

In the case of `store.query`, `meta` and `links` from the response are available on the
 `AdapterPopulatedRecordArray`, but only `meta` is available by proxy on the `PromiseArray`.
 Because folks do not correctly grok the async nature of this proxy, but notice that it "seems
 to just work in templates", they improperly reach into `PromiseArray.content` to get access
 to the `RecordArray`. Most often this happens in computed properties.

**Example Bad Pattern**

```js
export default Route.extend({
  model() {
    return this.get('store').findAll('juju');
  }
});

export default Controller.extend({
  badJuju: computed('model.length', function() {
    return this.get('model.content')
      .filter(r => r.get('isBad'));
  })
});
```

`AdapterPopulatedRecordArray` and `RecordArray`, which it extends, both extend `Ember.ArrayProxy`. Their
 content, instead of being an array of records, is an array of `InternalModel`s: a private construct.
 Although in the near future this will be an array of `ModelData`s, this will remain a point of confusion.
 Ember-data uses the `objectAt` method to switch `InternalModel` for a lazily-materialized record instance
 on access.
 
 For example, the following pattern does not work as expected, because `firstJuju` will be `undefined`,
 as `recordArray` is not a true array.

**Example of not "Just Javascript" 1** 

```js
export default Route.extend({
  model() { 
    return this.get('store').query('juju', {})
      .then(recordArray => ({
        firstJuju: recordArray[0],
        allJuju: recordArray
      }));
  }
});
```

**Example of not "Just Javascript" 2** 

The following anti-pattern also does not work as expected. Here, the developer
reaches in to access the content of the `RecordArray` seeking direct access to
the records. However, `firstJuju` will be an `InternalModel`, not a `Record`.

```js
export default Route.extend({
  model() { 
    return this.get('store').query('juju', {})
      .then(recordArray => ({
        firstJuju: recordArray.get('content')[0],
        allJuju: recordArray
      }));
  }
});
```

Adding to the confusion, these classes contain many public-looking-but-actually-private properties and
 methods in addition to `meta`, `links`.  Indeed, the actual "public" API surface area of
 [`RecordArray`](https://www.emberjs.com/api/ember-data/3.0/classes/DS.RecordArray) and
 [`AdapterPopulatedRecordArray`](https://www.emberjs.com/api/ember-data/3.0/classes/DS.AdapterPopulatedRecordArray)
 doesn't even include `meta`, `links` or `length`. That these classes are extensions of `ArrayProxy`
 is a side-note. Most of the `ArrayProxy` methods are unsafe for app developers to utilize, as they
 expect `InternalModel` or require the use of `ArrayProxy.replace()` which is overwritten to throw
 an error.  Incidentally, that [error](https://github.com/emberjs/data/blob/v3.0.0/addon/-private/system/record-arrays/record-array.js#L85)
 is what documents and alerts app developers to their one crutch: `toArray`. For the most part, even
 though `ArrayProxy` is a public class in Ember, it's public looking (and public documented if a developer
 realizes the `ArrayProxy` connection and foes and finds it) methods are private from the perspective of
 `RecordArray`.
 
`toArray()` creates a divergence in the API, as developers must now reason about whether they
   are interacting with a `RecordArray` or the result of calling `toArray()` and similarly must divine 
   whether this means `meta` and `links` are available to them or not.

 Unfortunately, the only clear and reliable way to avoid private API usage and manage an array of records
  is with `toArray()`.  However, this introduces another problem as it forks the array at the call site
  leading developers to lose the ability to respond to or make updates appropriately. When relied upon, 
  this negates any benefits of `lazy-materialization` (if there are any, it is unclear whether `RecordArray`s
  benefit from lazy materialization to the same degree as relationships).
  
**Example unsafe toArray usage**

```js
export default Route.extend({
  model() { 
    return this.get('store').query('juju', {})
      .then(recordArray => recordArray.toArray());
  },
  
export default Controller.extend({
  actions: {
    removeJuju(juju) {
      this.get('model').removeObject(juju);
    },
    addJuju(juju) {
      this.get('model').pushObject(juju);
    },
    updateJuju() {
      this.get('model').update();
    }
  }
});
```

Note how even after calling `toArray()` a user must use `removeObject` and `pushObject` to see
 updates propagate to their templates, using `splice` and `push` here do not "just work". However,
 this pattern is still susceptible to data flow problems. The `updateJuju` action will update
 the `recordArray` returned by the query; however, those updates won't be reflected into the array
 created by calling `toArray`.  Developers that recognize this issue will fall back to the following
 pattern instead.

```js
export default Route.extend({
  model() { 
    return this.get('store').query('juju', {})
      .then(r => r); // ensure model is the recordArray, not the promiseArray
  },

export default Controller.extend({

  jujuArr: computed('model.length', function() {
    return this.get('model').toArray();
  }),

  actions: {
    removeJuju(juju) {
      this.get('jujuArr').removeObject(juju);
    },
    addJuju(juju) {
      this.get('jujuArr').pushObject(juju);
    },
    updateJuju() {
      this.get('model').update();
    }
  }
});
```

Even with this careful preparation, developers must still face several challenges. How do they resolve
local changes post-update? How can they push local changes back onto the `RecordArray` itself? While
these are not questions that this RFC answers, moving a step closer to "just Javascript" will help us
to provide these answers in the future.

One of the more frustrating aspects of `store.query` is all calls to it bypass the cache, yet it is the
 only method by which to make a request for a specific collection today via a store that touts caching as
 its selling point.  `store.query` is also the mandatory method to use if you want access to `links` and `meta`.

The alternative to `store.query`, `store.findAll` comes with its own set of caveats.

`findAll` returns the `live-array` result of `store.peekAll`, meaning that all known records of a
 type are included, in unreliable order, regardless of state.  Effectively, this makes the result of a request
 via `store.findAll` useless without an array computed and using `RecordArray.toArray().filter(() => {})`.
 This negates the benefits of the `live-array` as well as of `lazy-materialization` when relied upon,
 as it often is.

`store.findAll` also cannot support `meta` and `links` and poorly supports `pagination`, because it is the agglomeration
 of all requests for records of a specific `type` via any find method or local creation.

The `PromiseArray` and `RecordArray` proxies also introduce a great amount of friction when attempting
 to manage the state of a list on the client side. `PromiseArray`, while proxying to an array, does not
 itself have any of the methods of `Array` or Ember's `MutableArray` or `ArrayProxy` (such as `toArray`).
 `RecordArray` meanwhile extends `ArrayProxy` thus exposing `MutableArray` methods.  These methods differ
 from the methods available in Native JS arrays and contribute to the feeling that working with array data
 in Ember is not "just Javascript".

Currently, app developers must first determine if they are working with the `PromiseArray`, the `RecordArray`,
 or the result of calling `toArray` to know how to access and manipulate the array. In the case of working with
 the native `Array` result of calling `toArray`, changes will not reflect back into record state and become
 difficult to manage or save, but working with `RecordArray` directly does not have clear guides or patterns
 established for managing membership and manipulating item order and saving these changes.

The layers of indirection `RecordArray` proxies have a great cost on the mental model required to use `ember-data`,
 and a significant performance cost as well, as these proxies are not cheap and the classes built with them are
 built on a complex inheritance chain.  What's more, the patterns used to work around the limitations of the proxies
 more often than not are themselves significant bottlenecks in applications.

Altogether, these issues make managing arrays of records one of the more painful ergonomic and performance experiences 
 in `ember-data`.

#### Ergonomic and performance benefits to be gained from a simpler mental model

**TODO expand on these points**

- aligning usage more closely with json-api principles enables us to provide more robust primitives
- treating json-api documents as first-class citizens solves real-world problems with meta/links/errors/
  collections that do not map as efficiently to records.
- allows end users to more easily solve querying, caching, and updating edge cases on their own
  - enable more advanced patterns, improve flexibility (membership, pagination, caching, meta)
- no surprises, "just JS", easier to debug
- separate the fetch from the current data
- pave a path for improvements and simplifications across ember-data in single-resource and
  relationship layer as well

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
Art exists [here](https://github.com/emberjs/rfcs/pull/161).*

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