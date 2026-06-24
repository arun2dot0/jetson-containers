# Running Containers

Let's say that you found a container image from the [Package List](/packages) or [DockerHub](https://hub.docker.com/u/dustynv), or [built your own container](/docs/build.md) - the normal way to run an interactive Docker container on your Jetson would be using [`docker run`](https://docs.docker.com/engine/reference/commandline/run/) like this:

``` bash
$ sudo docker run --runtime nvidia -it --rm --network=host CONTAINER:TAG
```

That's actually a rather minimal command, and doesn't have support for displays or other devices, and it doesn't mount the model/data cache ([`/data`](/data)). Once you add everything in, it can get to be a lot to specify by hand.  Hence, we have some helpers that provide shortcuts.

The [`jetson-containers run`](/run.sh) launcher can be run from any directory and forwards its command-line to [`docker run`](https://docs.docker.com/engine/reference/commandline/run/), with some added defaults - including the above flags, mounting the `/data` cache, and mounting various devices for display, audio, and video (like V4L2 and CSI cameras)

``` bash
$ jetson-containers run CONTAINER:TAG                   # run with --runtime=nvidia, default mounts, ect
$ jetson-containers run CONTAINER:TAG my_app --abc xyz  # run a command (instead of interactive mode)
$ jetson-containers run --volume /path/on/host:/path/in/container CONTAINER:TAG  # mount a directory
```

The flags and arguments to [`jetson-containers run`](/run.sh) are the same as they are to [`docker run`](https://docs.docker.com/engine/reference/commandline/run/) - anything you specify will be passed along.

## `autotag`

To solve the issue of finding a container with package(s) that you want and that's compatible with your version of JetPack/L4T, there's the [`autotag`](/autotag) tool.  It locates a container image for you - either locally, pulled from a registry, or built from source:

``` bash
$ jetson-containers run $(autotag pytorch)   # find pytorch container to run for your version of JetPack/L4T
```

What's happening here with the `$(autotag xyz)` syntax, is that Bash command substitution expands the full container image name and forwards it to the `docker run` command.  For example, if you do `echo $(autotag pytorch)` it would print out something like `dustynv/pytorch:r35.2.1` (assuming that you don't already have the `pytorch` image locally).

You can of course use [`autotag`](/autotag) interspersed along with other command-line arguments to launch the container:

``` bash
$ jetson-containers run $(autotag pytorch) my_app --abc xyz  # run a command (instead of interactive mode)
$ jetson-containers run --volume /path/on/host:/path/in/container $(autotag pytorch)  # mount a directory
```

Or with using [`docker run`](https://docs.docker.com/engine/reference/commandline/run/) directly:

``` bash
$ sudo docker run --runtime nvidia -it --rm --network=host $(./autotag pytorch)
```

This is the order in which [`autotag`](/autotag) searches for container images:

1. Local images (found under `docker images`)
2. Pulled from registry (by default [`hub.docker.com/u/dustynv`](https://hub.docker.com/u/dustynv))
3. Build it from source (it'll ask for confirmation first)

When searching for images, it knows to find containers that are compatible with your version of JetPack-L4T.  For example, if you're on JetPack 4.6.x (L4T R32.7.x), you can run images that were built for other versions of JetPack 4.6.  Or if you're on JetPack 5.1 (L4T R35), you can run images built for other versions of JetPack 5.1 (and likewise for JetPack 6.0 and newer)

## `jtop`

If you have installed [**jetson-stats**](https://github.com/rbonghi/jetson_stats) (or `jtop`) on your host, now a container with jetson-stats (`jtop`) installed can work inside the container by communicating with host server through a socket `/run/jtop.sock` (with `-v /run/jtop.sock:/run/jtop.sock` argument for `docker run`).

Check the [official documentation](https://rnext.it/jetson_stats/docker.html) for the detail.

Make sure you install the same version of jetson-stats (`jtop`) both on your host and in the container.

## Running as Non-Root

By default all containers run as `root`. Set `DOCKER_USER` to run as a different identity:

```bash
DOCKER_USER=1000:1000 jetson-containers run $(autotag pytorch)
# or by username (the user must exist inside the image):
DOCKER_USER=jetson jetson-containers run $(autotag pytorch)
```

The launcher automatically adds `--group-add video --group-add audio --group-add i2c --group-add dialout --group-add plugdev` so that common hardware devices remain accessible. X11 forwarding is also updated to grant the declared user display access.

### Limitations when running non-root

| Feature | Root required? | Notes |
|---------|:--------------:|-------|
| GPU / CUDA / TensorRT | No | `--runtime nvidia` grants device access regardless of UID |
| LLM / VLM inference | No | No hardware devices needed |
| V4L2 cameras | No | Covered by `--group-add video` |
| USB devices | No | Covered by `--group-add plugdev` |
| Audio (PulseAudio) | No | Covered by `--group-add audio` |
| I2C / serial sensors | No | Covered by `--group-add i2c` / `dialout` |
| **CSI cameras (Argus)** | **Yes** | `nvargus-daemon` owns `/tmp/argus_socket` as root; incompatible with `DOCKER_USER` |
| `--csi2webcam` pipeline | No (host-side) | `modprobe` runs on host; GStreamer inside container needs `video` group |
| `/data` volume writes | Depends | Host `data/` dir must be writable by the container UID |
| Docker socket (`docker.sock`) | No | Requires `docker` group inside image |

### Building images that support non-root

Pass identity at build time via env vars and consume them in the Dockerfile:

```bash
export CONTAINER_USER=jetson CONTAINER_UID=1000 CONTAINER_GID=1000
jetson-containers build mypackage
```

These are forwarded as `--build-arg CONTAINER_USER=jetson --build-arg CONTAINER_UID=1000 --build-arg CONTAINER_GID=1000`. In the Dockerfile:

```dockerfile
ARG CONTAINER_USER=root
ARG CONTAINER_UID=0
ARG CONTAINER_GID=0

RUN if [ "$CONTAINER_UID" != "0" ]; then \
        groupadd -g $CONTAINER_GID $CONTAINER_USER && \
        useradd -u $CONTAINER_UID -g $CONTAINER_GID -m $CONTAINER_USER; \
    fi

USER $CONTAINER_USER
```

Declare the supported user in package metadata so tooling can discover it:

```yaml
run_user: jetson
```

## Runtime Secrets

Secrets passed as plain `--env` flags are visible in `docker inspect` output and the process list. Prefer file-backed secrets instead.

### HuggingFace token

```bash
# Preferred: token stays out of docker inspect / shell history
HUGGINGFACE_TOKEN_FILE=~/.hf_token jetson-containers run $(autotag llama3)

# Fallback (backwards-compatible, token visible in docker inspect):
HUGGINGFACE_TOKEN=hf_xxx jetson-containers run $(autotag llama3)
```

`HUGGINGFACE_TOKEN_FILE` mounts the file read-only at `/run/secrets/huggingface_token` inside the container and sets `HF_TOKEN_FILE` / `HUGGINGFACE_TOKEN_FILE` env vars pointing to it.

### Build-time secrets

Packages can declare secrets they need at build time. The value is never baked into an image layer:

```yaml
# in Dockerfile header or config.yaml
secrets: [huggingface_token]
```

```bash
export JETSON_SECRET_HUGGINGFACE_TOKEN=~/.hf_token
jetson-containers build mypackage
```

Inside the Dockerfile, consume via `--mount=type=secret`:

```dockerfile
RUN --mount=type=secret,id=huggingface_token \
    HF_TOKEN=$(cat /run/secrets/huggingface_token) \
    python3 download_model.py
```

If `JETSON_SECRET_<NAME>` is not set, a warning is printed and the build continues without that secret.