# Solving the 'shared data' layer

---

**Status:** Dead

This proposal has been superceded by [0004-simple-user-sessions](./0004-simple-user-sessions.md) and [0005-dat-archive-types](./0005-dat-archive-types.md).

---

This proposal introduces a few different concepts:

 - Archive types
 - User dats
 - The `navigator.sessions` API
 - Browsing profiles

This proposal also mentions WebDB, which is discussed in [0003-webdb.md](./0003-webdb.md).

Our goal is to create an ecosystem that users control. That means we want apps in Beaker to be able to share data and user-identities, so that users can add new apps to solve their own problems and to add more value.

One important part of this will be separating data from the application code. Rotonde is awesome, and it made a really sensible choice given the current tooling, which is to have each user fork the app code and then write their data to the app dat. We call that the "self-mutating dat" pattern, because it saves data by mutating (writing to) its own dat. It's a pattern we'll always keep, but it does bundle the user data with a specific application, and that's not always ideal. So, we wanted a solution for storing user data separately from applications.

## The common data layer, Dat

At the most basic level, Dat acts as the common data layer. If you have an application that edits files, like a text or photo editor, you simply use the `DatArchive.selectArchive` API to ask the user "which dat do you want to edit?" (We might need to add a `DatArchive.selectFile` modal as well, to select individual files and folders, but we haven't done that for 0.8.)

So: the Dat filesystem (FS) is the bottom-most abstraction for data.

## Dat archives and archive types

Within the Dat FS, there is the concept of "archives." Archives are basically a bundle of files, and, when browsing, they map very cleanly to the concept of a site. So, as a shorthand, you could think of 'archive' as == to 'site.' (Also, for what it's worth, we often just call archives "dats.")

In 0.8, we want to start toying with the idea of "archive types." This would be a way to specify the role, and perhaps even the contents, of a given dat. The way we chose to implement this is by adding a new `type` field to the dat.json, which can have a string or array of strings specifying the type. Here's an example:

```js
{
  title: "Bob",
  type: ["user"]
}
```

As of 0.8, the only type we're defining is `"user"`. Of course, you can leave off the type, in which case no special behavior or role is given.

## User dats

User dats (sometimes "userdats") are meant to represent users in applications. They are the thing which an app like Rotonde would use to write posts and profiles and follows and so on. Internally they're still just bundles of files, but they serve this special role of representing a user. We can think of a userdat's URL as the user's ID.

Userdats are treated differently by Beaker; they're automatically indexed by WebDB, and they're sometimes shown differently in UIs. Importantly for applications, they have special flows and APIs for obtaining read & write access. These flows behave like signin sessions. So, again using Rotonde as an example, a new user would click "Sign in," and then either choose an existing userdat, or create a new one. The rotonde app would receive access to the userdat, and could then read-from/write-to it to provide behaviors.

## Userdat sessions

The experience we want to immitate with userdats is that of existing web services which use a big provider's Single SignOn solution. SoundCloud is a good example of this; you click "Sign in with google" and a window opens listing your google profiles and the permissions being requested. You select a profile, and then you're returned to the app with an active profile. This is a nice, clear flow -- with no passwords! -- and we figured we could steal it.

In our proposal, applications "sign into" user dats for access to their files. This gives the user a clear flow for granting permissions and for choosing which user-dat to use. (This is especially handy because it grants the opportunity to create a new userdat, for testing untrusted apps, or for testing your own apps). Signing in also specifies permissions for accessing the userdat's files. Upon completion, a "session" is created which codifies the granted perms and the active user. That session also provides access to the APIs that affect the userdat.

## The `navigator.sessions` API

To access the signin and session flow for userdats, we propose the `navigator.sessions` API.

```js
var session = await navigator.sessions.request(service, {permissions?:}) // request session with a new service
var session = await navigator.sessions.get(service, index?) // get the handle to an existing session
var sessions = await navigator.sessions.list({service?:}) // get the handle to all existing sessions
session // => ServiceSession
session.id // a string containing the unique id of the session
session.permissions // an object describing the granted permissions
await session.destroy() // end the session
```

This API is designed to be generic enough to work with user dats (now) and remote services (later). It can also support multiple active sessions -- thus `list()` and the `index` option on `get()`. The `ServiceSession` object that is returned can be passed to the `WebDB` constructor to get access to its APIs.

Here is a simple example usage of the services API to help you get familiar with it:

```js
// globals
var sess // the webdb session
var loginErr // error during login
var profile // user's profile data
var latestPost // user's most recent timeline post
setup()

async function setup () {
  // fetch a session object
  // this may already be setup from a previous run
  // or it may be a totally new session
  // if null, no session is active
  sess = await navigator.sessions.get('webdb')
  if (sess) await getUserData()
  updateUI()
}

async function getUserData () {
  // use the webdb session to get the user data
  var webdb = new WebDB(sess)
  profile = await webdb.profiles.getActive()
  latestPost = await webdb.timeline.listPosts({limit: 1})
}

function updateUI() {
  // main render function
  // we call this function any time the state changes
  yo.update(document.querySelector('main'), yo`<main>
    <header>
      ${renderProfile()}
      ${renderLogin()}
    </header>
    <main>
      <h1>My latest post:</h1>
      <pre>${JSON.stringify(latestPost)}</pre>
    </main>
  </main>`)
}

function renderLogin () {
  // the UI for 'Log in' / 'Change profile'
  // renders any login error that might exist
  // then renders a button to login or change profile
  // (notice that onLogin works for both usecases)
  return yo`<div class="login">
    ${loginErr ? yo`<span class="error">${loginErr.toString()}</span>` : ''}
    <button onclick=${onLogin}>${sess ? 'Change profile' : 'Log in'}</button>
  </div>`
}

function renderProfile () {
  // the ui for the user's profile
  // renders nothing if not signed in
  // renders the configured profile info if active
  if (!profile) {
    return ''
  }

  // render profile info
  return yo`<div class="profile">
    <img src=${profile.avatar}>
    <span>${profile.name}</span>
  </div>`
}

async function onLogin () {
  // login event handler
  // attempts a login
  // saves the error on failure
  // then renders
  try {
    loginErr = null
    sess = await navigator.sessions.request('webdb', {
      permissions: {
        profiles: ['read'],
        timeline: ['read']
      }
    })
    await getUserData()
  } catch (e) {
    loginErr = e
  }
  updateUI()
}
```

## Beaker's browsing profiles

Separate from the user dats will be an internal concept of a "browsing profile." A browsing profile is meant to represent the users' settings and browsing data, such as their history and bookmarks and active user dats. Therefore the browsing profile sits at a lower level than the user dats.

Beaker 0.8 has some "application features" such as the ability to create public bookmarks. Therefore, a user dat is automatically created for the browsing profile. That user dat is the "browsing user." It has no special features, but it must exist, and can't be deleted unless another browsing user is chosen.