# Object-store Folder Spec

![Draft](https://img.shields.io/badge/Draft-In%20progress-yellow.svg) ![Not implemented](https://img.shields.io/badge/Status-Not%20implemented-red.svg)

Object-stores are managed folders for storing and publishing application data. They provide users with a clear permissioning system while helping applications enforce schemas.

## Example usage

```js
// Request read access to the private contacts and full access to the public fritter posts
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
var privdat = new DatArchive(privContacts.host)
console.log(await privdat.readdir(privContacts.pathname))

// Publish a new post
var pubPosts = navigator.session.getUrl('public:objects', 'dat://fritter.hashbase.io/schemas/post.json')
var pubdat = new DatArchive(pubPosts.host)
await pubdat.writeFile(`${pubPosts.pathname}/${Date.now()}.json`, JSON.stringify({
  type: 'text',
  text: 'Hello, world!'
}))
```

## Specification

Object-stores are folders which contain only `.json` files. They follow a specific structure and only allow JSON files which fit a declared schema. Each objects folder is provided with a human-readable description which is used during permission requests.

Object-stores are not isolated by origin. Apps may share each others' object-stores if they support the same schemas. This is to facilitate interoperation and avoid data siloes.

Applications should use the object-store as their primary storage. The apps interact with the object-store using standard `DatArchive` file APIs. They can use the "private object-store" for private data and the "public object-store" for publishing.

### Dat types

Objectstore folders are only recognized on dats with the type [`user-private-fs`](./dat-types.md#user-private-fs) and [`user-profile`](./dat-types.md#user-profile)

### Folder structure

The `data/` folder is created automatically as part of the construction of the [users' private and public dats](./beaker-user-fs.md). It is used to contain object-store folders. It is ["protected"](./index-json.md#type) so that only the browser or user can modify it (no apps).

The individual object-store are created as-needed during the [`navigator.session.request()`](./beaker-identities.md#navigatorsession-api) flow.

```
data/       - the data container
data/*/     - the objectstore folders
```

Object-store folders are given the type [`"objectstore"`](./index-json.md#type). They must have a [`schema`](./index-json.md#type) URL specified.

**`/data/index.json`**

```js
{
  "title": "User data",
  "type": "objectstore"
}
```

**`/data/fritter-posts/index.json`** (example)

```js
{
  "title": "Fritter posts",
  "type": "objects",
  "schema": "dat://fritter.hashbase.io/post.schema.json"
}
```

### Schema files

Objects in the object-store use a [JSON Schema file](https://json-schema.org/) to describe the objects. The schemas serve multiple purposes:

 - To identify objects to applications.
 - To automatically validate the objects.
 - To describe to the user what objects an app is accessing (i.e. in permission prompts).

The schemas are identified using their URL.

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

### Object-store creation

Object-store folders are created as-needed during the [`navigator.session.request()`](./beaker-identities.md#navigatorsessionrequestopts) flow. If any of the schemas specified are not yet in use, the browser generates a new objects-folder for the schema. Folder names are generated as "slugified" forms of the schemas' titles.

The `title` and `description` of the JSON Schema file are copied to the object folders when they are created. If no title is present, then a title will be generated from the schema URL. This has an important user-facing function: when asked for permission, the title and description are used to explain to the user what data is being requested.

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

Then the folder's `index.json` would look like this:

```js
{
  "title": "Fritter Posts",
  "description": "Microblog posts and status updates",
  "type": "objectstore",
  "schema": "dat://fritter.hashbase.io/schemas/post.json"
}
```

### Permissions and schema enforcement

The `data/` is ["protected"](./index-json.md#type) and can not be modified by applications. On the user's "private dat," the folder may not be read without permission via [`navigator.showFileDialog()`](./beaker-user-fs.md#navigatorshowfiledialog).

The [`"objectstore"`](./index-json.md#type) folders (at `data/*`) follow the same rules by default: they can not be modified by applications, and they can not be read on the private dat. However, by using [`navigator.session.request()`](./beaker-identities.md#navigatorsessionrequestopts) applications can request greater access to the folders.

The object-stores are identified using the following "resource IDs" in the `requestUserSession()` call:

  - `'private:objects'` The object-store on the user's private dat.
  - `'public:objects'` The object-store on the user's public dat.

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
