# Index.json Spec

**Last updated:** Nov 2, 2018

![Deprecated](https://img.shields.io/badge/Draft-Deprecated-ff69b4.svg) ![Not implemented](https://img.shields.io/badge/Status-Not%20implemented-red.svg)

*Deprecated, replaced by folder extensions. See [#13](https://github.com/beakerbrowser/specs/issues/13) and [#14](https://github.com/beakerbrowser/specs/issues/14) for discussion.*

A metadata file used to describe a folder.

```js
// /documents/index.json
{
  "title": "Documents",
  "description": "Documents, spreadsheets, presentations, etc"
}
```

To describe a folder, create an `index.json` file in the folder being described. The file's schema should match this specification.

This file differs from the [`dat.json`](https://github.com/datprotocol/dat.json) file by describing the folder instead of the entire dat; therefore it may exist next to the `/dat.json` in the root in order to describe the root folder instead of the entire dat.

## Fields

### `title`

Optional String.

### `description`

Optional String.

### `type`

Optional String or Array of Strings. A list of the folder's types. Defaults to `"none"`.

|Value|Description|
|-|-|
|`none`|Has no meaning. This is the default type if nothing is specified.|
|`protected`|The folder is protected from applications. Applications are not allowed to write to the folder.|
|`objectstore`|The folder contains JSON object-files. Applications must request access to the files using the [`navigator.session` API](./beaker-identities.md#navigatorsession-api) and are not allowed to create subfolders. The `index.json` must include a `"schema"` field. See [Object-store Folders](./object-store-folder.md) for more information.|
|`keystore`|The folder contains public keys which are used for encryption. Applications must request access using cryptography APIs and are not allowed to write to the folder.|

### `schema`

Optional String. The URL of the schema used for the files contained in the folder.

## Examples

**`/documents/index.json`**

```js
{
  "title": "Documents",
  "description": "Documents, spreadsheets, presentations, etc"
}
```

**`/data/index.json`**

```js
{
  "title": "User objects",
  "type": "protected"
}
```

**`/data/fritter-posts/index.json`**

```js
{
  "title": "Fritter posts",
  "type": "objectstore",
  "schema": "dat://fritter.hashbase.io/post.schema.json"
}
```
