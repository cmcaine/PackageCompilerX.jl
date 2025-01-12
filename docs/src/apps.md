# Apps

With an "app" we here mean a bundle of files where one of these files is an
executable and where the bundle can be sent to another machine while still allowing
the executable to run.

Use cases for Julia-apps is for example when one wants to provide some kind of
functionality where the fact that it was written in Julia is just an
implementation detail and forcing the user to download and use Julia to run the
code would be a distraction. There is also no need to provide the original
Julia source code for apps since everything gets baked into the sysimage.


## Relocatability

Since we want to send the app to other machines the app we create must be
"relocatable".  With an app being "relocatable" we mean it does not rely on
specifics of the machine where the app was created.  Relocatability is not an
absolute measure, most apps assume some properties of the machine they will run
on, like what operating system is installed and the presence of graphics
drivers if one want to show graphics. On the other hand, embedding things into
the app that is most likely unique to the machine, like absolute paths, means
that the application almost surely will not run properly on another machine.

For something to be relocatable, everything that it depends on must also be
relocatable.  In the case of an app, the app itself and all the Julia packages
it depends on must also relocatable. This is a bit of an issue because the
Julia package ecosystem has not thought much about relocatability since app
making has not been common in the Julia community. 

The main problem with relocatability of Julia packages is that many packages
are encoding fundamentally non-relocatable *into the source code*.  As an
example, many packages tend to use a `build.jl` file (which runs when the
package is first installed) that looks something like:

```
lib_path = find_library("libfoo")
write("deps.jl", "const LIBFOO_PATH = $(repr(lib_path))")
```

The main package file then contains

``` 
if !isfile("../build/deps.jl")
    error("run Pkg.build(\"Package\") to re-build Package")
end
include("../build/deps.jl")
function __init__()
    libfoo = Libdl.dlopen(LIBFOO_PATH)
end

```

The absolute path to `lib_path` that `find_library` found is thus effectively
included into the source code of the package. Arguably, the whole build system
in Julia is inherently non-relocatable because it runs when the package is
being installed which is a concept that doesn't make sense when distributing an
app.

Some packages do need to call into external libraries and use external binaries
so how are these packages supposed to do this in a relocatable way?  The answer
is to use the "artifact system" which was described in the following [blog
post][artifact-blog-url]. The artifact system is a declarative way of
downloading and using "external files" like binaries and libraries.  How this
is used in practice is described a bit later in this document.


## Creating an app

The source of an app is a package with a project and manifest file.
It should define a function with the signature

```jl
Base.@ccallable function julia_main()::Cint
  ...
end
```

which will be the entry point of the app (the function that runs when the
executable in the app is run). A skeleton of an app to start working from can
be found at
https://github.com/KristofferC/PackageCompilerX.jl/tree/master/examples/MyApp.

Regarding relocatability, PackageCompilerX provides a function
[`audit_app(app_dir::String)`](@ref) that tries to find common problems with
relocatability in the app. 

The app is then compiled using the [`create_app`](@ref) function that takes a
path to the source code of the app and the destination where the app should be
compiled to. This will bundle all required libraries for the app to run on
another machine where the same Julia that created the app could run.  As an
example, below the example app linked above is compiled and run:

```
~/PackageCompilerX.jl/examples
❯ julia -q --project

julia> using PackageCompilerX

julia> create_app("MyApp/", "MyAppCompiled")
[ Info: PackageCompilerX: creating base system image (incremental=false), this might take a while...
[ Info: PackageCompilerX: creating system image object file, this might take a while...

julia> exit()

~/PackageCompilerX.jl/examples
❯ MyAppCompiled/bin/MyApp
ARGS = ["foo", "bar"]
Base.PROGRAM_FILE = "MyAppCompiled/bin/MyApp"
...
ἔοικα γοῦν τούτου γε σμικρῷ τινι αὐτῷ τούτῳ σοφώτερος εἶναι, ὅτι ἃ μὴ οἶδα οὐδὲ οἴομαι εἰδέναι.
unsafe_string((Base.JLOptions()).image_file) = "/home/kc/PackageCompilerX.jl/examples/MyAppCompiled/bin/MyApp.so"
Example.domath(5) = 10
sin(0.0) = 0.0
```

The resulting executable is found in the `bin` folder in the compiled app
directory.  The compiled app directory `MyAppCompiled` could now be put into an
archive and sent to another machine or an installer could be wrapped around the
directory, perhaps providing a better user experience than just an archive of
files.

### Precompilation

In the same way as files for precompilation could be given when creating
sysimages, the same keyword arguments are used to add precompilation to apps.

### Incremental vs non-incremental sysimage

