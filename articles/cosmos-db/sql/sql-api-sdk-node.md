---
title: 'Azure Cosmos DB: SQL Node.js API, SDK & resources'
description: Learn all about the SQL Node.js API and SDK including release dates, retirement dates, and changes made between each version of the Azure Cosmos DB Node.js SDK.
author: anfeldma-ms
ms.service: cosmos-db
ms.subservice: cosmosdb-sql
ms.devlang: javascript
ms.topic: reference
ms.date: 12/09/2021
ms.author: anfeldma
ms.custom: devx-track-js

---
# Azure Cosmos DB Node.js SDK for SQL API: Release notes and resources
[!INCLUDE[appliesto-sql-api](../includes/appliesto-sql-api.md)]
> [!div class="op_single_selector"]
> * [.NET SDK v3](sql-api-sdk-dotnet-standard.md)
> * [.NET SDK v2](sql-api-sdk-dotnet.md)
> * [.NET Core SDK v2](sql-api-sdk-dotnet-core.md)
> * [.NET Change Feed SDK v2](sql-api-sdk-dotnet-changefeed.md)
> * [Node.js](sql-api-sdk-node.md)
> * [Java SDK v4](sql-api-sdk-java-v4.md)
> * [Async Java SDK v2](sql-api-sdk-async-java.md)
> * [Sync Java SDK v2](sql-api-sdk-java.md)
> * [Spring Data v2](sql-api-sdk-java-spring-v2.md)
> * [Spring Data v3](sql-api-sdk-java-spring-v3.md)
> * [Spark 3 OLTP Connector](sql-api-sdk-java-spark-v3.md)
> * [Spark 2 OLTP Connector](sql-api-sdk-java-spark.md)
> * [Python](sql-api-sdk-python.md)
> * [REST](/rest/api/cosmos-db/)
> * [REST Resource Provider](/rest/api/cosmos-db-resource-provider/)
> * [SQL](sql-query-getting-started.md)
> * [Bulk executor - .NET v2](sql-api-sdk-bulk-executor-dot-net.md)
> * [Bulk executor - Java](sql-api-sdk-bulk-executor-java.md)

