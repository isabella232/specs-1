# Dat Types Spec

![Draft](https://img.shields.io/badge/Draft-In%20progress-yellow.svg) ![Not implemented](https://img.shields.io/badge/Status-Not%20implemented-red.svg)

All dats have one or more "type" values which are specified in the [`dat.json`](https://github.com/datprotocol/dat.json) file. This spec defines the core types which Beaker understands and explains how they are interpretted.

## Motivation

Dat types help clients interpret a dat. These standard types let Beaker know when to expect certain files and folders.

## Overview

Beaker defines the following core dat types:

|Type|Description|
|-|-|
|`user-private-fs`|Contains a user's private filesystem.|
|`user-profile`|Represents a user identity. Contains the user's public data.|

## Types

### `user-private-fs`

The dat is a user's private filesystem. In addition to standard dat files (dat.json, favicon.ico) it contains the following standard files:

 - `data/` Contains a collection of [object-store folders](https://github.com/beakerbrowser/object-store-folder-spec).
 - `public/` A mounted `user-profile` dat.

See the [Beaker User Filesystem spec](https://github.com/beakerbrowser/beaker-fs-spec) for more information.

### `user-profile`

The dat is a public user identity. In addition to standard dat files (dat.json, favicon.ico) it contains the following standard files:

 - `thumbnail.(png|jpg|jpeg|gif)` A square image that represents the user (ideally 256x256).
 - `data/` Contains a collection of [object-store folders](https://github.com/beakerbrowser/object-store-folder-spec).

See the [Beaker User Filesystem spec](https://github.com/beakerbrowser/beaker-fs-spec) and the [Beaker Identities spec](https://github.com/beakerbrowser/beaker-identities-spec) for more information.

