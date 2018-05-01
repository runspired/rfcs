- Start Date: 2018-05-01
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Ember Data | RequestManager

## Summary

Reduce and simplify the mental model of fetching data.

## Motivation

- simpler adoption story: "from fetch to full ember-data"
- smaller core footprint and clear separation of concerns amongst ember-data primitives
- more flexibility and control over requests for end users

## Detailed design

### Stage 1

In stage one, we implement a new base adapter method `fetchDocument` to replace
`finders` and to re-implement our existing store calls to adapter find methods
over-top of. Here, we introduce a very primitive `Query` class with only one mandatory
key: `op`.

```typescript
class Adapter {
  fetchDocument(query: Query, options: Options): Promise<JsonApiDocument> {}
}

class Query {
  op: String;
  [String]: any;
}

abstract class Options {
  shouldBackgroundReload: Boolean;
  shouldReload: Boolean;
  [String]: any;
}

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

abstract class Resource {
  type: String;
  id: String;
  attributes?: Object;
  relationships?: Object;
  meta?: Object;
  links?: Links;
}

abstract class JsonApiDocument {
  meta?: Object;
  links: Links|PaginationLinks;
  data?: Resource|Array<Resource>;
  included?: Array<Resource>;
  errors?: Object;
}
```

Example conversions:

**before**
```js
let modelClass = store.modelFor(modelName);
let recordArray = store.peekAll(modelName);
let snapshotRecordArray = recordArray._createSnapshot(originalOptions);

return adapter.findAll(store, modelClass, sinceToken, snapshotRecordArray)
  .then((rawPayload) => {
    let jsonApiDocument = serializer.normalizeResponse(store, modelClass, rawPayload, null, 'findAll');
    return store.push(jsonApiDocument);
  });
```

**after**
```js
let options = originalOptions.adapterOptions;
let query = {
  op: "findAll",
  _store: store,
  modelName,
  sinceToken,
  options: originalOptions,
  _snapshots() {
    let recordArray = store.peekAll(modelName);
    let snapshots = recordArray._createSnapshot(originalOptions);
    snapshots.adapterOptions = options;
    return snapshots;
  }
};

delete originalOptions.adapterOptions;

return adapter.fetchDocument(query, options)
  .then((jsonApiDocument) => store.push(jsonApiDocument));
```

### Stage 2

Implement more advanced `Query` and `Mutation` interfaces and builders that implement
 `Request`. The details here are a bit more fuzzy, but this is where we move more towards
 orbit's model of query building and do a similar request cleanup for mutations.

### Stage 3

In stage 3, we separate out existing adapters, serializers, transforms, and errors into
  an addon.  We pull existing `finders` into a new addon (separate from the legacy adapter story)
  as well, introducing `store.fetchDocument` in it's place.  These addons can ship with ember-data
  by default for a transition period, and be dropped by build time flag (ala jQuery removal).
  
We introduce a new default `RequestManager` which is also responsible for normalization.
  As a transition phase, `normalizeResponse` can delegate to a serializer's `normalizeResponse`.

```typescript
class Response {
  get headers(): Object {}
  get responseData(): any {}
  get status(): Number {}
}

class RequestManager {
  fetchDocument(query: Query, options: Options): Promise<JsonApiDocument> {
    // responsible for calling fetch
    //  looking up serializer, and normalizing response
    //   to return a json-api document
  }

  normalizeResponse(query: Query, otpions: Options, response: Response): JsonApiDocument {}

  fetch(query: Query, options: Options): Promise<Response> {
    // calls fetch or xmlhttprequest or websocket etc.
    //   is responsible for calling headerFor / optionsFor / urlFor
    //   bodyFor / methodFor
  }

  // this needs to be async to support cached security headers living in indexdb
  //   the other methods are async to maintain a unified interface
  headersForRequest(query: Query, options: Options): Promise<Object> {}

  bodyForRequest(query: Query, options: Options): Promise<> {}

  methodForRequest(query: Query, options: Options): Promise<String> {}
  
  optionsForRequest(query: Query, options: Options): Promise<Object> {}

  urlForRequest(query: Query, options: Options): Promise<String> {
     // separate path for request from params for request?
  }
}

class Model {}

class Document {
  meta?: Object;
  links: Links|PaginationLinks;
  data?: Model|Array<Model>;
  errors?: Object;
}

class Store {
  fetchDocument(query: Query, options: Options): Promise<Document> {
    // calls adapter.fetchDocument
    // pushes data and included into the store
    // returns a Document (with no included member) and `data` pointing at materialized records.
  }
}
```


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

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
