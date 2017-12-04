# WebDB

---

**Status:** Dead

This proposal has been superceded by [0004-simple-user-sessions.md](./0004-simple-user-sessions.md) and [0005-dat-archive-types.md](./0005-dat-archive-types.md).

---

WebDB is a proposed toolset for reading and writing semantic objects on the Dat network. It provides the ability to run queries against the data in multiple files, and to generally think in terms of data records instead of files. It's designed to be much easier to use than the Dat files APIs, without actually getting away from the Dat core protocols. In fact, each individual WebDB record is a single JSON file in a Dat archive.

WebDB is built with [Ingest](https://github.com/beakerbrowser/ingestdb), which is a crawling indexer that runs on top of the Dat FS. [Watch this video](https://www.youtube.com/watch?v=EUPxAvxWlwQ) for a primer on Ingest (originally called "Injest"). Any application can run its own copy of Ingest on the page, but Beaker includes its own instance of Ingest, and that is WebDB.

WebDB is essentially a set of pre-programmed data types, such as "posts" and "user profiles." It includes type-specific methods for reading, querying, and mutating those data types. The long-term goal is to provide a suite of types which can cover many different applications. 0.8 will have a few starting types, and we will expand that set with each release of Beaker.

Here is a small set of example usages:

```js
var webdb = new WebDB(await navigator.sessions.request('webdb'))
await webdb.profiles.get(bob.url)                  // get bob's profile
await webdb.profiles.getActive()                   // get this session user's profile
await webdb.timeline.post({text: 'Hello, world!'}) // post to the timeline
await webdb.timeline.getThread(postUrl)            // get a post and its replies
await webdb.votes.set(postUrl, 1)                  // upvote a post
await webdb.votes.tabulate(postUrl)                // calculate the vote tally on a post
```

As you can see, these APIs are highly opinionated. We have a set of justifications for including WebDB, which are as follows:

 1. Handling permissions
 2. Ensuring compatibility between applications
 3. Improving app performance and reliability
 4. Reducing the number of indexes
 5. Providing easy-to-learn APIs

Let's briefly cover these justifications, and then discuss our proposal for how WebDB will work in more detail.

### 1. Handling permissions

Providing access-control to user data is an important part of creating a rich ecosystem. If users can't protect themselves against apps, then the ecosystem won't work.

As of 0.7, access-control is very coarse grained. Any application with the URL for a dat can read its files, and write access is given to the entire dat. That is much too coarse for most applications.

WebDB gives us a very clear set of data types and related actions which we can mediate. Rather than simply prompting the user that an app "Would like to be able to edit your profile," we can say more specifically, "Would like to be able to edit your profile and post to your timeline."

### 2. Ensuring compatibility between applications

Making sure applications can speak each others' data formats is an enormous challenge. The Semantic Web movement made this a priority for years, and created a number of very interesting solutions based around the "Resource Description Framework (RDF)."

We gave RDF a close look, including the specs for SPARQL and JSON-LD. We liked aspects of it, but decided against using the tech because there was not enough ready-made software for us to use, and we felt the learning curve of the tech was too high for our audience. Most of RDF is based on custom formats and the "graph" model for databases, which is not a frequenlty used DB model in the software industry. We also felt the ecosystem around RDF has failed to prove out the value of its design -- though we admit that could change.

Put *very* simply, RDF solves the problem of compatibility by assigning URLs to all schema definitions. This has the advantage of being very specific, but the disadvantage of being very verbose. After a few days of toying with that solution, we began to ask ourselves, are we sure we need developers to jump through these hoops? And, if we need this level of specificity, is it possible that some software will be able to handle it for developers later? Therefore we decided, if URL-based schema labels are the solution, we'd prefer to arrive at that decision after having tried and discarded all other options.

With WebDB, we delay answering this question by simply baking some opinions into the platform. Since our ecosystem is still small, we know there won't be many other schemas for us to handle, so we have little risk of collisions or conflicts. This gives us a functional solution now, and gives us time to examine the problem with real-world usage. (We'll work with the rotonde community to get yall onto WebDB -- unless you want to stay separate.)

