---
title: Use w3name for changing data
description: Learn how to use w3name for content that evolves over time
---

import { Tabs, TabItem } from 'components/tabs/tabs';
import Callout from 'components/callout/callout';

# Update your data with w3name

All the data you [store on web3.storage](/docs/how-tos/store) is [content-addressed](/docs/concepts/content-addressing), which means that you always get location-independent, cryptographically verifiable links to your content.

Content addressing is a very powerful tool, but because the address is directly derived from the content, it is limited by definition to content that already exists, and any changes to the content will result in an entirely new address.

When you need to refer to something that might change over time, or that may not exist yet at all, content addressing alone isn't enough. You also need a way to update things as they change without breaking all of your links and references.

[w3name][w3name-github] is a service that provides secure, stable identifiers for data that changes over time (also known as "mutable" data). It uses the [IPNS][ipns-docs] protocol to seamlessly interoperate with the public IPFS network, so the links you create with w3name can be used with any IPFS client software or HTTP gateway.

All records created and updated using w3name are signed locally with each user's private publishing key. This means that the w3name service never sees your keys, and it also doesn't require any authentication to use - no account or API keys required!

In this guide, we'll discover how to use the JavaScript `w3name` package to create and manage name records for data stored with web3.storage.

## What's in a name?

w3name is similar to the Domain Name System (DNS), in that it allows you to query for an identifier and resolve the latest value. Unlike DNS, however, you cannot choose your own identifiers, and the "names" produced by IPNS and w3name are not human-readable, like `web3.storage` or `ipfs.io`.

The "names" in IPNS and w3name are unique identifiers, and each one is a string representation of the public half of a cryptographic key-pair.

Each "name" maps to a "record", which contains:

- The CID of the IPFS content that the name is pointing to.
- A sequence number and validity date.
- A verification signature, created using the private key, providing proof that this record was generated by the key holder.

Because the verification key is embedded in the name, all names created with w3name are "self-certifying," meaning that you can validate any record published to that name without needing any other authority or "source of truth" than the name itself.

## Getting started

The easiest way to use the w3name service is with the [w3name][w3name-npm] JavaScript library, which provides methods for creating, signing, publishing and resolving records.

### Install client library

The [w3name npm package][w3name-npm] provides a JavaScript / TypeScript API for creating and managing IPNS name records.

Install the library into your project using your favorite JS dependency manager:

<Tabs>
<TabItem value="npm" label="npm">
```bash
npm install w3name
```
</TabItem>
<TabItem value="yarn" label="yarn">
```bash
yarn add w3name
```
</TabItem>
</Tabs>

To import the library into your code, use the syntax that matches your JS environment.

<Tabs>
<TabItem value="esm" label="ES Modules">
```js
import * as Name from 'w3name';
```
</TabItem>
<TabItem value="cjs" label="Common JS">
```
const Name = require('w3name');
```
</TabItem>
</Tabs>

## Creating a new name

You can create a new IPNS name with the `Name.create` function:

```js
const name = await Name.create();
```

<Callout type="info">
  The example above uses the [`await` operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await), because `Name.create()` returns a `Promise`. To use  `await`, the code must be placed inside an [`async` function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function). Alternatively, you can resolve the promise using `.then()`, for example:

```js
Name.create().then(name => {
  // do something with name here.
});
```

</Callout>

You can see the name as a string by calling `.toString()` on it:

```js
console.log('created new name: ', name.toString());
// will print something similar to:
//  created new name: k51qzi5uqu5di9agapykyjh3tqrf7i14a7fjq46oo0f6dxiimj62knq13059lt
```

<Callout type="info">
  So far, the signing key for your new name only exists in memory. To save it for publishing updates, see [Saving and loading keys](#saving-and-loading-keys) below.
</Callout>

## Publishing the first revision

Above, we saw how to [create a name](#creating-a-new-name), but so far there's no value associated with it.

To publish a value, we need to create a "revision." A revision contains a value that can be published to w3name, along with some IPNS metadata. All revisions have a sequence number that gets incremented each time the name is updated with a new value.

To create the initial revision for a name, use the `Name.v0` function:

```js
// value is an IPFS path to the content we want to publish
const value = '/ipfs/bafkreiem4twkqzsq2aj4shbycd4yvoj2cx72vezicletlhi7dijjciqpui';
// since we don't have a previous revision, we use Name.v0 to create the initial revision
const revision = await Name.v0(name, value);
```

We now have a `revision` object that's ready to publish, but so far it's only in memory on our local machine. To publish the value to the network, use `Name.publish`:

```js
await Name.publish(revision, name.key);
```

See below to learn how to [update the name](#publishing-an-updated-revision) with subsequent revisions.

## Publishing an updated revision

Each revision contains a sequence number, which must be incremented when publishing a new revision. Once you've [published the initial revision](#publishing-the-first-revision), you can use the `Name.increment` function to create a new revision based on the previously published one.

For example, if our initial revision is stored in a variable called `revision`, we can create a new revision `nextRevision` using `Name.increment`:

```js
const nextValue = '/ipfs/bafybeiauyddeo2axgargy56kwxirquxaxso3nobtjtjvoqu552oqciudrm';
// Make a revision to the current record (increments sequence number and sets value)
const nextRevision = await Name.increment(revision, nextValue);
```

Publication works the same as with the initial revision:

```js
await Name.publish(nextRevision, name.key);
```

If you no longer have the old revision locally, you can [resolve the current value](#getting-the-latest-revision) and use the returned revision as input to `Name.increment`.

Note that you must have the original signing key in order to publish updates. See [Saving and loading keys](#saving-and-loading-keys) below to learn about key management.

## Getting the latest revision

You can resolve the current value for any name using the `Name.resolve` function.

```js
const name = Name.parse('k51qzi5uqu5di9agapykyjh3tqrf7i14a7fjq46oo0f6dxiimj62knq13059lt');
const revision = await Name.resolve(name);
console.log('Resolved value:', revision.value);
```

## Saving and loading keys

To create revisions to a name after publication, you'll need the original signing key. You can get the binary representation of a name with the `key.bytes` property, which can then be saved to disk or stored in a secure key management system.

```js
import fs from 'fs';

async function saveSigningKey(name, outputFilename) {
  const bytes = name.key.bytes;
  await fs.promises.writeFile(outputFilename, bytes);
}
```

Later, you can use `Name.from` to convert from the binary representation to an object suitable for publishing revisions.

```js
import fs from 'fs';

async function loadSigningKey(filename) {
  const bytes = await fs.promises.readFile(filename);
  const name = await Name.from(bytes);
  return name;
}
```

<Callout type="warning">
Be careful where you save your keys! Your private signing keys allow the holder to update your published records. Be careful to save your keys to a secure location, and never store your private keys in a source code repository.
</Callout>

[concepts-content-addressing]: /docs/concepts/content-addressing
[w3name-github]: https://github.com/web3-storage/w3name
[w3name-npm]: https://www.npmjs.com/package/w3name
[ipns-docs]: https://docs.ipfs.tech/concepts/ipns/
