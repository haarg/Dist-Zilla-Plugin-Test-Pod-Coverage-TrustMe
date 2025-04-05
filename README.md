# NAME

Dist::Zilla::Plugin::Test::Pod::Coverage::TrustMe - An author test for Pod Coverage

# SYNOPSIS

```
# Add this line to dist.ini
[Test::Pod::Coverage::TrustMe]

# Run this in the command line to test for POD coverage:
$ dzil test --release
```

# DESCRIPTION

This extension provides a test file to check for Pod coverage using
[Test::Pod::Coverage::TrustMe](https://metacpan.org/pod/Test%3A%3APod%3A%3ACoverage%3A%3ATrustMe). It is meant as a replacement for the
[\[PodCoverageTests\]](https://metacpan.org/pod/Dist%3A%3AZilla%3A%3APlugin%3A%3APodCoverageTests) plugin, but
providing some additional options and using a different underlying test module.

# OPTIONS

- filename

    The name of the test file to generate. Defaults to `xt/author/pod-coverage.t`.

- modules

    The modules to check for coverage.

    Aliased as `module`.

- finder

    The [file finder](https://metacpan.org/pod/Dist%3A%3AZilla%3A%3ARole%3A%3AFileFinderUser#default_finders)
    used to find modules to check. Will only be used if a list of modules is not
    given. Defaults to `:InstallModules`.

    To generate the list of modules, the output of this finder will be filtered to
    only files ending in `.pm` and will be transformed to module names after
    stripping an initial `lib/`.

- private

    Methods to treat as private, with their coverage not checked. Will be passed to
    [Test::Pod::Coverage::TrustMe](https://metacpan.org/pod/Test%3A%3APod%3A%3ACoverage%3A%3ATrustMe). String values can be passed directly. Regexp
    values can be passed with the form `/.../`. Can be specified multiple times.

- also\_private

    Exactly the same as ["private"](#private), but adds to the list of default private methods
    rather than replacing it.

- trust\_methods

    A list of methods to always treat as covered. Will be passed to
    [Test::Pod::Coverage::TrustMe](https://metacpan.org/pod/Test%3A%3APod%3A%3ACoverage%3A%3ATrustMe). Can be specified multiple times.

- require\_content
- trust\_parents
- trust\_roles
- trust\_packages
- trust\_pod
- require\_link
- export\_only
- ignore\_imported

    These options are all passed directly to [Test::Pod::Coverage::TrustMe](https://metacpan.org/pod/Test%3A%3APod%3A%3ACoverage%3A%3ATrustMe).

- options

    Additional options to pass to [Test::Pod::Coverage::TrustMe](https://metacpan.org/pod/Test%3A%3APod%3A%3ACoverage%3A%3ATrustMe). Options should
    be specified like:

    ```
    options = extra_option = 1
    ```

# BUGS

Please report any bugs or feature requests on the bugtracker website
[https://github.com/haarg/Dist-Zilla-Plugin-Test-Pod-Coverage-TrustMe/issues](https://github.com/haarg/Dist-Zilla-Plugin-Test-Pod-Coverage-TrustMe/issues)

When submitting a bug or request, please include a test-file or a
patch to an existing test-file that illustrates the bug or desired
feature.

# AUTHOR

Graham Knop <haarg@haarg.org>

# COPYRIGHT AND LICENSE

This software is copyright (c) 2025 by Graham Knop.

This is free software; you can redistribute it and/or modify it under
the same terms as the Perl 5 programming language system itself.
