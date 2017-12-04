# Simple user sessions

This is a proposal to provide new APIs and UIs for creating sessions with user identities built on Dat. The goal of this proposal is to remain as minimal and simple as possible.

In the existing Beaker Browser toolset, it is possible to prompt the user to select Dat archives using `DatArchive.selectArchive`. That API could be adapted to act as a kind of login experience (eg "Select the dat you want to login with"). However, we wish to provide users with a clear set of actions and interfaces around user identities. In particular, we think it is important that we:

 - Represent "user dats" as identities
 - Protect access to user dats
 - Indicate which user dat the application is actively using
 - Give tools to change the active user dat
 - Provide a nice developer experience

Therefore, in this document we will propose a concise set of concepts, APIs, and UIs for user dats and user dat sessions.

## Concepts

### User dats

User dats are Dat archives with the `'user'` type (see [0005-dat-archive-types](./0005-dat-archive-types.md)).

User dats contain all of the content and data which should be attached directly to the user identity. The browser may contain many user dats, and users can freely switch between them within an application (therefore changing the active session).

When viewed from the browser's builtin interfaces, a user dat is represented as a user identity. This may include special icons, labels, and categorizations.

### User dat sessions

User dat sessions are created by the user completing the signin flow. Each session has an ID, which is the URL of the user dat. An application can request a session using `navigator.session.request()`.

#### Lifetime

The session has no expiration time, but may be ended by the user through the browser UI. An application can end its session using `navigator.session.end()`.

#### Permissions

Applications with an active session may read and write any non-protected file on the user dat. After the session ends, all write permission is revoked.

As with any other Dat archive, an application may request permission to write to a user dat without a user session. The browser should give the user a special warning and confirmation during the permission prompt (eg "This dat is a user profile. Are you sure you want to give write access?").

## APIs

To access the signin and session flow for user dats, we propose the `navigator.session` API.

```js
navigator.session.id // => the url of the session's dat, or false if no session exists
navigator.session.isActive // => bool
await navigator.session.request()
await navigator.session.end()
```

## User interfaces

### URL bar Indicator UI

The URL bar should display an Indicator UI for the active session. This might exist as a small icon and/or label. Clicking on the Indicator should open the Session Viewer.

### Signin UI

The Signin is a popover or modal. It is triggered by a click on the Indicator, or by the `navigator.session.request()` API. It includes:

 - A selectable list of available users
 - A control to create a new user

### Session Viewer UI

The Session Viewer is a popover. It is triggered by a click on the Indicator. It includes:

 - Information about the active session (User name, avatar)
 - Links to view the session's user profile in more detail
 - Controls to logout and change the active session

## Example code

Here is a simple example usage of the session API:

```js
var profile // user's profile data
setup()

async function setup () {
  if (navigator.session.isActive) {
    await getUserData()
  }
  updateUI()
}

async function getUserData () {
  // use the webdb session to get the user data
  var archive = new DatArchive(navigator.session.id)
  profile = JSON.parse(await archive.readFile('/profile.json'))
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
      ...
    </main>
  </main>`)
}

function renderLogin () {
  // the UI for 'Log in' / 'Change profile'
  return yo`<div class="login">
    <button onclick=${doLogin}>${navigator.session.isActive ? 'Change profile' : 'Log in'}</button>
  </div>`
}

function renderProfile () {
  if (!profile) {
    return ''
  }
  return yo`<div class="profile">
    <img src=${profile.avatar}>
    <span>${profile.name}</span>
  </div>`
}

async function doLogin () {
  try {
    await navigator.session.request()
    await getUserData()
  } catch (e) {
    // failed, probably because the user cancelled
  }
  updateUI()
}
```