In practice, WebDB identifies data by their paths in a dat. Timeline posts, for instance, are any file which match `/timeline/*.json`. Votes are files which match `/votes/*.json`. User profiles are `/profile.json`.

### 3. Improving app performance and reliability

The Dat FS is a relatively low-level API for building modern applications. Though devs may often fantasize about only needing the FS for their work, the reality is that for most non-trivial apps, you'll need to run queries across multiple data points. Accomplishing that with files will require loading each possible file into memory, and then scanning each file for relevant information. If the files get large, this could involve a lot of memory and long disk reads. And, the reality is, this process is exactly what a database is designed to accomplish -- and so WebDB simply solves that process once for everybody.

There are other subtle challenges which WebDB solves. Timestamps, for instance, can't be trusted in a global network, and WebDB can provide sensible solutions for handling order. WebDB also breaks records into multiple files which it can selectively replicate, which improves load performance noticeably compared to single large files.

### 4. Reducing the number of indexes

The toolset that powers WebDB, [Ingest](https://github.com/beakerbrowser/ingestdb), can be embedded into a Web application. This is great for giving full control to an application, but it means that the app must maintain its own computed indexes inside `IndexedDB`. By using WebDB, we can have applications share their indexes, reducing the overall data usage and load times.

### 5. Providing easy-to-learn APIs

Our goal with Beaker is to provide extremely accessible APIs. Our motto is, "Every user is a potential developer." That means we don't want users to have to learn complex concepts before making an app: we want them to sit down and be productive.

WebDB helps us accomplish that goal by giving very accessible (and fun!) APIs to work with. [Ingest](https://github.com/beakerbrowser/ingestdb) isn't terrible to learn, but it has a higher learning curve than we want for new users.

## WebDB in 0.8

We're still working on the final APIs and data schemas, so these specs may change. (Feedback is appreciated!)

WebDB is built on top of the "user dats" system. It is acquired using the `navigator.sessions` API, discussed in [0002-shared-data-layer.md](./0002-shared-data-layer.md). First, you acquire a session. Then, you instantiate a WebDB object, passing the session into the constructor.

```js
var sess = await navigator.sessions.request('webdb', {
  permissions: {
    profiles: ['edit-profile', 'edit-social'],
    timeline: ['create-post', 'edit-post'],
    comments: ['create-comment', 'edit-post'],
    votes: ['create-vote']
  }
})
```

Presently, because all WebDB is public, there are no 'read' permissions, and all data may be read without asking for additional permission.

WebDB indexes data which is either A) created by the user, or B) followed by the user. In a sense, that makes WebDB a "social database," but this indexing strategy will vary by the different data types and so will become more sophisticated in the future.

Here are the data types for WebDB which we're proposing for 0.8:

 - User profiles
 - Timeline posts
 - Comments
 - Votes

This is the basic toolset needed to make a simple Twitter clone. We'll expand it with new data models and types with subsequent 0.8.x releases.

### User profiles

User profiles contain four standard attributes. They are `name`, `description`, `avatar`, and `follows`. The last one, `follows`, is an array of followed users.

```js
{
  "name": "Bob Roberts",
  "description": "A simple webdb user.",
  "avatar": "/bob.jpg",
  "follows": [{"url": "dat://.../", "name": "Alice Allison"}]
}
```

The APIs for this core API are:

```js
await webdb.profiles.get(user)
await webdb.profiles.update(user, {name:, description:, avatar:})
await webdb.profiles.setImage(user, buffer, imageType)
```

They also include variants for quickly accessing the user's data:

```js
await webdb.profiles.getActive()
```

The API for followers is as follows:

```js
await webdb.profiles.follow(user)
await webdb.profiles.unfollow(user)
await webdb.profiles.listFollowers(user, {
  bidirectional: boolean, only include followers who also follow the target user
  mutual: boolean, only include followers who the current user follows
})
await webdb.profiles.countFollowers(user, {
  bidirectional: boolean, only include followers who also follow the target user
  mutual: boolean, only include followers who the current user follows
})
await webdb.profiles.isFollowing(userA, userB, {
  bidirectional: boolean, only return true if both follow each other
})
```

