---
date: 2024-07-29
description: Some helpful tips about containerizing Julia services.
---
Some helpful tips about containerizing Julia services.
## Creating a Dockerfile

### Template

Here's an example Dockerfile you can use to get started:

```dockerfile
FROM ubuntu:22.04

ARG JULIA_VERSION_SHORT=1.10
ARG JULIA_VERSION_LONG=1.10.4

RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends curl wget ca-certificates git openssh-client clang tar unzip && \
    rm -rf /var/lib/apt/lists/*

RUN wget -nv https://julialang-s3.julialang.org/bin/linux/x64/$JULIA_VERSION_SHORT/julia-$JULIA_VERSION_LONG-linux-x86_64.tar.gz && \
    tar xf julia-$JULIA_VERSION_LONG-linux-x86_64.tar.gz && \
    rm julia-$JULIA_VERSION_LONG-linux-x86_64.tar.gz
RUN ln -s /julia-$JULIA_VERSION_LONG/bin/julia /usr/local/bin/julia

# TODO: Change me!
ENV SERVICEPATH=/runtime/Service.jl
COPY Service.jl/Project.toml Service.jl/Manifest.toml Service.jl/clean_julia.sh $SERVICEPATH/
COPY Service.jl/src $SERVICEPATH/src/

# TODO: Change me!
ENV JULIA_CPU_TARGET="generic;sandybridge,-xsaveopt,clone_all;haswell,-rdrnd,base(1)"

RUN julia -t auto -e 'using Pkg; \
    Pkg.activate(ENV["SERVICEPATH"]); \
    Pkg.Registry.add([RegistrySpec(name="General")]); \
    retry(Pkg.instantiate)(); \
    Pkg.build(); \
    Pkg.precompile();' && \
    $SERVICEPATH/clean_julia.sh

COPY Service.jl/entrypoint.sh /runtime/entrypoint.sh
RUN chmod +x /runtime/entrypoint.sh

ENTRYPOINT [ "/runtime/entrypoint.sh" ]
```

### JULIA_CPU_TARGET

This is relevant even if you are not creating a sysimage or pkgimage.

`JULIA_CPU_TARGET` must be set correctly for the union of all architectures you run the code on; notably, the architecture you deploy to is probably different than what you build on. For example, you might build on GitHub Actions (which actually uses multiple architectures) and deploy to an `m6i` type EC2 instance. In this case, you should set `JULIA_CPU_TARGET` to `generic;skylake-avx512,clone_all`. Pay attention to whether you encounter precompilation in your target environment; if you do, this is an indication that you did not set `JULIA_CPU_TARGET` correctly.

Note that Julia v1.9 and earlier may fail to invalidate the compile cache and encounter bad instructions at runtime. Julia v1.10 fixes this problem.

You can set multiple architectures in `JULIA_CPU_TARGET` and the correct one will be loaded at runtime. This works with sysimages, and recently also works with pkgimages. Check the [Julia docs](https://docs.julialang.org/en/v1/devdocs/sysimg/#sysimg-multi-versioning) for more information.

### Cold Start Time

Building a container is a perfect time to optimize your program's load time, especially if your deployment will be sensitive to the cold start time of the container.
You can build a [sysimage](https://julialang.github.io/PackageCompiler.jl/stable/sysimages.html#sysimages) of your Julia application using [PackageCompiler.jl](https://julialang.github.io/PackageCompiler.jl/stable/) and load that sysimage when starting Julia in your container's entrypoint.

You can also try building a [pkgimage](https://docs.julialang.org/en/v1/devdocs/pkgimg/), but that is in a less stable state.

Neither solution will eliminate load times, but will still improve time-to-first-x (TTFX) latency by not relying on the JIT compiler.

### Remove Extra Files

When Pkg.jl installs your dependencies, all the files from those packages repositories are installed. This includes files which are not needed at runtime (documentation, build scripts, etc.). Notably, this can also include other code (projects used to generate bindings, etc.). If you use a tool like [Trivy](https://aquasecurity.github.io/trivy) to generate an SBOM of your container, you will see many additional Julia packages in the container. These can be removed using a script that runs after Pkg.jl is done installing, building, and precompiling. Note that you need to run this script in the same build step as your other Pkg.jl code; otherwise, these files will still be comitted into an earlier layer in the container.

```sh
#!/bin/bash

set -o errexit
set -o nounset
set -o pipefail

# Remove the standard Julia test dir in all packages
find "$HOME/.julia/packages" -type d -name test -exec rm -rf {} +;

# Remove CI/CD stuff
find "$HOME/.julia/packages" -type d -name .github -exec rm -rf {} +;
find "$HOME/.julia/packages" -type d -name .fluxbot -exec rm -rf {} +;

# Other random stuff
find "$HOME/.julia/packages" -type d -name benchmark -exec rm -rf {} +;
find "$HOME/.julia/packages" -type d -name benchmarks -exec rm -rf {} +;
find "$HOME/.julia/packages" -type d -name docs -exec rm -rf {} +;
find "$HOME/.julia/packages" -type d -name dev -exec rm -rf {} +;
find "$HOME/.julia/packages" -type d -name res -exec rm -rf {} +;
find "/" -type d -name .devcontainer -exec rm -rf {} +;

# Can't remove entire gen dirs because we need some of those files but we can remove the manifests used to generate them
find "$HOME/.julia/packages" -type d -name gen | xargs -I RR rm -rf RR/Manifest.toml

# stdlib tests
find "julia-$JULIA_VERSION_LONG/share/julia/stdlib" -type d -name test -exec rm -rf {} +;
find "julia-$JULIA_VERSION_LONG/share/julia/stdlib" -type d -name docs -exec rm -rf {} +;
find "julia-$JULIA_VERSION_LONG/share/julia/stdlib" -type d -name benchmark -exec rm -rf {} +;
find "julia-$JULIA_VERSION_LONG/share/julia/stdlib" -type d -name benchmarks -exec rm -rf {} +;
rm -rf "julia-$JULIA_VERSION_LONG/share/julia/test"

# Remove .julia stuff that isn't needed after precompilation/building is done
rm -rf "$HOME/.julia/registries"
rm -rf "$HOME/.julia/logs"
rm -rf "$HOME/.julia/clones"
```

### Make Sure You Use The Correct Julia Version

I find it's useful to make sure you're running the Julia version you really think you are:

```sh
ARG JULIA_VERSION_SHORT=1.10
ARG JULIA_VERSION_LONG=1.10.4
# install Julia using those ARGs

# later on...
RUN test "$JULIA_VERSION_LONG" = "$(grep julia_version /path/to/Manifest.toml | cut -d' ' -f3 | xargs)"
```

It's easy to accidentally update your Manifest.toml without remembering to also update your Dockerfile.

## Source Layout

I mainly containerize services; I've found it helpful to keep all the code and configuration related to that service in the same repository (i.e. a monorepo per service). An example layout from one of my Julia services:

```
- Service.jl
    - src
    - test
    - Dockerfile
    - Manifest.toml
    - Project.toml
    - clean_julia.sh
    - entrypoint.sh
    - <other files omitted>
- infra (terraform)
    - infra_prod
    - infra_stage
    - modules
- src
- test
    - service
        - integ
        - smoke
    - <test files for the root package>
- Manifest.toml
- Project.toml
- <other files omitted>
```
