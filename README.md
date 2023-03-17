# CMake PAUL

Platform agnostic utility libraries using CMake.

## Why this organization?

At my day-job, I recently got confronted with some utility libraries written in `C/C++`.
The libraries were required to run on different platforms (to be precise: different targets in an embedded software development setting).
I.e. the libraries were expected to be compileable for different platforms without source code changes required.
The different platforms shared a set of common functionality, differing in subtle ways in how this functionality is provided.

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
  These parent project resemble different self-contained applications.
- The utility libraries are expected to compile for different platforms without any source code change required.
  I.e. the utility libraries may not rely on subtleties of one platform or the other.
- The utility libraries do not know, which platform they are running on.
  I.e. it is up to the parent project to pick one of the platforms available.
- All the components are expected to be self-contained.
  I.e. apart from deciding which platform to use, there should not be any additional plumbing required from a parent project in order to use one of the utility libraries on top of one of the platforms.

## My take

In the following `parent_X`, `util_X` and `platform_X` represent a parent project, an utility library and a platform respectively.

- To represent the common functionality shared by all platforms, we introduce an `INTERFACE` library `common`.
  `common` provides a common set of declarations of functions, types, etc.
  `common` abstracts away the differences of one platform compared to another.
  `common` is the only way `util_X` accesses `platform_X`'s capabilities - i.e. there is no direct access to `platform_X` itself.
  `platform_X` on the other hand provides implementations of everything declared in `common`.