Permissions:

 - `'edit-profile'` - Can call `update()`
 - `'edit-social'` - Can call `follow()` and `unfollow()`

### Timeline posts

The timeline is a social feed for sharing posts and media. It is designed to work similarly to Twitter's feed, and includes support for threaded conversations. The 'post' object contains three standard attributes: `id`, `text`, and `createdAt`.

```js
{
  "id": "0ja8m8iqc",
  "text": "Hi Bob!",
  "createdAt": 1511208019667
}
```

The API for posting, viewing feeds, and getting threads is as follows:

```js
await webdb.timeline.post({
  text: string, the text of the post
})
await webdb.timeline.updatePost(post, {
  text: string, the text of the post
})
await webdb.timeline.removePost(post)
await webdb.timeline.listPosts({
  author: url, only show posts by the given author
  after: timestamp
  before: timestamp
  offset: number
  limit: number
  reverse: boolean
  fetchAuthor: boolean, fetch the .author object in each post
  fetchReplies: boolean, fetch the .replies array for each post
  countVotes: boolean, fetch and tabulate the .votes object for each post
})
await webdb.timeline.countPosts({
  author: url, only show posts by the given author
  rootPostsOnly: boolean, dont include replies
  after: timestamp
  before: timestamp
  offset: number
  limit: number
})
await webdb.timeline.getThread(url)
```

The `fetchReplies` option in `listPosts()`, as well as the `getThread()` method, will include a threaded listing of comments. Similarly, the `countVotes` option, as well as the `getThread()` method, will include a tabulation of votes on each post or comment. Similarly, the `fetchAuthor` option, as well as the `getThread()` method, will include the profile of the author of each post or comment.

Permissions:

 - `'create-post'` - Can call `post()`
 - `'edit-post'` - Can call `updatePost()` and `removePost()`

### Comments

Comments are a generic way to attach text to any given URL. Comments and their replies are threaded. Each comment contains contains five standard attributes: `id`, `subject`, `parentComment`, `text`, and `createdAt`.

```js
{
  "id": "0j9mz86f9",
  "subject": "dat://3f7edcb04ab30543d2a0d412b66b65dae50ae50f20baf9d553aa90b2fb94bb75/posts/0j84y24t1.json",
  "text": "Great post, Alice!",
  "createdAt": 1511208019667
}
```

The API for posting comments and getting threads is as follows:

```js
await webdb.comments.create({
  text: string, the text of the comment
  subject: string, the url which the comment is about
  parent: string, the url of the parent comment
})
await webdb.comments.edit(comment, {
  text: string, the text of the comment
})
await webdb.comments.remove(comment)
await webdb.comments.getThread(subject, {
  depth: number, an optional maximum depth to descend in the thread
  limit: number, an optional maximum number of comments to fetch
  fetchAuthor: boolean, fetch the .author object in each post
  countVotes: boolean, fetch and tabulate the .votes object for each post
})
await webdb.comments.getReplies(comment, {
  limit: number, an optional maximum number of comments to fetch
  fetchAuthor: boolean, fetch the .author object in each post
  countVotes: boolean, fetch and tabulate the .votes object for each post
})
```

Permissions:

 - `'create-comment'` - Can call `create()`
 - `'edit-post'` - Can call `edit()` and `remove()`

### Votes

Votes are a generic way to approve or disapprove of something. It enables you to vote 'up' or 'down' on any given URL. Each vote contains four standard attributes: `id`, `subject`, `vote`, and `createdAt`.

```js
{
  "id": "0j84x9oi4",
  "subject": "dat://3f7edcb04ab30543d2a0d412b66b65dae50ae50f20baf9d553aa90b2fb94bb75/posts/0j84y24t1.json",
  "vote": 1,
  "createdAt": 1506632670122
}
```

The API for creating, listing, and tabulating votes is as follows:


