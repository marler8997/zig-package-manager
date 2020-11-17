
# Filetrees

## Goal

The end goal is to create a good package management system for Zig.  This document describes a set of features that all package managers need, but that I've identified can be implemented independently of Zig's package manager.  Its postulated that decoupling these features from the package manager will reduce complexity and enable more functionality for the end user.

## Terminology "filetree"

This document uses to term "filetree" to mean a tree of files/directories.  For example, every directory is also a "filetree".  Also, `.tar` and `.zip` files can be considered filetrees.

## Proposal

It's proposed that the Zig build system keep "filetree acquisition" and "package management" separate.  Packages should work independently of how they are retreived.  A single repository should be able to house multiple packages that reference each other without having to go through a URL.  This document describes a design for filetree acquisition that can be used to declare filetree dependencies for a project that can also be leveraged by the package management system later on.

This proposed filetree feature requires a way for projects to tell the Zig build system:

1. How to retrieve a filetree (i.e. a URL)
2. How to verify a filetree is correct (i.e. a content hash)

In order to support modifying filetrees alongside each other, this also requires a way to:

1. Set/Get the local path to any filetree required by a project

### Example:

```zig
const assets = b.addArchive(.{
     .url = "http://example.com/assets-1.0.tar.xz",
     .hash = "9f1c9710f7126d5a55b990fa70a676ea1bb08d3c44acf6df264ba6f4942cb6e1",
});
const libbar = b.addGitRepo(.{
    .url = "https://github.com.com/someorg/bar",
    .hash = "cabfb482790204e41a779f25156048cab4977098",
});
```

When these filetrees are requested to be "built", the Zig build system will take care of downloading them to a local directory (probably in `zig-cache`) and verifying their contents.  Note that because we use a hash to identify the filetree, the build system can tell whether a filetree is available without touching the network.  When a user wants to make changes to these filetrees, they can redirect them to a local directory using the `.zig-redirect` directory:

```
# a file to redirect urls to local pathnames
.zig-redirect/url

# the subdirectory "byname" contains one file per redirect whose basename matches the name of the redirect (if it has one)
.zig-redirect/byname/foo
```

Here is an example of what the `url` file could look like for the example above:
```
http://example.com/assets-1.0.tar.xz ../assets
```

Note that there can be multiple `.zig-redirect` directories for Zig to scan.  They can exist in the project directory (alongside `build.zig`) or global redirects can exist in the user's HOME directory.  The `url` example above tells the build system that instead of downloading "assets" from the given URL, it should direct tools to the local path "../assets" (relative to the project directory).

It's important that the build system does not verify the contents of filetrees on each build, it only verifies contents of newly downloaded filetrees and saves the hash somewhere (probably in the pathname to the filetree).   Why is this?  First, if a user has redirected a filetree, Zig trusts the user to verify/maintain this filetree to be whatever they want even if it doesn't match the configured hash in the project.  The enables workflows where developers can work with multiple repositories/filetrees without breaking their build because the contents have changed. Second, even if a user has not redirected a filetree, the main purpose of the hash is to determine whether a filetree has already been retrieved and was retrieved in the same state as it was for the user who configured the hash.  However, there's no reason for the build to enforce that the contents never change, rather, it's purpose is only to detect if the configured hash has been modified so it can identify that it needs to download a new instance of the filetree.  That being said, user's should never have to modify a downloaded filetree, instead, there should be commands to instantiate a filetree in a "developer/draft mode", where it downloads or copies the filetree to a convenient location and sets up a redirect for it.

> NOTE: Once Zig has downloaded a filetree, it must have some way of storing/retriving the original hash it was verified with regardless of whether the content has changed (this could be as simple as storing the filetree in a directory with the hash in the name).

> NOTE: a use case for a global redirect could be to save disk space by sharing a filetree between multiple zig projects

This design requires that local paths to "filetrees" are not fixed.  The local directory path to any filetree must be queried.  Symlinks would work well for this on linux, but won't work on windows, so maybe we'll just use files with the pathname in them.  I think when the zig build system is invoked, it will update the `zig-cache` directory with entries for these filesets with the paths to them.

> IDEA: to enforce that tools always query the filetree path, when we download filetrees, we can store them in a subdirectory with a randomized sequence of characters.  This feature could also be a command-line option that only CI's use for testing.

To access a filetree outside the Zig build system, there needs to be a way to reference the file tree. This could be done via URL, hash, or the project can assign the filetree a name.  This could be done with a new compiler command, let's call "filetree".

```sh
$ zig filetree url http://example.com/assets-1.0.tar.xz
/path/to/my/project/zig-cache/filetree/abajej10f_this_is_a_randomized_string_fjalsjdf

$ zig redirect url http://example.com/assets-1.0.tar.xz ../assets
# above command puts this entry into .zig-redirect/url
# http://example.com/assets-1.0.tar.xz ../assets

$ zig filetree url http://example.com/assets-1.0.tar.xz
/path/to/my/assets
```

