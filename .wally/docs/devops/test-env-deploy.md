# Test environment deployment

## Sources and working directory

Derived from `INSTALL.md`, `autogen.sh`, `Makefile.in`, `bin/jemalloc-config.in` and the CI configs (`.travis.yml`, `.cirrus.yml`, `.appveyor.yml`). Working directory: repository root. The repository contains no Dockerfile, Compose file or deployment manifest, so jemalloc is deployed as a library installed into a prefix (`include/`, `lib/`, `bin/`) rather than as a service.

## Prerequisites and authorization

`autoconf`, a C/C++ compiler and `make`. A git checkout has no committed `configure`, so `./autogen.sh` must run first (it runs `autoconf`, then `./configure --enable-autogen`). For a packaged release use `./configure` directly. No target environment, owner, install prefix/host or access method is documented; the authorization boundary is undefined.

## Build and deploy (real commands)

    ./autogen.sh && make && make install

Packaged-release flow from `INSTALL.md`: `./configure && make && make install`. Set the install prefix with `./configure --prefix=<dir>` or stage it with `DESTDIR` at install time. There is no separate artifact synchronization or service deployment step.

## Verification

`jemalloc-config --version` (in the build tree: `bin/jemalloc-config --version`) prints the built version; compare it with the intended release. `make check` exercises the unit and integration test suite as a build-level smoke check.

## Failure and rollback

No rollback version, trigger or commands are defined. `make clean`, `make distclean` and `make relclean` only remove build artifacts; they do not uninstall installed files.

## Gaps (readiness: pending)

1. No target test/deployment environment (no id, owner, host, prefix or access method).
2. No credential acquisition method for a target environment.
3. No rollback version or commands.
4. No deployment acceptance criteria or effective-version baseline for a target environment.

This file records static verification only; it is not evidence of execution.