In the section about creating sysimages, there was a short discussion about
incremental vs non-incremental sysimages. In short, an incremental sysimage is
built on top of another sysimage while a non-incremental is created from
scratch. For sysimages, it made sense to use an incremental sysimage built on
top of Julia's default sysimage since we wanted the benefit of having a snappy
REPL that it provides.  For apps, this is no longer the case, the sysimage is
not meant to be used when working interactively, it only needs to be
specialized for the specific app.  Therefore, by default, `incremental=true` is
used for `create_app`. If, for some reason, one wants an incremental sysimage,
`incremental=true` could be passed to `create_app`.  With the example app, a
non-incremental sysimage is about 70MB smaller than the default sysimage.

### Filtering stdlibs

By default, all standard libraries are included in the sysimage.  It is
possible to only include those standard libraries that the project needs by
passing the keyword argument `filter_stdlibs=true` to `create_app`.  This
causes the sysimage to be smaller, and possibly load faster.  The reason this
is not the default is that it is possible to "accidentally" depend on a
standard library without it being reflected in the Project file.  For example,
it is possible to call `rand()` from a package without depending on Random,
even though that is where it is defined. If Random was excluded from the
sysimage that call would then error. Same applies to matrix multiplication,
`rand(3,3) * rand(3,3)` requires both `LinearAlgebra` and `Random` This is
because these standard libraries do "type piracy" so just loading them can
cause code to change behavior.

Nevertheless, the option is there to use, just make sure to properly test the
app with the resulting sysimage.


### Artifacts

The way to depend on external libraries or binaries when creating apps is by
using the [artifact system][artifact-pkg-url]. PackageCompilerX will bundle all
artifacts needed by the project, and set up things so that they can be found
during runtime on other machines.

The example app uses the artifact system to depend on a very simple toy binary
that prints some greek text. It is instructive to see how the [artifact
file](https://github.com/KristofferC/PackageCompilerX.jl/blob/d12d8d9b2286bb7d57ca28ad0f2a8cd130c70c81/examples/MyApp/Artifacts.toml)
is [used in the source
code](https://github.com/KristofferC/PackageCompilerX.jl/blob/d12d8d9b2286bb7d57ca28ad0f2a8cd130c70c81/examples/MyApp/src/MyApp.jl#L4-L7)

### What things are being leaked about the build machine and the source code

While the created app is relocatable and no source code is bundled with it,
there are still some things about the build machine that can be observed.

#### Absolute paths of build machine

Julia records the paths and line-numbers for methods when they are getting
compiled.  These get cached into the sysimage and can be found e.g. by dumping
all strings in the sysimage:

```
~/PackageCompilerX.jl/examples/MyAppCompiled/bin
❯ strings MyApp.so | grep MyApp
MyApp
/home/kc/PackageCompilerX.jl/examples/MyApp/
MyApp
/home/kc/PackageCompilerX.jl/examples/MyApp/src/MyApp.jl
/home/kc/PackageCompilerX.jl/examples/MyApp/src
MyApp.jl
/home/kc/PackageCompilerX.jl/examples/MyApp/src/MyApp.jl
```

This is a problem that Julia itself has:

```
julia> @which rand()
rand() in Random at /buildworker/worker/package_linux64/build/usr/share/julia/stdlib/v1.3/Random/src/Random.jl:256
```

#### Using reflection and finding lowered code

There is nothing preventing someone from starting Julia with the sysimage that
comes with the app.  And while the source code is not available one can read
the "lowered code" and use reflection to find things like the name of fields in
structs and global variables etc:

```
~/PackageCompilerX.jl/examples/MyAppCompiled/bin kc/docs_apps*
❯ julia -q -JMyApp.so
julia> MyApp = Base.loaded_modules[Base.PkgId(Base.UUID("f943f3d7-887a-4ed5-b0c0-a1d6899aa8f5"), "MyApp")]
MyApp

julia> names(MyApp; all=true)
10-element Array{Symbol,1}:
 Symbol("#eval")
 Symbol("#include")
 Symbol("#julia_main")
 Symbol("#real_main")
 :MyApp
 :eval
 :include
 :julia_main
 :real_main
 :socrates

julia> @code_lowered MyApp.real_main()
CodeInfo(
1 ─ %1  = MyApp.ARGS
│         value@_2 = %1
│   %3  = Base.repr(%1)
│         Base.println("ARGS = ", %3)
│         value@_2
│   %6  = Base.PROGRAM_FILE
│         value@_3 = %6
│   %8  = Base.repr(%6)
│         Base.println("Base.PROGRAM_FILE = ", %8)
│         value@_3
│   %11 = MyApp.DEPOT_PATH
```

[artifact-blog-url]: https://julialang.org/blog/2019/11/artifacts
[artifact-pkg-url]: https://julialang.github.io/Pkg.jl/v1/artifacts/

