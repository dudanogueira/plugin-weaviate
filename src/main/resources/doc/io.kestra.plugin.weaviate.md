# How to use the Weaviate plugin

Search, insert, and manage objects and collections in Weaviate from Kestra flows. The plugin uses the Weaviate Java client v6, which talks to Weaviate over REST and gRPC and requires **Weaviate 1.32 or later**.

## Connection

Set `url` to your Weaviate URL, e.g. `http://localhost:8080`. For authenticated clusters, set `apiKey`. Store secrets in [secrets](https://kestra.io/docs/concepts/secret). Set connection properties on each task, or once per flow with `pluginDefaults` as in the example below.

Queries and batch inserts use gRPC, so the gRPC port must be reachable from the Kestra worker:

- Self-hosted Weaviate: gRPC defaults to the host of `url` on port `50051`. Override it with `grpcHost` and `grpcPort`. gRPC uses TLS when `url` is https.
- Weaviate Cloud: URLs ending in `.weaviate.cloud` or `.weaviate.network` are detected automatically and connect to the matching gRPC endpoint on port 443. `apiKey` is required.

Pass extra headers with `headers`, for example vectorizer API keys such as `X-OpenAI-Api-Key`. They are sent on both REST and gRPC calls.

## Tasks

`SchemaCreate` creates a collection. Set `className`, `fields` as a map of property name to Weaviate data type (`text`, `int`, `number`, `boolean`, `date`, `uuid`, their `[]` array forms, `blob`, `geoCoordinates`, `phoneNumber`), and optionally a `vectorizer`. Set `skipIfExists: true` to make the task idempotent.

`BatchCreate` inserts objects in batches of `batchSize`. Set `objects` to a `kestra://` URI of an ION file or an inline list of maps. Use `uuidKey` and `vectorKey` to take object IDs and precomputed vectors from the input. The task fails if Weaviate rejects any object, unless `failOnError` is false; rejected objects are then listed in the `errors` output.

`Query` searches a collection. Choose the search with `searchType`:

| `searchType` | Uses |
|---|---|
| `FETCH_OBJECTS` (default) | lists objects, optionally filtered |
| `BM25` | keyword search on `query`, optionally limited to `queryProperties` |
| `HYBRID` | blends keyword and vector search on `query`; `alpha` sets the balance, `vector` is optional |
| `NEAR_TEXT` | vector search on `query`, vectorized by the collection's vectorizer |
| `NEAR_VECTOR` | vector search on `vector` |

Each result row is `{uuid, properties, metadata, vector?, vectors?}`. Request metadata such as `DISTANCE` or `SCORE` with `returnMetadata`, and vectors with `includeVector`. Control result handling with `fetchType` (default `STORE`).

`Delete` removes objects from a collection: set `objectId` to delete one object, or filters to delete every match. Deleting all objects requires `deleteAll: true`. The task fails if Weaviate rejects any deletion, unless `failOnError` is false.

## Filters

`Query` and `Delete` accept two filter forms, which can be combined (both must match):

- `filter`: a map of property to value, combined with AND. Text values use `LIKE`, so `*` wildcards work; other values use `EQUAL`.
- `filters`: a list of `{property, operator, value, valueType}` conditions, combined with `filtersOperator` (`AND` or `OR`). Operators: `EQUAL`, `NOT_EQUAL`, `GREATER_THAN`, `GREATER_THAN_EQUAL`, `LESS_THAN`, `LESS_THAN_EQUAL`, `LIKE`, `CONTAINS_ANY`, `CONTAINS_ALL`, `CONTAINS_NONE`, `IS_NULL`. Set `valueType: DATE` or `UUID` for dates and UUIDs written as strings. Use `_id` to filter by object UUID.

## Example: load and search a collection

This flow creates a collection, loads a few articles, and runs a hybrid search for the `question` input. Weaviate turns `title` and `body` into vectors with the collection's vectorizer, so no embeddings are computed in the flow. `skipIfExists` and `uuidKey` make the flow safe to run again: the collection is reused, and objects with the same `id` are replaced instead of duplicated.

`TEXT2VEC_MODEL2VEC` needs the `text2vec-model2vec` module on a self-hosted server. On Weaviate Cloud, use `TEXT2VEC_WEAVIATE` and add `apiKey` to `pluginDefaults`.

```yaml
id: weaviate_load_and_search
namespace: company.team

inputs:
  - id: question
    type: STRING
    defaults: how do I keep my data safe

tasks:
  - id: create_collection
    type: io.kestra.plugin.weaviate.SchemaCreate
    className: Article
    vectorizer: TEXT2VEC_MODEL2VEC
    skipIfExists: true
    fields:
      title: text
      body: text
      category: text

  - id: insert
    type: io.kestra.plugin.weaviate.BatchCreate
    className: Article
    uuidKey: id
    objects:
      - id: 2f1c9a52-6a0e-4c43-9d63-1b6f1d0a0001
        title: Backups that actually restore
        body: Test your restore process regularly and keep copies in another region.
        category: operations
      - id: 2f1c9a52-6a0e-4c43-9d63-1b6f1d0a0002
        title: Encrypting data at rest
        body: Turn on disk encryption and rotate keys to protect stored data.
        category: security
      - id: 2f1c9a52-6a0e-4c43-9d63-1b6f1d0a0003
        title: Scaling reads with replicas
        body: Add read replicas to spread query load across nodes.
        category: operations
      - id: 2f1c9a52-6a0e-4c43-9d63-1b6f1d0a0004
        title: Least-privilege API keys
        body: Give each service its own key with only the permissions it needs.
        category: security

  - id: search
    type: io.kestra.plugin.weaviate.Query
    className: Article
    searchType: HYBRID
    query: "{{ inputs.question }}"
    alpha: 0.5
    limit: 3
    returnProperties:
      - title
      - category
    returnMetadata:
      - SCORE
    fetchType: FETCH

  - id: log
    type: io.kestra.plugin.core.log.Log
    message: |
      {%- for row in outputs.search.rows %}
      {{ row.metadata.score | numberFormat('0.00') }} {{ row.properties.title }} ({{ row.properties.category }})
      {%- endfor %}

pluginDefaults:
  - type: io.kestra.plugin.weaviate
    values:
      url: http://localhost:8080
```

With the default question, the log shows "Encrypting data at rest" first. Each row in `outputs.search.rows` is `{uuid, properties, metadata}`. With the default `fetchType: STORE`, rows go to an ION file at `outputs.search.uri` instead, which suits large result sets.

## Upgrading from the GraphQL-based plugin

Plugin versions built on the Weaviate Java client v6 replace the GraphQL-based `Query` with the structured search above, and requires Weaviate 1.32 or later. The main changes:

- `Query.query` is now search text, not a GraphQL string. Use `searchType`, `limit`, `returnProperties` and `filters` instead of a `Get { ... }` query.
- Query rows changed from `{ClassName: {...}}` to `{uuid, properties, metadata}`: `row.ClassName.title` becomes `row.properties.title`, and `_additional.id` becomes `uuid`.
- `BatchCreate` now returns `insertedCount`, `failedCount` and `errors`, and fails on rejected objects by default.
- `Delete` with neither `objectId` nor filters now requires `deleteAll: true`, and fails on rejected deletions by default (`failOnError`).
- `SchemaCreate` takes one data type per field; the legacy `string` type is created as `text`. It defaults to `vectorizer: NONE` instead of the server's default vectorizer, so set `vectorizer` if you relied on it.
- Aggregations, `groupBy`, generative search and `Explore` are not supported. Stay on a plugin version built on the v5 client, or call the Weaviate REST API with `io.kestra.plugin.core.http.Request`.
