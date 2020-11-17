# Zig Package Manager

### Feature 1: Solve the "Build Dependencies" Problem

Address how to share objects between `build.zig` files (and potentially other tools outside of Zig).

[BuildDependencies.md](BuildDependencies.md)

### Feature 2: Solve the problem of automatically acquiring files

How does a project access files from a remote resource while at the same time supporting making changes to those resources?

[Filetrees.md](Filetrees.md)


# Example

With features 1 and 2, we'll be able to write things like this:

`build.zig`:
```zig
const Builder = @import("std").build.Builder;

pub fn build(b: *Builder) void {
    const zig_sdl = b.addGitRepo(.{
        .name = "zig-sdl",
        .url = "https://github.com/andrewrk/zig-sdl.git",
        .hash = "13cb642ba827bb069c5b66a912dd26a072dfe9b0",
    });
    const assets = b.addArchive(.{
        .name = "assets",
        .url = "http://myserver.net/mygameassets-2020-11-20.0.tar.xz",
        .hash = "eb606c9ae73a751543c2ecd3aae647e75a49dc5814b5909ad0a1c8b387d46166",
    });

    const zig_sdl_builder = b.getBuilder(zig_sdl.relativePath("build.zig"));

    const game = b.addExecutable("game", "src/game.zig");
    game.setTarget(target)
    game.setBuildMode(mode);
    game.addPackage(zig_sdl_builder.getLibrary("zigsdl").pkg);
}

```
