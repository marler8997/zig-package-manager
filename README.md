
# Zig Package Manager

I've created this repository to have a central location to explore and document design ideas for the Zig Package manager.  I feel this part of the Zig ecosystem will be critical to its success and that its very difficult to design a good package manager that fits everyone's needs without being overly complex.

# The Package Manager's Role and Scope

* download things

# Andrew's Current Direction

* Decentralized
    - no global package repository

# My Personal Design Requests

* Able to build without touching the network
    - Able to verify dependency contents are correct without touching the network
    - Able to check whether dependencies are available without touching the network
    - To faciliate the 2 requirements above, dependencies should be referenced using a hash of their contents
* Accomodate working in multiple repositories that have a dependency relationship.
    - Users should be able to redirect dependencies to local repositories by modifying something on the filesystem, probably a file with a fixed name that can be added to .gitignore like ".redirects" or something.  We probably also want a command line tool to manage these redirects (zig list-redirects, zig redirect NAME PATH, ...)

# Design Ideas

* The ability to specify the downloading of resources, and zig packages.

```zig
const assets = b.addUrlResource("http://example.com/assets.tar.xz", "f4ab967db0370c82da6f447154ebef3d63912defc0b6c0ecf2f0b5c62becb927");
const someclib = b.addUrlResource("http://example.com/someclib-1.0.tar.xz", "f4ab967db0370c82da6f447154ebef3d63912defc0b6c0ecf2f0b5c62becb927");
const someziglib = b.addUrlResource("http://example.com/someziglib-1.0.tar.xz", "f4ab967db0370c82da6f447154ebef3d63912defc0b6c0ecf2f0b5c62becb927");
b.addCLibrary(libbar.subpath("src"));
b.addZigLib(someziglib);
```

* One decision package managers have is whether to support is sharing dependency storage in a global location on the filesystem.  However, managing dependences in this way creates problems that otherwise don't exist if each project keeps its own copy of their dependencies.  These problems include managing dependency ownership/lifetimes to know when to clean them and synchronizing access to global metadata about the dependencies. Things get alot simpler if each project has their own copy of dependencies.  I expect it will be better for Zig to implement the latter, and leave dependency sharing to the developer to manage through redirects.

* A command to quickly move from a fixed dependency, to a "development" dependency
    - For example, say we have downloaded a project and built it along with its dependencies.  Now we want to do some development, but we need to make changes to one of the dependencies.  The package manager already knows how to retrieve the dependency, so we should be able to ask it to download a copy of it that we can start developing on and simultaneously configure the current project to use that new development copy instead of the fixed one declared in the project's configuration.

```
# setup the "foo" project for development
zigpkg develop foo
```

# If we want global dependencies (we may not)

* Use bare git repositories to share disk space?
    - when a dependency is a git repository, the package manager can clone a bare repository, then clone any number of "non-bare" repositories that reference the bare repository.  This enables multiple instances of a repository on different commits without duplicating the full git history in each one.

* If the package manager maintains a globally shared database of repositories, then we can store each repositories' history in a folder matching the dependencies name.  When there are multiple repositories that share the same dependency name, then their git history/contents will be stored in the same directory.  This is actually OK.  In fact, we could store all repositories in the same global git repository, however, I'm sure this would run into some type of limitation with the filesystem or git itself, so, I think storing each repository into a subdirectory identified by it's name is a happy medium.

* Garbage collection on dependency database
    - I think we'll want to use the last access time of each dependency to determine which dependnecies are old and can be discarded
    - We can run the git garbage collector on repositories that we still need but that may have some instances/commits that are no longer needed
    - should be able to query the size of the database and each dependency (by name)
    - should be able to manually invoke the garbage collector somehow?