There's also use cases to redirect hashes:

`.zig-redirect/byhash/9f1c9710f7126d5a55b990fa70a676ea1bb08d3c44acf6df264ba6f4942cb6e1`:
```
/home/me/assets-1.0
```

This tells the Zig build system that if something needs the filetree with the given hash, no matter what it is, instead of downloading it, it can just point projects to this path.  A user can put redirects their HOME directory, and download/keep around assets that are commonly used between projects to avoid having to redownload them each time.

This would allow someone to do this:
```
$ zig filetree hash 9f1c9710f7126d5a55b990fa70a676ea1bb08d3c44acf6df264ba6f4942cb6e1
/home/me/assets-1.0
```

So an archive can have the following things:

1. A URL

This tells Zig how to download the filetree.  This could be optional.  For example, one could omit the URL and instead only provide a content hash.  The Zig build system could then resolve this filetree by looking at the redirects for the hash or the name (if it has one).

2. A content hash

Zig will use this to identify the filetree and verify that newly downloaded filetrees are correct. I can't think of a use case yet where this would be optional.  It could be set to something like "placeholder" or "todo" to mark that the dependency is being worked on so we haven't setup a hash yet, but the value should be something that obviously needs to be filled in before the project is setup properly.

3. A name

This is an optional configuration.  It provides a name that can be used outside of Zig to identify the filetree, to query its path.  Note that even if a filetree has no URL, and no name, then tools can still query the asset with its content hash, this seems like a valid use case.

A name is useful to assign to a filetree if other tools want to access it.  If a filetree has a name, a tool can easily find it's local directory pathname.  Say we have a filetree named "my_filetree", then zig can store the path in a file like this:

```
zig-cache/filetree/my_filetree
```

This would be easy to implement with symlinks, but that would be problematic for windows, so we might want to just make it a file with the target path as it's contents.


# Examples

### Example: download assets for a game

```
const assets = b.addArchive(.{
     .name = "assets",
     .url = "http://example.com/assets-1.0.tar.xz",
     .hash = "2486003ddaba2bc6a58e0ea5ea057b80265fd3f48dcb24fcdce94377673b273a",
});

//...

mygame.dependOn(&assets);
```

```sh
# zig build will download the assets
$ zig build
...

$ zig filetree name assets
/this_project_path/zig-cache/tree/2486003ddaba2bc6a58e0ea5ea057b80265fd3f48dcb24fcdce94377673b273a

# list the asset files
$ ls $(zig filetree name assets)
```
> NOTE: if running `zig filetree <identifier>` is too complex for a client to get the path to a filetree, then zig could store the paths in files as well, but this would mean having more than one way to get the path, and this way would be worse in that the paths would become stale if build.zig or .zig-redirects are updated without running zig build to update the filetree paths.  By using a command to get the path, the command can always return the most up-to-date path, and provide good error messages if needed.

Now let's say we want to make changes to our assets.
```sh
$ mkdir /tmp/assets-draft
$ cp -r $(zig filetree name assets) /tmp/assets-draft
$ zig filetree redirect name assets /tmp/assets-draft

$ zig filetree name assets
/tmp/assets-draft
```

### Example: zig library dependency

```
const zig_sdl = b.addGitRepo(.{
    .name = "zig-sdl",
    .url = "https://github.com/tiehuis/zig-sdl2",
    .hash = "43e0f5fa4a2b7f8a156c13cbd8540ef18373561e",
});

...

mygame.addPackagePath("SDL", zig_sdl.getPath());
mygame.dependOn(&zig_sdl);
```


# Shared Filetree Dependencies Between Projects

Consider the case when a project B, that has filetree dependencies, is itself an filetree dependency of project A.  So project A depends on project B.  Let's say that project C is a dependency of both A and B.  How does the Zig build system know that A's dependency of C and B's dependency of C is the same?  The hash will tell us that.  But what if the this project C is a library where project A depends on one version and project B depends on another?  What should be done in this case?  Are there different ways to resove this?

Case 1: project C is a zig library

Say we have three project A, B and C.  A depends on B and C, also, B depends on C.  So C is a shared filetree dependency.  Let's say that A and B depend on different versions of C, A depends on C1 and B depends on C2.  Where this will cause a problem in compilation is when the C package is imported with `@import("C")`.  The build will break down to these operations which is what causes the problem:

> Note: I've added a theoretical getZigBuildObject function that takes a path to a build.zig file and an object name

```
// in B/build.zig
B.addPackagePath("C", "path_to_C2");

// in A/build.zig
A.addLibrary(getZigBuildObject("path_to_B/build.zig", "B"));
A.addPackagePath("C", "path_to_C1");
```

Zig will be able to build B, however, when it goes to build A, it will have 2 different path values for package "C", which would produce an error like this:

```
Error: package "C" has 2 different paths
  A/build.zig: path_to_C1
  B/build.zig: path_to_C2
```
