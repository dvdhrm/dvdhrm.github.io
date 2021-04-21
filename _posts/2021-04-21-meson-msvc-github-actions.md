---
layout: post
caption: Meson with MSVC on GitHub Actions
categories: [fedora]
tags: [actions, github, meson, microsoft, msvc, visual studio]
---

The Meson Build System provides support for running on Microsoft Windows,
including support for Microsoft Visual Studio C++. GitHub Actions provides
public access to CI machines running Microsoft Windows. But trying to tie both
together is not as straightforward as it sounds.

Sometimes you stumble over a task you never thought you have to deal with. This
story is about one of those times. In particular, I was faced with running CI
tests for a simple C library on
[Microsoft Visual Studio C++](https://en.wikipedia.org/wiki/Microsoft_Visual_C%2B%2B)
(_MSVC_). Gladly, GitHub already provides simple access to machines running
Microsoft Windows Server 2016 and 2019, so this sounded like a straightforward
task. Unfortunately, my infinite ignorance of anything Windows made this harder
than it should have been.

The root of this problem is that the
[Meson Build System](https://en.wikipedia.org/wiki/Meson_%28software%29)
needs to run in the _MSVC Developer Shell_. This shell has all the necessary
environment variables prepared for a particular install of _MSVC_. Since you
can have multiple versions installed in parallel, _Meson_ cannot know which
install to use if run outside of such a shell. Unfortunately, *GitHub Actions*
has no simple way to enter this shell. Therefore, running *Meson* on
*GitHub Actions* will end up using *GCC* rather than *MSVC*, since this is what
it detects by default in the *GitHub Actions Environment*. This is not what we
wanted, so adjustments are needed.

Luckily, Microsoft provides a tool called
[*vswhere*](https://github.com/microsoft/vswhere)
which finds _MSVC_ installs on a Windows system. We can use this to find the
required setup scripts and then import the environment variables into our
GitHub Actions setup. This tool is pre-deployed on *GitHub Actions*, so we can
simply invoke it to find a suitable _MSVC_ install. From there on, we look for
`DevShell.dll`, which provides the required integration. We load it into
PowerShell and invoke the provided `Enter-VsDevShell` function. By comparing
our own environment variables before and after that call, we can extract the
changes and export them into the _GitHub Actions_ environment. Thus,
the following workflow-steps will have access to those variables as well.

I plugged this into a re-usable _GitHub Action_ using the new _composite_ type.
To use it in a _GitHub Actions_ workflow, simply use:

```yml
- name: Prepare MSVC
  uses: bus1/cabuild/action/msdevshell@v1
  with:
    architecture: x64
```

This queries the _MSVC_ environment and exports it to your _GitHub Actions_
job. Following steps will thus run as if in an _MSVC Developer Shell_. A full
example is appended at the bottom, which shows how to get _Meson_ to compile
and test a project on _MSVC_ for both *Windows Server 2016* and *2019*.

If you rather import the code into your own project, you can find it on
[GitHub](https://github.com/bus1/cabuild/blob/8c91ebf06b7a5f8405cf93c89a6928e4c76967e0/action/msdevshell/action.yml).
Note that this uses PowerShell syntax, so it might look alien to linux
developers.

While this is only roughly 50 lines of PowerShell scripting, it still feels a
bit too hacky. The _Meson_ developers are aware of this, but so far no patches
have found their way upstream. Lets hope that this workaround will one day be
obsolete and _Meson_ invokes _vswhere_ itself.

---

Following a full example workflow:

```yml
name: Continuous Integration

on: [push, pull_request]

jobs:
  ci-msvc:
    name: CI with MSVC
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [windows-2016, windows-latest]

    steps:
    - name: Fetch Sources
      uses: actions/checkout@v2
    - name: Setup Python
      uses: actions/setup-python@v2
      with:
        python-version: '3.x'
    - name: Install Python Dependencies
      run: pip install meson ninja
    - name: Prepare MSVC
      uses: bus1/cabuild/action/msdevshell@v1
      with:
        architecture: x64
    - name: Prepare Build
      run: meson setup build
    - name: Run Build
      run: meson compile -v -C build
    - name: Run Test Suite
      run: meson test -v -C build
```
