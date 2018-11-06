# Object-store Folder Spec

**Last updated:** Nov 6, 2018

![Draft](https://img.shields.io/badge/Draft-In%20progress-yellow.svg) ![Not implemented](https://img.shields.io/badge/Status-Not%20implemented-red.svg)

Object-stores are managed folders for storing and publishing application data. They provide users with a clear permissioning system while helping applications enforce schemas.

Object-stores use the `.objs` folder extension. They are created at fixed paths according to the dat type (see [`user-private-fs`](./dat-types.md#user-private-fs) and [`user-profile`](./dat-types.md#user-profile)).

## Example usage

```js
// Request an access session
await navigator.session.request({
  resources: [{
    // Private contacts
    type: 'private:objects',
    schema: 'dat://walled.garden/contact.schema.json',
    permissions: ['read']
  }, {
    // Public fritter posts
    type: 'public:objects',
    schema: 'dat://fritter.hashbase.io/schemas/post.json',
    permissions: ['create', 'read', 'update', 'delete']
  }]
})

// Log the private contact filenames
var privContacts = navigator.session.getUrl('private:objects', 'dat://walled.garden/contact.schema.json')
var privdat = new DatArchive(privContacts.hostname)
console.log(await privdat.readdir(privContacts.pathname))

// Publish a new post
var pubPosts = navigator.session.getUrl('public:objects', 'dat://fritter.hashbase.io/schemas/post.json')
var pubdat = new DatArchive(pubPosts.hostname)
await pubdat.writeFile(`${pubPosts.pathname}/${Date.now()}.json`, JSON.stringify({
  type: 'text',
  text: 'Hello, world!'
}))
```

## Specification

Object-stores are managed folders which primarily contain `.json` files. They follow a fixed structure and enforce declared schemas. Each group of objects is provided with a human-readable description which is used during permission requests.

Object-stores are not isolated by origin. Apps may share each others' objects if they support the same schemas. This is to facilitate interoperation and avoid data siloes.

Applications should use the object-store as their primary storage. The apps interact with the object-store using standard `DatArchive` file APIs. They can use the "private object-store" for private data and the "public object-store" for publishing.

### Dat types

Object-store folders are only recognized on dats with the type [`user-private-fs`](./dat-types.md#user-private-fs) and [`user-profile`](./dat-types.md#user-profile). The location of the folder is enforced by those types (`/data.objs`).

### Folder structure

The `/data.objs` folder is created automatically as part of the construction of the [users' private and public dats](./beaker-user-fs.md). It is used to contain schema-specific folders which are themselves created as-needed during the [`navigator.session.request()`](./beaker-identities.md#navigatorsession-api) flow.

```
data.objs/             - the objectstore folder
data.objs/index.json   - a directory of the contained data
data.objs/*/           - the individual schema folders
```

### The index.json file

The `index.json` folder contains information about the contained folders. It provides metadata and maps to facilitate quick lookups.

**`/data.objs/index.json`**

```js
{
  "folders": {
    "fritter-posts": {
      "title": "Fritter posts",
      "description": "Microblog posts and status updates",
      "schema": "fritter.hashbase.io/post.schema.json"
    }
  },
  "schemas": {
    "fritter.hashbase.io/post.schema.json": "fritter-posts"
  }
}
```

The structure of `index.json` must follow these requirements:

 - `folders` required object. Maps the contained folders to some metadata.
   - `key` string. The folder name. Every folder inside the `.objs` should have their name in this object.
   - `value` object.
     - `title` optional string. The title of the folder's content. Copied from the schema's title.
     - `description` optional string. A description of the folder's content. Copied from the schema's description.
     - `schema` required string. The normalized URL of the schema.
 - `schemas` required object. Maps the contained folders' schemas to their folder names.
   - `key` string. The normalized schema URL. Every folder inside the `.objs` folder should have their schema in this object.
   - `value` string. The folder name.

A client which wants to lookup the folder path of objects according to their schema URL would follow this algorithm:

```js
async function lookupFolder (targetDat, schemaUrl) {
  schemaUrl = normalizeSchemaUrl(schemaUrl)
  var indexJson = await targetDat.readFile('/data.objs/index.json', 'json')
  return indexJson.schemas[schemaUrl]
}
```

### Schema files

Folders in the object-store use a [JSON Schema file](https://json-schema.org/) to describe their objects. The schemas serve multiple purposes:

 - To identify objects to applications.
 - To automatically validate the objects.
 - To describe to the user what objects an app is accessing (i.e. in permission prompts).

The schemas are identified using their URL (in a normalized form).

To help visualize how applications use schemas, refer to this example call of [`navigator.session.request()`](./beaker-identities.md#navigatorsessionrequestopts):

```js
// Request full access to the public fritter posts
await navigator.session.request({
  resources: [{
    type: 'public:objects',
    schema: 'dat://fritter.hashbase.io/schemas/post.json',
    permissions: ['create', 'read', 'update', 'delete']
  }]
})
```

In this call, the object-store is referenced using a resource ID (`'public:objects'`) and a schema ( `'dat://fritter.hashbase.io/schemas/post.json'`). The resulting session object provides the URL to the folder:

```js
// List the public fritter-post files
var postsUrl = navigator.session.getUrl('public:objects', 'dat://fritter.hashbase.io/schemas/post.json')
var pubdat = new DatArchive(postsUrl.host)
await pubdat.readdir(postsUrl.pathname)
```

It should be clear from this usage that schemas are used as identifiers for the objects. The browser uses the schema URLs to lookup the appropriate folder and to find metadata which describes the data.

### Schema URL normalization

Schema URLs are normalized to avoid splitting datasets due to trivial differences. The normalization algorithm should strip all content except the `hostname` and `pathname`. For instance, `dat://foo.com/bar.html?q=v#hi` should be normalized to `foo.com/bar.html`.

The normalization algorithm is easy to express using the current `URL` Web API:

```js
function normalizeSchemaUrl (url) {
  url = new URL(url)
  return url.hostname + url.pathname
}
```

### Object folder creation

Object-store subfolders are created as-needed during the [`navigator.session.request()`](./beaker-identities.md#navigatorsessionrequestopts) flow. If any of the schemas specified are not yet in use, the browser generates a new objects-folder for the schema. Folder names are generated as "slugified" forms of the schemas' titles.

The `title` and `description` of the JSON Schema file are copied to the object folders' metadata when they are created. If no title is present, then a title will be generated from the schema URL. This has an important user-facing function: when asked for permission, the title and description are used to explain to the user what data is being requested.

As an example, if a JSON Schema file looked like this:

```js
{
  "$id": "dat://fritter.hashbase.io/schemas/post.json",
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Fritter Posts",
  "description": "Microblog posts and status updates",
  // ...
}
```

Then the folder's relevant `index.json` entries would look like this:

```js
{
  "folders": {
    "fritter-posts": {
      "title": "Fritter Posts",
      "description": "Microblog posts and status updates",
      "schema": "fritter.hashbase.io/schemas/post.json"
    }
  },
  "schemas": {
    "fritter.hashbase.io/schemas/post.json": "fritter-posts"
  }
}
```

### Permissions and schema enforcement

By default, the `/data.objs` folder and its children can not be modified by applications. On the user's "private dat," it may not be read either.

To gain elevated access, applications can use [`navigator.showFileDialog()`](./beaker-user-fs.md#navigatorshowfiledialog) or [`navigator.session.request()`](./beaker-identities.md#navigatorsessionrequestopts). However, the permissions given by those two APIs (and all other APIs) can not violate the following rules:

 - The `/data.objs` folder can not be deleted.
 - The immediate children of `/data.objs` can not be written or deleted. (That is, files and folders in `/data.objs` can not be modified.)
 - Only `.json` files may be written inside the child folders of `/data.objs`. You may not create subfolders or non-JSON files.
 - The `.json` files written inside the child folders of `/data.objs` must pass schema validation.

The primary way to access the object-store folder is using [`navigator.session.request()`](./beaker-identities.md#navigatorsessionrequestopts). The object-stores are identified using the following "resource IDs" in the `requestUserSession()` call:

  - `'private:objects'` The `/data.objs` object-store on the user's private dat.
  - `'public:objects'` The `/data.objs` object-store on the user's public dat.

The application may request the following resource permissions:

 - `'read'` Can read the files in the objects folder.
 - `'create'` Can create new files in the objects folder.
 - `'update'` Can update existing files in the objects folder.
 - `'delete'` Can delete existing files in the objects folder.

Only `.json` files may be written to the objects folders. Files are validated against the JSON Schema on write; if validation fails, then the write will reject with the validation error message.

### Additional considerations

#### Schema version migration

There is currently no mechanism for applications to migrate across breaking changes. Object-store files are validated on write instead of read, and so a breaking change to a schema will not automatically invalidate prior objects. However, there are two concerns to consider:

 1. Breaking changes to a schema will cause any dependent applications to break until they've been updated.
 2. Breaking changes force applications to handle legacy schema versions.

It's recommended that breaking changes are published as separate schema-files at new URLs to avoid these issues.

#### Schema load failures

If a schema fails to load, then the application's [`navigator.session.request()`](./beaker-identities.md#navigatorsessionrequestopts) flow will fail with a load error. It's important for application developers to ensure that schemas stay available.