```js
await webdb.votes.set(subject, vote) // vote must be -1, 0, or 1
await webdb.votes.listFor(subject, {
  after: timestamp
  before: timestamp
  offset: number
  limit: number
  reverse: boolean
})
await webdb.votes.countFor(subject, {
  after: timestamp
  before: timestamp
  offset: number
  limit: number
})
await webdb.votes.listBy(author, {
  after: timestamp
  before: timestamp
  offset: number
  limit: number
  reverse: boolean
})
// this returns {up: number, down: number, value: number, upVoters: array of urls, downVoters: array of urls, currentUsersVote: number}
await webdb.votes.tabulate(subject, {
  listUpVoters: boolean, include an array of upvoter URLs
  listDownVoters: boolean, include an array of downvoter URLs
})
```

Permissions:

 - `'create-vote'` - Can call `set()`

### The temporary index

Any time data is fetched for a user that isn't followed, using `webdb.profiles.get()` or `webdb.timeline.listPosts({author: ...})` for instance, WebDB will use the temporary index. If the target information is not yet loaded in the temp index, then WebDB will delay the response until it's loaded.

### Timestamps

Timestamps on WebDB records can not always be trusted, but they're still useful. For instance, WebDB sorts the timeline by the `createdBy` timestamp, which means that somebody could give their post a far-future timestamp in order to always stay at the top of the feed. (This can happen accidentally!)

To mitigate this, WebDB will use countermeasures that depend on the data type. For instance, in the timeline, it's possible that a post that's marked far in the future will simply be rejected from the index. In all cases, if a timestamp is ever found to be in the future, it will be replaced with the current time.

## This is way too opinionated!

WebDB is very opinionated. It's much more opinionated than we intended it to be when we started working on 0.8.

It's possible that we're making a bad judgment call, or have a bad assumption about the importance of a requirement. That's why we're doing our best to record our requirements and justifications. We may misunderstand what's important for Beaker and people who use it!

That said, the reason we're ultimately comfortable with this design is that we believe we can gradually externalize our opinions. For instance, the APIs and permissions are a highly opinionated feature of WebDB. Presumably we'd like any application to be able to have APIs and a permissions prompt for that API. However, to provide that, we'd need a way for applications to export APIs to each other, and provide long-running processes, and to specify their available permissions, and so on. This is all doable, and a good direction for the future of Beaker, but it's not doable in the timeline we have for 0.8.

Likewise, we may be able to find a spec-format which configures Ingest, and therefore makes the builtin data types of WebDB unnecessary. We strongly considered that, but realized we don't understand all the requirements that a spec like that would require. It would also make the APIs harder to work with, in the name of getting flexibility which we might not need yet.

Therefore our solution is to let WebDB in 0.8 be highly opinionated. It includes decisions which we think will cover many use cases, and which we think we can make sufficiently general and/or sophisticated to cover a lot more. (In fact, we want feedback and suggestions for how we can develop the API.) Over time, as Beaker gains more a sophisticated process model, we will look for ways to move WebDB entirely into userland.

## The three layers of abstraction: WebDB, Ingest, and the Dat FS

If you're looking for something different than what WebDB gives you, you have two other options: you can use [Ingest](https://github.com/beakerbrowser/ingestdb) directly in your application, or you can use the Dat FS directly. This creates our 3 layers of abstraction: WebDB, Ingest, and Dat.

Ingest can be imported into both nodejs and Beaker web apps. In nodejs, it uses leveldb as its backing store; in Beaker, it uses indexeddb.

Using Ingest as a module requires more work as a developer, and loses some of the advantages of WebDB. Your index is no longer shared between applications, so it has to be populated at load time. There's no fine-grained way to define access-permissions to your data either; another app will just have to ask for read/write access to the Dat's files, and then construct its own Ingest instance with a similar data schema as yours. There is also more questions about inter-application compatibility.

However, if you use Ingest or the Dat FS directly, you will be able to customize your data schemas completely. Since WebDB cannot possibly cover all use cases, this may be a good reason to use Ingest or Dat.