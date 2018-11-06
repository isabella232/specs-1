# Beaker User Filesystem Spec

**Last updated:** Nov 6, 2018

![Draft](https://img.shields.io/badge/Draft-In%20progress-yellow.svg) ![Not implemented](https://img.shields.io/badge/Status-Not%20implemented-red.svg)

Beaker uses the dat network to create a browsable network of Web content. This spec introduces a filesystem structure which manages the user's personal information and mediates access by applications.

## Specification

The user filesystem is comprised of two dats called the "Private" and the "Public" dats.

### Private dat

The private dat is the root of the user's filesystem. It is kept off the network and can only be read by apps with permission. It is created with the `type` value of `["user-private-fs"]` and will have the following folder structure created automatically:

```
data.objs/     - private objectstore
public/        - a mount of the public dat
```

The filestructure has the following properties:

 - The `data.objs/` folder is a managed [object-store folder](./object-store-folder.md).

### Public dat

The public dat represents a public identity and contains information that the user wants published (though some of that information may be encrypted). Unlike the private dat, the public dat can be read freely by apps, but they must ask permission to 1) know the URL, and 2) make changes.

The public dat is created with the `type` value of `["user-profile"]`. It will have the following folder structure created automatically:

```
data.objs/     - public objectstore
user.keys/     - public keystore
```

The filestructure has the following properties:

 - The `data.objs/` folder is a managed [object-store folder](./object-store-folder.md).
 - The `user.keys/` folder is a managed key-store folder.

### Permissions

By default, applications can't write to folders on the public dat, and can't read or write to folders on the private dat. Applications can gain access to those files and folders using the [`navigator.showFileDialog()`](#navigator-showFileDialog) and [`navigator.session.request()`](./beaker-identities.md#navigatorsession-api) APIs.

## APIs

### navigator.showFileDialog()

Method to select a file or folder for reading or writing.

```js
await navigator.showFileDialog({
  type: 'open' | 'save',
  title: string,
  buttonLabel: string,
  filters: {
    isOwner: boolean,
    file: [{name: string, extensions: [string...]}, ...]
  },
  properties: [
    'openFile',
    'openDirectory',
    'multiSelections'
  ]
})
```

Returns an array of objects representing the selected files & folders and the capabilities given by the user. Each object fits the following schema:

```js
{
  url: string, // the full URL of the selected file/folder
  origin: string, // the origin segment of the URL
  path: string, // the path segment of the URL
  isDirectory(): boolean,
  isFile(): boolean
}
```

The application will be given the following permissions on the selected target depending on the conditions:

 - `stat()` and `readFile()` when the target is a file and the action is 'open'
 - `stat()`, `readdir()`, and `readFile()` when the target is a folder and the action is 'open'
 - `writeFile()` when the target is a file and the action is 'save'
 - All operations when the target is a folder and the action is 'save'

### navigator.session

The `navigator.session` API creates access sessions with multiple resources in the users' FS. This is documented in the [Beaker Identities Spec (`NavigatorSession` API)](./beaker-identities.md#navigatorsession-api).

## Future features

The Beaker filesystem should be designed to accomodate new features. Here are some of the features to consider on the horizon.

### Multiwriter

"Multiwriter" is a feature which enables multiple devices to write to a dat folder. It is useful for two specific use-cases:

 1. Pairing multiple devices (e.g. mobile, desktop, and laptop)
 2. Collaboration between multiple users

It's hard to predict exactly how multiwriter will be integrated into the Beaker FS. It seems likely that the private and public dat will use multiwriter to pair the datasets across devices. Or, perhaps, the private dats will stay device-specific, while the public dats will pair with multiwriter. For collaboration, there will probably need to be special flows for creating "shared folders."

### Authenticated networking

Currently the Dat networking stack is unauthenticated, meaning that peers do not prove their identity. This will soon change as Dat is integrating the NOISE protocol. When this happens, it will be possible to limit connections to specific peers, effectively adding access-control. This is very coarse-grained (can only apply to the entire dat) and so will need to work into flows that specifically fit that model.

One possibility is selective-sync of the private dat for backup. Another possibility is the creation of selectively-shared folders, perhaps as part of the multiwriter collaborative toolset.

### Non-hyperdrive mounts

Dat's primary data structure, Hypercore, is a log structure which can be repurposed for many different cases. The primary use-case is Hyperdrive, the filesystem data structure which drives the Beaker FS and websites. Other feasible data structures might be a key-value store, a video/audio feed, or even a log of data. This is especially useful for implementing CRDTs.

In the future, it may be possible to mount non-hyperdrive dats onto a hyperdrive. This would make it possible to create items which are represented as "files" or "folders" but which have a custom internal data structure.

### "Local filesystem" integration

The "Beaker FS" is separate from the filesystem of the OS that runs Beaker (Windows, MacOS, etc). Conceptually it is similar to the division between the OS filesystem and a cloud drive such as Dropbox or GoogleDrive.

The purpose of the separate Beaker FS is to provide networking features that would not be possible in the host OS. The private and public dats can be synced between devices and to cloud services, making them portable and easy to back up.

However, integration into the host OS's local filesystem has its own advantages. Currently, the user must explicitly "upload" or "download" files into and out of the Beaker FS. It would be much easier if applications could interface directly with the local filesystem. This would expand the conceptual space from:

 - Private dat fs
 - Public dat fs
 
To

 - Local fs
 - Private dat fs
 - Public dat fs

To accomplish this, a future spec could introduce a local filesystem API, and the access APIs such as `navigator.showFileDialog()` could be expanded to select from the local FS.
