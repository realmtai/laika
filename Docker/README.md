# Go2 Docker

Containerised dev environments for the Unitree Go2, built on ROS 2 Foxy and
CycloneDDS 0.10.x.

| File | Image | Contents |
| --- | --- | --- |
| `Dockerfile.go2` | `arcs-go2` | Full stack: ROS 2 Foxy desktop, `unitree_sdk2` (C++), `unitree_sdk2_python`, `unitree_ros2` and a source-built CycloneDDS workspace. |
| `Dockerfile.ros2` | `arcs-ros2` | ROS 2 Foxy desktop plus the `unitree_ros2` workspace built against CycloneDDS 0.10.x. |

Both images talk to the robot over DDS, so they need host networking and a
CycloneDDS config pointing at the interface the Go2 is plugged into.

## Configuration

`config/cyclonedds.xml` selects the network interface used for DDS discovery.
Edit the `NetworkInterface` name to match your machine before running — find it
with `ip a`:

```xml
<NetworkInterface name="enx4cea4169512d"/>
```

The `config/` directory is mounted into the container at `/config`, and
`CYCLONEDDS_URI` already points at `/config/cyclonedds.xml`, so changes take
effect on the next `docker run` without a rebuild. `Dockerfile.go2` also bakes a
copy of `config/` into the image at `/config`, so that image still works with no
mount at all; `arcs-ros2` has no fallback and needs the mount.

### When `/config` comes up empty

**Run `docker run` from this `Docker/` directory.** `-v "$(pwd)/config:/config"`
is relative to your shell's cwd, and if `$(pwd)/config` does not exist on the
host, Docker does not error — it *creates* an empty directory there and mounts
that. The empty mount then shadows the baked-in `/config`, so the container ends
up with a `/config` that exists but holds no `cyclonedds.xml`, and DDS falls back
to default discovery on the wrong interface.

Running from the repo root is the usual way to hit this — there is no `config/`
up there, so you get a silent empty mount plus a stray `config/` directory left
behind in the repo root. Use an absolute path if you need to launch from
elsewhere:

```bash
docker run -it --network host -v /home/teo/Documents/projects/laika/Docker/config:/config arcs-go2
```

The mount is fixed at container *creation*, so
`docker start -ai arcs-go2-container` reuses whatever path was recorded by the
original `docker run`. A container created from the wrong cwd stays broken —
`docker rm -f` it and create it again.

To check what a running container actually sees:

```bash
docker exec arcs-go2-container cat /config/cyclonedds.xml  # should print the XML
docker inspect -f '{{json .Mounts}}' arcs-go2-container    # should print the host path
```

## Build

```bash
docker build -f Dockerfile.go2  -t arcs-go2  .
docker build -f Dockerfile.ros2 -t arcs-ros2 .
```

## Run

```bash
docker run -it \
  --name arcs-go2-container \
  --network host \
  -v "$(pwd)/config:/config" \
  arcs-go2
```

### With GUI and full host access

Needed for RViz, Gazebo and anything else that opens a window. Which form you
want depends on whether you are sitting at the machine or SSH'd into it.

#### Sitting at the machine

The container talks to the X server over the socket in `/tmp/.X11-unix`, so it
just needs permission. Run this on the host first:

```bash
xhost +local:docker
```

```bash
docker run -it \
  --name arcs-go2-container \
  --network=host \
  --ipc=host \
  --pid=host \
  --privileged \
  -v "$(pwd)/config:/config" \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -e QT_X11_NO_MITSHM=1 \
  arcs-go2
```

#### Over SSH — X11 forwarding and `.Xauthority`

Connect with X11 forwarding enabled:

```bash
ssh -X unitree@192.168.123.18  # -Y if -X gets rejected by extension checks
```

`-X` forwards X11 through the SSH tunnel; `-Y` is the trusted variant, which
skips the X security extension — RViz and Gazebo often need it, and it is the
usual fix when a window dies with a `BadAccess` or extension error.

