# 0001 Projects and Workspaces

We decided to make a change to the staging area which will reduce a bit of complexity. We hope that the end result won't feel shockingly different, but like a nicer version of what you had.

## The staging area problem

Beaker 0.7 has a "staging area" for all dats. The staging area is a "checkout" of the latest files in a dat, copied into a folder on your drive so that you can easily make edits. We created it so that developers can easily work on their sites and apps.

While there is a lot that the staging area does right, we fell out of love with this solution for a few reasons:
 
 - We added the staging area to all dats, which meant that apps using the `DatArchive` API needed to be aware of it. App devs would sometimes forget to `commit()` their changes, and (to be honest) having to write the extra `commit()` line was lame. Since we're unsure whether apps need the staging area at all, this has a pretty bad suck-to-potential-value ratio.
 - Adding the concept of a staging-area split the dat ecosystem; Beaker was the only dat client to have it.
 - To support the staging area, Beaker automatically creates a folder in the `~/Sites` directory for each dat you create or save. This can be cool, but it gets cluttered pretty fast. You end up with a folder full of "Untitled" dats, and worse...
 - ...it creates complex questions around control of the staging area files. For instance, when you delete a dat in Beaker, should the staging folder get deleted too? We put in a dialog box for the user to choose, which seemed like a good idea, but then I promptly lost an important personal site by misreading the dialog and making the wrong choice. So, now the grudge is personal.

All that said, there are some good things about the staging area. Most importantly: it creates a folder on your device so you can edit your files directly, and has a nice "review and publish" flow, which is important for working on websites and apps. We wanted to keep those features, but simplify the rest.

## Replacing the staging area

Our solution has a few components. First, a summary of what we removed:

 - We removed the concept of the staging area from the `DatArchive` API. No more `commit()`, `revert()`, or `diff()`. (Those methods will remain as noops with deprecation warnings.)
 - We no longer automatically create a staging area folder inside `~/Sites`.
 - The `dat://` URLs now only show what is published to the dat, not what's in the staging area.

So, dats no longer have any staging area built in. We're back to square one.

### Projects

From there, we added a replacement concept called "Projects."

Why call it Projects? Well, the use case we're solving is specifically developers building sites and apps. That's the only time where we feel it's vital to have a direct connection to the OS filesystem, because you need your editor, build tooling, etc. to modify the files. In every other case, dats will contain user data, and the flow is better served by explicitly "exporting" and "importing" the files from/to the DAT filesystem

So with Projects, when you want to create a site or app, you can use Beaker's built-in menus to "Create a website" or "Create an app," and that will create a new Project for you. Projects are for dats that you're actively working on. In addition to creating a Project for a new website or app, you can point a new Project to an existing dat.

### The project "workspace"

When you create a project, you choose a folder on your drive to be your "workspace." The workspace is very much like the staging area, but totally detached from the `DatArchive` API. We created a Projects toolset which gives you controls for configuring your Project and for publishing or reverting changes, much like the Library did with staging areas in 0.7.*. (Bonus: it even provides line-by-line diffs now!)

### The "workspace url"

Finally, so that developers can see the status of their work, we added a `workspace://` URL scheme which serves the current version of your site/app that is in the workspace.

Currently the workspace URL just serves files directly from the disk. This was a simple solution but I think it'll need to change to using dat internally. I won't get into why now, but once you try out projects/workspaces, I'm certain you'll eventually run into one of the cases that make you go "oh yeah workspaces need to be dats internally." Anyway, it's a TODO item.