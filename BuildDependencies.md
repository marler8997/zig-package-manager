
# Build Dependencies

In order to have `build.zig` files that can be both self-contained and used by other builds, there needs to be a way to share build objects between `build.zig` files.  The nice thing about `build.zig` is that even though it is written in an imperitive language like Zig, it generates a declarative structure that makes it easy to process.  Here's an example of how that could look:

`/home/game/zlib/build.zig`
```zig
const Builder = @import("std").build.Builder;

pub fn build(b: *Builder) void {
    const mode = b.standardReleaseOptions();

    const zlib = b.addLibrary("zlib", "src/zlib.zig");
    zlib.setBuildMode(mode);
}
```

`/home/game/engine/build.zig`
```zig
const Builder = @import("std").build.Builder;

pub fn build(b: *Builder) void {
    const zlib_builder = b.getBuilder("../zlib");

    const mode = b.standardReleaseOptions();

    const engine = b.addLibrary("engine", "engine.zig");
    engine.setBuildMode(mode);
    engine.addPackage(zlib_builder.getLibrary("zlib").pkg);
}
```

`/home/game/build.zig`
```zig
const Builder = @import("std").build.Builder;

pub fn build(b: *Builder) void {
    const engine_builder = b.getBuilder("./engine");

    const mode = b.standardReleaseOptions();
    const target = b.standardTargetOptions(.{});

    const game = b.addExecutable("game", "src/game.zig");
    game.setTarget(target)
    game.setBuildMode(mode);
    game.addPackage(engine_builder.getLibrary("engine").pkg);
}
```

One thing to note is that I'm using a non-existent function `addLibrary`.  Currently we only have `addStaticLibrary` and `addSharedLibrary`, however, how the library is built should be an option left for the user rather than the library itself.  So we'll probably want to tweek some of the build API to separate library responsibility from the responsibilitiese of the user of the library.

# Implementation

So how do we share objects between `build.zig` files?  Also, should Zig support sharing objects with tools outside of Zig?  For `build.zig` files two mechanisms come to mind.  The first is that before compiling `build.zig`, the compiler could have some way of finding multiple `build.zig` files and compile them all together into one binary that allows builds to reference each other.  This should work, however, if we want other tools to be able to access this declarative data, there might be a better mechanism.  Another solution is to compile `build.zig` files that serialize the `builder` object to JSON. This has another advantage that each `build.zig` build would only need to be compiled once, and if one build.zig changes, you only need to recompile that one build file.