Over SSH there is no local socket to share. `sshd` sets `DISPLAY` to something
like `localhost:10.0` and writes a **magic cookie** into `~/.Xauthority` on the
remote machine; without that cookie the container is refused with
`Authorization required, but no authorization protocol specified`. `xhost` will
not save you here — the auth is per-cookie, not per-host. So mount the file in
and point `XAUTHORITY` at it:

```bash
sudo docker run -it \
  --name arcs-go2-container \
  --network=host \
  --ipc=host \
  --pid=host \
  --privileged \
  -v "$(pwd)/config:/config" \
  -e DISPLAY="$DISPLAY" \
  -e XAUTHORITY=/root/.Xauthority \
  -v /home/unitree/.Xauthority:/root/.Xauthority:ro \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -e QT_X11_NO_MITSHM=1 \
  arcs-go2
```

`/home/unitree/.Xauthority` is the path on the Go2's onboard computer — use
`$HOME/.Xauthority` for any other account, and drop the `sudo` if your user is
in the `docker` group. Things that trip this up:

- `--network=host` is doing real work here, not just DDS. The forwarded display
  is a TCP port on the host's loopback, so without host networking the container
  cannot reach `localhost:10.0`.
- The file may not be where you expect. Check `echo $XAUTHORITY` — desktop
  sessions often point it somewhere like `/run/user/1000/gdm/Xauthority`
  instead, and mount that path. `xauth list` should print a cookie for your
  display; if it prints nothing, X11 forwarding did not come up and the
  container mount cannot fix it.
- Mounting `:ro` is deliberate. The cookie is a credential, and a rotated
  `.Xauthority` on the host means recreating the container to pick it up.

Swap `arcs-go2` for `arcs-ros2` to run the lighter image.

## Docker cheatsheet

### Containers

```bash
docker ps                                      # running containers
docker ps -a                                   # all containers, including stopped
docker start -ai arcs-go2-container            # restart a stopped container, attached
docker exec -it arcs-go2-container bash        # extra shell in a running container
docker stop arcs-go2-container                 # graceful stop
docker rm arcs-go2-container                   # delete a stopped container
docker rm -f arcs-go2-container                # force-delete, running or not
docker logs -f arcs-go2-container              # follow stdout/stderr
docker cp arcs-go2-container:/root/file.txt .  # copy a file out
```

`docker exec` is the one you want for a second terminal into a live session —
`docker run` would start a brand new container instead.

### Images

```bash
docker images                                            # list images
docker rmi arcs-ros2                                     # delete an image by name or ID
docker build --no-cache -f Dockerfile.go2 -t arcs-go2 .  # rebuild ignoring cache
docker history arcs-go2                                  # per-layer size breakdown
```

### Disk usage and cleanup

```bash
docker system df     # what is using disk: images, containers, volumes, cache
docker system df -v  # same, itemised
```

Cleanup, least to most destructive:

```bash
docker container prune            # remove stopped containers
docker image prune                # remove dangling (untagged) images
docker builder prune              # remove build cache
docker image prune -a             # remove every image not used by a container
docker system prune               # stopped containers + unused networks + dangling images + build cache
docker system prune -a --volumes  # everything unused, volumes included
```

Long ROS builds leave a lot of build cache behind. `docker builder prune` usually
reclaims the most space and only costs you a slower next build.

> `prune -a` will delete the `arcs-go2` and `arcs-ros2` images if nothing is
> running from them — a full rebuild takes a while, so check `docker images`
> first.

### Reset a name collision

`docker run --name` fails if the name is already taken. Either reuse the
existing container with `docker start -ai`, or drop it:

```bash
docker rm -f arcs-go2-container
```

Add `--rm` to `docker run` for throwaway sessions that clean themselves up on
exit — but note that anything written outside `/config` is lost when the
container exits.
