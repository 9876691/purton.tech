---
title: "Faster Rust Incremental Builds in Docker"
date: 2021-12-16T15:58:40Z
---

## Long story short

By using [Mold](https://github.com/rui314/mold) a new fast linker and docker volumes for the `target` folder I'm able to get incremental build times quicker in a container than running native on a 2019 Macbook Pro.

## Introduction

I develop using VSCode devcontainer on a daily basis this means I have my development environment as code. All the tools I need are installed into the container.

This is great for being able to quickly on board new developers or to bring a new machine quickly up to speed but does sacrifice some performance.

When doing full stack we development in rust I'd ideally like sub 1 second incremental build times. That makes me feel like I'm going fast.

## Using the deno project as a benchmark

[Deno](https://deno.land/) is a runtime for Javascript and Typescriot written in rust. It comes with a `.devcontainer` folder containing a Dockerfile which we will enhance for performance.

### Testing Mac OSX 2019 with rustc v1.56

![About My Mac](/mysetup.png)

I did `clone https://github.com/denoland/deno` and then a `cargo build`. Just for good measure I run another another `cargo clean` then `cargo build` to take out any network downloads.

1. `git clone https://github.com/denoland/deno`
1. `cd deno`
1. `cargo build`

Now to test incremental build speeds I run `touch src/cache.rs` a file I picked at random. Now another `cargo build` and I'm getting incremental build times of 15 seconds. This is a standard rust installation on OSX.

```bash
ianpurton@MacBook-Pro deno % cargo build       
   Compiling deno v1.16.4 (/Users/ianpurton/Documents/Scratch/deno/cli)
    Finished dev [unoptimized + debuginfo] target(s) in 14.75s
```

### Debug = false

If we set `debug = false` in our `Cargo.toml` we can improve the incrementl build times, however, in this case not by much.

We shave off another second.

```bash
ianpurton@MacBook-Pro deno % touch cli/cache.rs
ianpurton@MacBook-Pro deno % cargo build       
   Compiling deno v1.16.4 (/Users/ianpurton/Documents/Scratch/deno/cli)
    Finished dev [unoptimized] target(s) in 13.72s
```

So that's our time to beat. ***13.72s***

## How we setup the devcontainer

We need to move the existing `.devcontainer` setup to be docker compose based. This is so we can add a volume for the target folder which speeds up the build in docker considerably.

```yaml
# .devcontainer/docker-compose.yml
version: '3.4'
services:
  development:
    build: 
      context: .
      dockerfile: Dockerfile
    volumes:
      - ..:/vscode:delegated
      # For building dependencies always use a volume, you get waaaay better performance.
      - target:/vscode/target
    # Overrides default command so things don't shut down after the process ends.
    command: sleep infinity
    working_dir: /vscode
volumes:
  # The volume for cargo build
  target:
```

```Dockerfile
# .devcontainer/Dockerfile

# Build stage to create our mold executable
FROM ubuntu:20.04 as mold

RUN apt-get update && \
  TZ=Europe/London apt-get install -y tzdata && \
  apt-get install -y build-essential git clang lld cmake \
    libstdc++-10-dev libxxhash-dev zlib1g-dev libssl-dev && \
  rm -rf /var/lib/apt/lists/*

RUN git clone https://github.com/rui314/mold.git 

ENV LDFLAGS='-fuse-ld=lld -static'
ENV LDFLAGS="$LDFLAGS -Wl,-u,pthread_rwlock_rdlock"
ENV LDFLAGS="$LDFLAGS -Wl,-u,pthread_rwlock_unlock"
ENV LDFLAGS="$LDFLAGS -Wl,-u,pthread_rwlock_wrlock"

RUN make -C /mold -j$(nproc) EXTRA_LDFLAGS="$LDFLAGS"

# Our rust container
FROM mcr.microsoft.com/vscode/devcontainers/rust:0-1

# Update to Rust 1.56.1
RUN rustup update 1.56.1 && rustup default 1.56.1

# Install Deno
ENV DENO_INSTALL=/usr/local
RUN curl -fsSL https://deno.land/x/install/install.sh | sh

RUN sudo mkdir -p /vscode/target && sudo chown vscode:vscode /vscode/target

# Copy over mold from our build stage
COPY --chown=vscode --from=mold /mold/mold /usr/bin/mold
COPY --chown=vscode --from=mold /mold/mold-wrapper.so /usr/bin/mold-wrapper.so
```

```json
// .devcontainer/devcontainer.json
{
  "name": "Rust",
	"dockerComposeFile": [
		"docker-compose.yml"
	],
	"service": "development",
  "workspaceFolder": "/vscode",
  "runArgs": ["--cap-add=SYS_PTRACE", "--security-opt", "seccomp=unconfined"],

  "settings": {
    "lldb.executable": "/usr/bin/lldb",
    // VS Code don't watch files under ./target
    "files.watcherExclude": {
      "**/target/**": true
    }
  },

  "extensions": [
    "matklad.rust-analyzer",
    "bungcip.better-toml",
    "vadimcn.vscode-lldb",
    "mutantdino.resourcemonitor"
  ],

  "remoteUser": "vscode"
}
```


## Docker build time

Let's try an incremental build without mold. Here I'm running inside the `devcontainer` in vscode.

```bash
vscode ➜ /vscode (main ✗) $ touch cli/cache.rs 
vscode ➜ /vscode (main ✗) $ cargo build
   Compiling deno v1.16.4 (/vscode/cli)
    Finished dev [unoptimized] target(s) in 26.10s
```

As you would expect running in a container on a docker desktop is slower than native.

## Mold

To use the mold linker prefix your cargo commands with `mold --run`. Again I'm running in the vscode devcontainer.

```bash
vscode ➜ /vscode (main ✗) $ touch cli/cache.rs 
vscode ➜ /vscode (main ✗) $ mold --run cargo build
   Compiling deno v1.16.4 (/vscode/cli)
    Finished dev [unoptimized] target(s) in 9.93s
```

So now we are faster than running a standard OSX rust install. 

## Conclusion

There we have it, due to the parallel nature of the mold linker we can get fast build times even under a virtualised docker environment.

|Environment|Incremental Build Time|
|-----------|----------------------|
| Rust on MacOS|13.72s|
| Rust in Debian VSCode Devcontainer|9.93s|

[Mold](https://github.com/rui314/mold) is building out OSX support so in the future perhaps we get even faster native build times.

