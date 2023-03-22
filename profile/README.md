# CMake PAUL

Platform agnostic utility libraries using CMake.

## Why this organization?

At my day-job, I recently got confronted with some utility libraries written in `C/C++`.
The libraries were required to run on different platforms (to be precise: different targets in an embedded software development setting).
I.e. the libraries were expected to be compileable for different platforms without source code changes required.
The different platforms shared a set of common functionality, differing in subtle ways in how this functionality was provided.

There were some measures taken to hide the platforms' differences from the utility libraries.
Still, the code base deserved improvement to obtain modularity and easy extensibility.

Out of curiosity, I tried coming up with a solution implemented in CMake alone.
There definitely are other ways of achieving similar outcomes (e.g. I've seen `C++` templates being used for this).
This organization resembles a toy implementation of my take at the problem.

See [Setting and requirements](#setting-and-requirements) for a detailed description of the requirements.

Disclaimer: this organization does not resemble any organization in the true sense of the word.
As GitHub lacks the notion of grouped repositories, I misuse the concept of an organization to achieve the grouping anyway.
I.e. this organization is not linked in any way with the folks behind [CMake](https://cmake.org/) itself!

## Setting and requirements

- There are different platforms.
  Each platform provides the same shared low-level functionality, but in slightly different ways (e.g. differing signatures of 'call-backs').
- There are utility libraries.
  These utility libraries implement some high-level functionality on top of the shared functionality provided by the platforms.
- There are parent projects.
  These parent projects resemble different self-contained applications.
- The utility libraries are expected to compile for different platforms without any source code change required.
  I.e. the utility libraries may not rely on subtleties of one platform or the other.
- The utility libraries do not know which platform they are running on.
  I.e. it is up to the parent project to pick one of the platforms available.
- All the components are expected to be self-contained.
  I.e. apart from deciding which platform to use, there should not be any additional plumbing required from a parent project in order to use one of the utility libraries on top of one of the platforms.

## My take

In the following `parent_X`, `util_X` and `platform_X` represent a parent project, an utility library and a platform respectively.

- To represent the common functionality shared by all platforms, we introduce an `INTERFACE` library `common`.
  `common` provides a common set of declarations of functions, types, etc.
  `common` abstracts away the differences of one platform compared to another.
  `platform_X` then provides implementations of everything declared in `common`.
- To represent the dependency of `util_X` on some implementation of `common` given by some `platform_X`, we _could_ introduce a `target_link_libraries` dependency between `util_X` and `platform_X`.
  We would have to place the `target_link_libraries` in `platform_X`'s `CMakeLists.txt`, as only one platform is allowed at the same time (i.e. we cannot list all platforms in `util_X`'s `CMakeLists.txt`).
  This is far from ideal for at least two reasons:
  - Every time a new utility library `util_X'` is introduced, we would have to add a `target_link_libraries` dependency for each and every `platform_X` and `util_X'`.
    This defeats the goal of easy extensibility.
  - More importantly, if not all `util_X` libraries are used at the same time by all parent projects `parent_X`, we end up with references to non-existing CMake targets (those corresponding to unused `util_X`s).

  Instead, we introduce another interface library `common_umbrella` and use it as follows:
  - Each utility library `util_X` building on the interface defined in `common` adds `common_umbrella` as link dependency: `target_link_libraries(util_X PRIVATE common_umbrella)`.
    `util_X` does not reference `common` directly.
  - Each `platform_X` library providing an implementation of the interface defined in `common` registers itself with `common_umbrella` using an additional CMake-function `register_with_common`.
    `register_with_common` basically adds a link dependency `target_link_libraries(common_umbrella INTERFACE platform_X)` but can be extended to feature additional functionality (e.g. checking for exactly one platform library `platform_X` being present at the same time).
  - To help IDEs finding `common`s headers e.g. when working on some `util_X` (on its own, neither a parent nor a platform given), we can optionally add a link dependency between `common` and `common_umbrella`.
    Without this optional link dependency, the dependency is brought into play indirectly via the `platform_X` library at use.

  This way, we get rid of both issues mentioned above:
  - Introducing a new `util_X` library, we just add `common_umbrella` as link dependency and immediately have all the platforms available.
  - The `platform_X` are fully agnostic of the `util_X` libraries and no longer reference potentially non-existing targets.

  Note that with the layout described above, we achieve both a modular and an extensible layout:
  - All three types of targets (`parent_X`, `util_X` and `platform_X`) can be represented in a self-contained way.
  - When selecting a specific platform in some parent project `parent_X`, no additional plumbing is required (apart from including the platform itself).
  - Introducing a new target of one type does not require adapting any target of another type (it doesn't even require adapting targets of the same type).

## Example

The repositories living under this origanization resemble a basic example consisting of two parent projects [`parent_A`](https://github.com/cmake-paul/parent_a) and [`parent_B`](https://github.com/cmake-paul/parent_b), two utility libraries [`util_A`](https://github.com/cmake-paul/util_a) and [`util_B`](https://github.com/cmake-paul/util_b) alongside two platforms [`platform_A`](https://github.com/cmake-paul/platform_a) and [`platform_B`](https://github.com/cmake-paul/platform_b).
[`common`](https://github.com/cmake-paul/common) defines the common interface for all platforms as described above.
The high-level link dependency relationships are shown in the following diagram.

![Diagram](profile/img/diagram.svg)

To highlight modularity, here is the above diagram with an emphasis put on the link dependencies introduced in `parent_B`'s, `util_A`'s, `platform_A`'s and `common`'s `CMakeLists.txt` respectively.

![parent_B](profile/img/parent_B.svg)

![util_A](profile/img/util_A.svg)

![platform_A](profile/img/platform_A.svg)

![common](profile/img/common.svg)

Checkout the respective `CMakeLists.txt` for details.

## Shortcomings

There is no free lunch, unfortunately.
To make the above work, dependencies have to be handled via `FetchContent` or some other caching mechanism.
I.e. `git submodule`s do not work, as these will likely pull in `common` multiple times, leading to a redefinition of its CMake targets.
This redefinition breaks CMake in consequence.
