# Dat archive types

In the Dat filesystem, an archive is a bundle of files and directories, and in the context of a Web browser, the concept of a Dat archive maps cleanly to the concept of a website. However, there are many other potential use cases for Dat archives on the Web. 

In this proposal, we introduce the concept of "archive types." Types are a way to specify the role and possibly contents of a given Dat archive. Because types carry special meaning to the browser and can change core behaviors, they may only be changed by software with elevated permissions (the browser).

## The `type` field

The archive type is indicated with a new `type` field to in `dat.json`, which can be either a string or array of strings which indicate the type:

```js
{
  title: "Bob",
  type: ["user"]
}
```

An archive can have multiple types. The available values should be determined by further proposals, but we will document some types in this proposal.

## Reading `type`

The `type` may be read by using [`archive.getInfo()`](https://beakerbrowser.com/docs/apis/dat.html#getinfo):

```js
console.log(await bob.getInfo())
/* => {
  title: 'Bob',
  type: ['user'],
  ...
}
```

## Modifying `type`

Types may only be modified by the browser or by applications with elevated permissions (if supported by the browser). This is to protect users from accidental or malicious modification of archive roles. For instance, if an application could remove the `'user'` type from an archive, it could seemingly remove the user's profiles from login prompts.

## Proposed types

### Type: 'user'

Archives with the `'user'` type (aka "user dats") are meant to represent users in applications. They often contain profile information and some amount of content published by the user. We can think of a user dat's URL as the user's ID.

User dats are represented specially by the browser. They are used in user-session UIs as login targets (see [0004-simple-user-sessions](./0004-simple-user-sessions.md)) and represented in builtin UIs as the user's identities.

### Type: 'application'

Archives with the `'application'` type (aka "application dats") are meant to represent applications.

Applications often contain HTML, CSS, and Javascript files. They may in future proposals be given special capabilities and limitations, such as:

 - The ability to be "installed" and given special privileges and behaviors, such as registering for handling "intents" or having buttons added to the browser UI
 - A restriction against modifications via the `DatArchive` API, to protect against malicious edits