|Resource  |Link  |
|---------|---------|
|Download SDK  |   [NPM](https://www.npmjs.com/package/@azure/cosmos) 
|API Documentation  |  [JavaScript SDK reference documentation](/javascript/api/%40azure/cosmos/)
|SDK installation instructions  |  [Installation instructions](https://github.com/Azure/azure-sdk-for-js)
|Contribute to SDK | [GitHub](https://github.com/Azure/azure-sdk-for-js/tree/main)
| Samples | [Node.js code samples](sql-api-nodejs-samples.md)
| Getting started tutorial | [Get started with the JavaScript SDK](sql-api-nodejs-get-started.md)
| Web app tutorial | [Build a Node.js web application using Azure Cosmos DB](sql-api-nodejs-application.md)
| Current supported platform | [Node.js v12.x](https://nodejs.org/en/blog/release/v12.7.0/) - SDK Version 3.x.x<br/>[Node.js v10.x](https://nodejs.org/en/blog/release/v10.6.0/) - SDK Version 3.x.x<br/>[Node.js v8.x](https://nodejs.org/en/blog/release/v8.16.0/) - SDK Version 3.x.x<br/>[Node.js v6.x](https://nodejs.org/en/blog/release/v6.10.3/) - SDK Version 2.x.x<br/>[Node.js v4.2.0](https://nodejs.org/en/blog/release/v4.2.0/)- SDK Version 1.x.x<br/> [Node.js v0.12](https://nodejs.org/en/blog/release/v0.12.0/)- SDK Version 1.x.x<br/> [Node.js v0.10](https://nodejs.org/en/blog/release/v0.10.0/)- SDK Version 1.x.x

## Release notes

Release history is maintained in the azure-sdk-for-js repo, for detailed list of releases, see the [changelog file](https://github.com/Azure/azure-sdk-for-js/blob/main/sdk/cosmosdb/cosmos/CHANGELOG.md#release-history).

## Migration guide for breaking changes

If you are on an older version of the SDK, it's recommended to migrate to the 3.0 version. This section details the improvements you would get with this version and the bug fixes made in the 3.0 version.
### Improved client constructor options

Constructor options have been simplified:

* masterKey was renamed key and moved to the top level
* Properties previously under options.auth have moved to the top level

```javascript
// v2
const client = new CosmosClient({
    endpoint: "https://your-database.cosmos.azure.com",
    auth: {
        masterKey: "your-primary-key"
    }
})

// v3
const client = new CosmosClient({
    endpoint: "https://your-database.cosmos.azure.com",
    key: "your-primary-key"
})
```

### Simplified query iterator API

In v2, there were many different ways to iterate or retrieve results from a query. We have attempted to simplify the v3 API and remove similar or duplicate APIs:

* Remove iterator.next() and iterator.current(). Use fetchNext() to get pages of results.
* Remove iterator.forEach(). Use async iterators instead.
* iterator.executeNext() renamed to iterator.fetchNext()
* iterator.toArray() renamed to iterator.fetchAll()
* Pages are now proper Response objects instead of plain JS objects
* const container = client.database(dbId).container(containerId)

```javascript
// v2
container.items.query('SELECT * from c').toArray()
container.items.query('SELECT * from c').executeNext()
container.items.query('SELECT * from c').forEach(({ body: item }) => { console.log(item.id) })

// v3
container.items.query('SELECT * from c').fetchAll()
container.items.query('SELECT * from c').fetchNext()
for await(const { result: item } in client.databases.readAll().getAsyncIterator()) {
    console.log(item.id)
}
```

### Fixed containers are now partitioned

The Cosmos service now supports partition keys on all containers, including those that were previously created as fixed containers. The v3 SDK updates to the latest API version that implements this change, but it is not breaking. If you do not supply a partition key for operations, we will default to a system key that works with all your existing containers and documents.

### Upsert removed for stored procedures

Previously upsert was allowed for non-partitioned collections, but with the API version update, all collections are partitioned so we removed it entirely.

### Item reads will not throw on 404

const container = client.database(dbId).container(containerId)

```javascript
// v2
try {
    container.items.read(id, undefined)
} catch (e) {
    if (e.code === 404) { console.log('item not found') }
}

// v3
const { result: item }  = container.items.read(id, undefined)
if (item === undefined) { console.log('item not found') }
```

### Default multi-region writes

The SDK will now write to multiple regions by default if your Cosmos configuration supports it. This was previously opt-in behavior.

### Proper error objects

Failed requests now throw proper Error or subclasses of Error. Previously they threw plain JS objects.

### New features

#### User-cancelable requests

The move to fetch internally allows us to use the browser AbortController API to support user-cancelable operations. In the case of operations where multiple requests are potentially in progress (like cross-partition queries), all requests for the operation will be canceled. Modern browser users will already have AbortController. Node.js users will need to use a Polyfill library

```javascript
 const controller = new AbortController()
 const {result: item} = await items.query('SELECT * from c', { abortSignal: controller.signal});
 controller.abort()
```

#### Set throughput as part of db/container create operation

```javascript
const { database }  = client.databases.create({ id: 'my-database', throughput: 10000 })
database.containers.create({ id: 'my-container', throughput: 10000 })
```

#### @azure/cosmos-sign

Header token generation was split out into a new library, @azure/cosmos-sign. Anyone calling the Cosmos REST API directly can use this to sign headers using the same code we call inside @azure/cosmos.

#### UUID for generated IDs

v2 had custom code to generate item IDs. We have switched to the well known and maintained community library uuid.

#### Connection strings

It is now possible to pass a connection string copied from the Azure portal:

```javascript
const client = new CosmosClient("AccountEndpoint=https://test-account.documents.azure.com:443/;AccountKey=c213asdasdefgdfgrtweaYPpgoeCsHbpRTHhxuMsTaw==;")
Add DISTINCT and LIMIT/OFFSET queries (#306)
 const { results } = await items.query('SELECT DISTINCT VALUE r.name FROM ROOT').fetchAll()
 const { results } = await items.query('SELECT * FROM root r OFFSET 1 LIMIT 2').fetchAll()
```

### Improved browser experience

While it was possible to use the v2 SDK in the browser, it was not an ideal experience. You needed to Polyfill several node.js built-in libraries and use a bundler like webpack or Parcel. The v3 SDK makes the out of the box experience much better for browser users.

* Replace request internals with fetch (#245)
* Remove usage of Buffer (#330)
* Remove node builtin usage in favor of universal packages/APIs (#328)
* Switch to node-abort-controller (#294)

### Bug fixes
* Fix offer read and bring back offer tests (#224)
* Fix EnableEndpointDiscovery (#207)
* Fix missing RUs on paginated results (#360)
* Expand SQL query parameter type (#346)
* Add ttl to ItemDefinition (#341)
* Fix CP query metrics (#311)
* Add activityId to FeedResponse (#293)
* Switch _ts type from string to number (#252)(#295)
* Fix Request Charge Aggregation (#289)
* Allow blank string partition keys (#277)
* Add string to conflict query type (#237)
* Add uniqueKeyPolicy to container (#234)

### Engineering systems
Not always the most visible changes, but they help our team ship better code, faster.

* Use rollup for production builds (#104)
* Update to Typescript 3.5 (#327)
* Convert to TS project references. Extract test folder (#270)
* Enable noUnusedLocals and noUnusedParameters (#275)
* Azure Pipelines YAML for CI builds (#298)

## Release & Retirement Dates

Microsoft provides notification at least **12 months** in advance of retiring an SDK in order to smooth the transition to a newer/supported version. New features and functionality and optimizations are only added to the current SDK, as such it is recommended that you always upgrade to the latest SDK version as early as possible.

| Version | Release Date | Retirement Date |
| --- | --- | --- |
| 3.4.2 | November 7, 2019   | --- |
| 3.4.1 | November 5, 2019   | --- |
| 3.4.0 | October 28, 2019   | --- |
| 3.3.6 | October 14, 2019   | --- |
| 3.3.5 | October 14, 2019   | --- |
| 3.3.4 | October 14, 2019   | --- |
| 3.3.3 | October 3, 2019    | --- |
| 3.3.2 | October 3, 2019    | --- |
| 3.3.1 | October 1, 2019    | --- |
| 3.3.0 | September 24, 2019 | --- |
| 3.2.0 | August 26, 2019    | --- |
| 3.1.1 | August 7, 2019     | --- |
| 3.1.0 |July 26, 2019 |--- |
| 3.0.4 |July 22, 2019 |--- |
| 3.0.3 |July 17, 2019 |--- |
| 3.0.2 |July 9, 2019 |--- |
| 3.0.0 |June 28, 2019 |--- |
| 2.1.5 |March 20, 2019 |--- |
| 2.1.4 |March 15, 2019 |--- |
| 2.1.3 |March 8, 2019 |--- |
| 2.1.2 |January 28, 2019 |--- |
| 2.1.1 |December 5, 2018 |--- |
| 2.1.0 |December 4, 2018 |--- |
| 2.0.5 |November 7, 2018 |--- |
| 2.0.4 |October 30, 2018 |--- |
| 2.0.3 |October 30, 2018 |--- |
| 2.0.2 |October 10, 2018 |--- |
| 2.0.1 |September 25, 2018 |--- |
| 2.0.0 |September 24, 2018 |--- |
| 2.0.0-3 (RC) |August 2, 2018 |--- |
| 1.14.4 |May 03, 2018 |August 30, 2020 |
| 1.14.3 |May 03, 2018 |August 30, 2020 |
| 1.14.2 |December 21, 2017 |August 30, 2020 |
| 1.14.1 |November 10, 2017 |August 30, 2020 |
| 1.14.0 |November 9, 2017 |August 30, 2020 |
| 1.13.0 |October 11, 2017 |August 30, 2020 |
| 1.12.2 |August 10, 2017 |August 30, 2020 |
| 1.12.1 |August 10, 2017 |August 30, 2020 |
| 1.12.0 |May 10, 2017 |August 30, 2020 |
| 1.11.0 |March 16, 2017 |August 30, 2020 |
| 1.10.2 |January 27, 2017 |August 30, 2020 |
| 1.10.1 |December 22, 2016 |August 30, 2020 |
| 1.10.0 |October 03, 2016 |August 30, 2020 |
| 1.9.0 |July 07, 2016 |August 30, 2020 |
| 1.8.0 |June 14, 2016 |August 30, 2020 |
| 1.7.0 |April 26, 2016 |August 30, 2020 |
| 1.6.0 |March 29, 2016 |August 30, 2020 |
| 1.5.6 |March 08, 2016 |August 30, 2020 |
| 1.5.5 |February 02, 2016 |August 30, 2020 |
| 1.5.4 |February 01, 2016 |August 30, 2020 |
| 1.5.2 |January 26, 2016 |August 30, 2020 |
| 1.5.2 |January 22, 2016 |August 30, 2020 |
| 1.5.1 |January 4, 2016 |August 30, 2020 |
| 1.5.0 |December 31, 2015 |August 30, 2020 |
| 1.4.0 |October 06, 2015 |August 30, 2020 |
| 1.3.0 |October 06, 2015 |August 30, 2020 |
| 1.2.2 |September 10, 2015 |August 30, 2020 |
| 1.2.1 |August 15, 2015 |August 30, 2020 |
| 1.2.0 |August 05, 2015 |August 30, 2020 |
| 1.1.0 |July 09, 2015 |August 30, 2020 |
| 1.0.3 |June 04, 2015 |August 30, 2020 |
| 1.0.2 |May 23, 2015 |August 30, 2020 |
| 1.0.1 |May 15, 2015 |August 30, 2020 |
| 1.0.0 |April 08, 2015 |August 30, 2020 |

## FAQ
[!INCLUDE [cosmos-db-sdk-faq](../includes/cosmos-db-sdk-faq.md)]

## See also
To learn more about Cosmos DB, see [Microsoft Azure Cosmos DB](https://azure.microsoft.com/services/cosmos-db/) service page.
