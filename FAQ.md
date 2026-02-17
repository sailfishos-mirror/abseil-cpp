# Abseil FAQ

## Is Abseil the right home for my utility library?

Most often the answer to the question is "no." As both the
[About Abseil](https://abseil.io/about/) page and our
[contributing guidelines](https://github.com/abseil/abseil-cpp/blob/master/CONTRIBUTING.md#contribution-guidelines)
explain, Abseil contains a variety of core C++ library code that is widely used
at [Google](https://www.google.com/). As such, Abseil's primary purpose is to be
used as a dependency by Google's open source C++ projects. While we do hope that
Abseil is also useful to the C++ community at large, this added constraint also
means that we are unlikely to accept a contribution of utility code that isn't
already widely used by Google.

## How do I set the C++ dialect used to build Abseil?

The short answer is that whatever mechanism you choose, you need to make sure
that you set this option consistently at the global level for your entire
project. If, for example, you want to set the C++ dialect to C++17, with
[Bazel](https://bazel.build/) as the build system and `gcc` or `clang` as the
compiler, there are several ways to do this: * Pass `--cxxopt=-std=c++17` on the
command line (for example, `bazel build --cxxopt=-std=c++17 ...`) * Set the
environment variable `BAZEL_CXXOPTS` (for example, `BAZEL_CXXOPTS=-std=c++17`) *
Add `build --cxxopt=-std=c++17` to your
[`.bazelrc` file](https://docs.bazel.build/versions/master/guide.html#bazelrc)

If you are using CMake as the build system, you'll need to add a line like
`set(CMAKE_CXX_STANDARD 17)` to your top level `CMakeLists.txt` file. If you are
developing a library designed to be used by other clients, you should instead
leave `CMAKE_CXX_STANDARD` unset and configure the minimum C++ standard required
by each of your library targets via `target_compile_features`. See the
[CMake build instructions](https://github.com/abseil/abseil-cpp/blob/master/CMake/README.md)
for more information.

For a longer answer to this question and to understand why some other approaches
don't work, see the answer to ["What is ABI and why don't you recommend using a
pre-compiled version of
Abseil?"](#what-is-abi-and-why-dont-you-recommend-using-a-pre-compiled-version-of-abseil)

## What is ABI and why don't you recommend using a pre-compiled version of Abseil?

For the purposes of this discussion,
[ABI](https://en.wikipedia.org/wiki/Application_binary_interface) refers to the
compiled representation of code interfaces. This contrasts with
[API](https://en.wikipedia.org/wiki/Application_programming_interface), which
refers to the interfaces defined in the source code. [Abseil has a strong
promise of API compatibility, but does not make any promise of ABI
compatibility](https://abseil.io/about/compatibility). Let's take a look at what
this means in practice.

You might be tempted to do something like this in a
[Bazel](https://bazel.build/) `BUILD` file:

```
# DON'T DO THIS!!!
cc_library(
    name = "my_library",
    srcs = ["my_library.cc"],
    copts = ["-std=c++17"],  # May create a mixed-mode compile!
    deps = ["@com_google_absl//absl/strings"],
)
```

Applying `-std=c++17` to an individual target in your `BUILD` file is going to
compile that specific target in C++17 mode, but it isn't going to ensure the
Abseil library is built in C++17 mode, since the Abseil library itself is a
different build target. If your code includes an Abseil header, then your
program may contain conflicting definitions of the same
class/function/variable/enum, etc. As a rule, all compile options that affect
the ABI of a program need to be applied to the entire build on a global basis.

C++ has something called the
[One Definition Rule](https://en.wikipedia.org/wiki/One_Definition_Rule) (ODR).
C++ doesn't allow multiple definitions of the same class/function/variable/enum,
etc. ODR violations sometimes result in linker errors, but linkers do not always
catch violations. Uncaught ODR violations can result in strange runtime
behaviors or crashes that can be hard to debug.

If you build the Abseil library and your code using different compile options
that affect ABI, there is a good chance you will run afoul of the One Definition
Rule. Examples of GCC compile options that affect ABI include (but aren't
limited to) language dialect (e.g. `-std=`), optimization level (e.g. `-O2`),
code generation flags (e.g. `-fexceptions`), and preprocessor defines (e.g.
`-DNDEBUG`).

If you use a pre-compiled version of Abseil, (for example, from your Linux
distribution package manager or from something like
[vcpkg](https://github.com/microsoft/vcpkg)) you have to be very careful to
ensure ABI compatibility across the components of your program. The only way you
can be sure your program is going to be correct regarding ABI is to ensure
you've used the exact same compile options as were used to build the
pre-compiled library. This does not mean that Abseil cannot work as part of a
Linux distribution since a knowledgeable binary packager will have ensured that
all packages have been built with consistent compile options. This is one of the
reasons we warn against - though do not outright reject - using Abseil as a
pre-compiled library.

Another possible way that you might run afoul of ABI issues is if you
accidentally include two versions of Abseil in your program. Multiple versions
of Abseil can end up within the same binary if your program uses the Abseil
library and another library also transitively depends on Abseil (resulting in
what is sometimes called the diamond dependency problem). In cases such as this
you must structure your build so that all libraries use the same version of
Abseil.
[Abseil's strong promise of API compatibility between releases](https://abseil.io/about/compatibility)
means the latest "HEAD" release of Abseil is almost certainly the right choice
if you are doing as we recommend and building all of your code from source.

For these reasons we recommend you avoid pre-compiled code and build the Abseil
library yourself in a consistent manner with the rest of your code.

## What is "live at head" and how do I do it?

From Abseil's point-of-view, "live at head" means that every Abseil source
release (which happens on an almost daily basis) is either API compatible with
the previous release, or comes with an automated tool that you can run over code
to make it compatible. In practice, the need to use an automated tool is
extremely rare. This means that upgrading from one source release to another
should be a routine practice that can and should be performed often.

We recommend you update to the
[latest commit in the `master` branch of Abseil](https://github.com/abseil/abseil-cpp/commits/master)
as often as possible. Not only will you pick up bug fixes more quickly, but if
you have good automated testing, you will catch and be able to fix any
[Hyrum's Law](https://www.hyrumslaw.com/) dependency problems on an incremental
basis instead of being overwhelmed by them and having difficulty isolating them
if you wait longer between updates.

If you are using the [Bazel](https://bazel.build/) build system with
[Bzlmod](https://bazel.build/external/overview#bzlmod), you can use a
`git_override` in your `MODULE.bazel` file to track the latest commit.

For example, to update to the latest commit, you would add (or update) the
following snippet in your `MODULE.bazel` file:

```starlark
bazel_dep(name = "abseil-cpp", version = "20260107.1")

git_override(
    module_name = "abseil-cpp",
    remote = "https://github.com/abseil/abseil-cpp.git",
    # Replace the following line with the latest commit.
    commit = "6ec9964c325db0610a376b3cb81de073ea6ada90",
)
```

You can commit the updated `MODULE.bazel` file to your source control every time
you update, and if you have good automated testing, you might even consider
automating this.
