Package a Rust application
==========================

In this guide, you will learn how to package your average Rust application as an Ubuntu package that conforms to the
Ubuntu packaging guidelines.
Rust programming language knowledge is not required to follow this guide, if you are not modifying the application you
are trying to package.

.. note::
    Please note that as the Rust ecosystem is vibrant and fast-moving, this guide will not and can not cover all the cases.

Introduction
------------

Due to the pecularities of the Rust ecosystem, we will use a slightly different way of explaining how to make a package of Rust applications.

There are numerous types of Rust applications, but from the packaging stand-point, we can define them to have one or more of these traits:

- Uses non-Cargo build systems
- Contains executable binaries
- Contains shared libraries that export C APIs
- Uses Cargo workspace to host multiple sub-projects
- Contains preprocessed JavaScript code
- Uses system or vendored C/C++ libraries
- Contains native extensions or plugins for another programming language

In the following sections, we will review and suggest how you may handle applications with these traits.

Rust Build System Basics
------------------------

Rust provides a build system called Cargo, which is designed to be integrated into the Rust ecosystem.
Most Rust developers will choose this build system over other language-agnostic build systems due to easy to use with Rust projects.
Cargo also handles the download and compilation of external Rust libraries (called "crates") hosted on crates.io website. These external
Rust libraries will be built and statically linked into the final binary files, so most Rust applications can be called "mostly statically-linked"
applications, and do not require a lot of system libraries to function.

This obviously will be an issue for Ubuntu, as we don't have Internet access in the build environment for the build system to download those
external libraries. But luckily, we could prepare these files in advance, using a process called "vendoring".

"Vendoring" can be easily done via running the ``cargo vendor --locked`` command from the root directory of the project (the outermost directory
where the Cargo.lock file is located). If the project does not provide the ``Cargo.lock`` file, you can run ``cargo generate-lockfile`` to create
one.

Cargo build system will include all the external libraries that will be needed to build the project, this also includes the libraries intended to use
on other operating systems, like Windows or macOS. These libraries often times can be very large and take a lot of disk space. We can trim them
down to a smaller size by using the ``cargo-vendor-filterer`` tool: https://crates.io/crates/cargo-vendor-filterer. You will need to install the tool
yourself as we currently do not have the tool available in the Ubuntu archive. To manually install this tool, you can run this command:

    cargo install -f cargo-vendor-filterer

By running ``cargo vendor-filterer --platform='*-unknown-linux-gnu'``, we can auto-remove all the files that are not used for building this project on Ubuntu.

Then you can pack the ``vendor`` directory as a separate original tarball like this: ``tar cJf ../<package_name>_<package_version>.orig-vendor.tar.xz vendor``.

Non-Cargo build systems
-----------------------

There are, however, some projects choose to use build systems like CMake and Meson to build their Rust projects.
You will need to investigate how the project use the other build system to build the Rust parts. It is very likely that the build system is using
Cargo in some forms under the hood.

The common issues with non-Cargo build systems include:

- Does not correctly pass the ``RUSTFLAGS`` environment variables (patching the build scripts needed)
- Can not find the correct Rust compiler (maybe fixable by setting the ``RUSTC`` environment variable)
- Does not correctly setup the Rust workspace and does not use the offline vendored source files (patching the build scripts needed)

Executable binaries
-------------------

For those with (only) executable binaries, you can directly use ``dh_cargo`` to build and install executable binaries automatically.
Here is a simple example:

.. code-block:: make

    %:
        dh $@ --buildsystem=cargo

Shared libraries that export C APIs
-----------------------------------

``dh_cargo`` does not handle projects that output shared libraries.
However, you might be use a workaround as demostrated below:

.. code-block:: make

    override_dh_auto_install:
        # here we build the project called "cramjam"
        DEB_HOST_GNU_TYPE=$(DEB_HOST_GNU_TYPE) \
        DEB_HOST_RUST_TYPE=$(DEB_HOST_RUST_TYPE) \
        CARGO_HOME=$(CURDIR)/debian/cargo_home \
        DEB_CARGO_CRATE=$(CRAMJAM_CRATE_NAME)_$(CRAMJAM_CRATE_VERSION) \
        $(CARGO) build --release -v
        # manually install the built binary to the staging area
        install -Dvm755 $(CURDIR)/target/$(DEB_HOST_RUST_TYPE)/release/libcramjam.so $(CURDIR)/debian/tmp/usr/lib/$(DEB_HOST_GNU_TYPE)/libcramjam.so

The Rust toolchain in the Ubuntu archive has been patched to emit ``SONAME`` tags by default for libraries that export C APIs.

Uses Cargo workspace to host multiple sub-projects
--------------------------------------------------

Cargo workspace is a feature of Cargo that allows the developer to manage multiple Rust projects together.
This creates a challenge because ``dh_cargo`` was not designed to work inside a workspace.
However, you might be use a workaround as demostrated below:

.. code-block:: make

    override_dh_auto_install:
        # here we build the project called "keylime_agent"
        DEB_HOST_GNU_TYPE=$(DEB_HOST_GNU_TYPE) \
        DEB_HOST_RUST_TYPE=$(DEB_HOST_RUST_TYPE) \
        CARGO_HOME=$(CURDIR)/debian/cargo_home \
        DEB_CARGO_CRATE=$(KEYLIME_CRATE_NAME)_$(KEYLIME_CRATE_VERSION) \
        $(CARGO) build --release -v -p keylime_agent
        # manually install the built binary to the staging area
        install -Dvm755 $(CURDIR)/target/$(DEB_HOST_RUST_TYPE)/release/keylime_agent $(CURDIR)/debian/tmp/usr/bin/keylime_agent

Contains preprocessed JavaScript code
--------------------------------------

Some Rust projects may contain a JavaScript/TypeScript sub-project to provide web server or GUI functionality.
You will need to check if the project builds the JavaScript/TypeScript sub-project during the build or ships
a pre-built copy of minified/obfuscated JavaScript bundle inside.

In the latter case, you might want to figure out a way to build the JavaScript bundle from scratch and strip
out the pre-built JavaScript bundle for security reasons.

Some projects may also "prime" the JavaScript bundle during the build time to generate JavaScript engine
cache to optimize start-up performance. You might want to audit if the project will inject some other
JavaScript code snippets during the "prime" process in the project build script (typically in the ``build.rs`` file).

.. TODO: mention Tauri/Deno stuff?

Uses system or vendored C/C++ libraries
-----------------------------------------

Some Rust projects may use system libraries or vendored C/C++ libraries.

Usually we want to remove these vendored libraries and use the system ones instead. Commonly vendored libraries include:

- OpenSSL
- libssh2
- libcurl
- zlib
- libzstd
- SQLite
- libgit2

You can choose to remove only the C/C++ code from the libraries, as most of those Rust libraries can automatically find and use
the system libraries if you have the system shared libraries installed.

Contains native extensions or plugins for another programming language
----------------------------------------------------------------------

There are increasing number of Node.js and Python projects using Rust to speed up their libraries.

For Python projects, usually they will use ``maturin`` to build the Python packages. Some packages may be buildable using the
"Shared libraries that export C APIs" method.
For other projects that relies on ``setuptools-rust``, you can use the ``pybuild`` build system instead like this:

.. code-block:: make

    export DH_VERBOSE=1
    export PYBUILD_NAME=cramjam
    export PYBUILD_DIR=cramjam-python
    export CARGO_NET_OFFLINE=true
    export CARGO_HOME=$(shell pwd)/debian/cargo_home

    %:
        dh $@ --with python3 --buildsystem=pybuild

.. TODO: more complicated situations?