# Go2 Docker

Containerised dev environments for the Unitree Go2, built on ROS 2 Foxy and
CycloneDDS 0.10.x.

| File | Image | Contents |
| --- | --- | --- |
| `Dockerfile.go2` | `go2` | Full stack: ROS 2 Foxy desktop, `unitree_sdk2` (C++), `unitree_sdk2_python`, `unitree_ros2` and a source-built CycloneDDS workspace. |
| `Dockerfile.ros2` | `ros2` | ROS 2 Foxy desktop plus the `unitree_ros2` workspace built against CycloneDDS 0.10.x. |

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
effect on the next `docker run` without a rebuild.

## Build

```bash
docker build -f Dockerfile.go2  -t go2  .
docker build -f Dockerfile.ros2 -t ros2 .
```

## Run

```bash
docker run -it \
  --name go2-container \
  --network host \
  -v "$(pwd)/config:/config" \
  go2
```

### With GUI and full host access

Needed for RViz, Gazebo and anything else that opens a window. Run
`xhost +local:docker` on the host first to let the container reach your X
server.

```bash
docker run -it \
  --name go2-container \
  --network=host \
  --ipc=host \
  --pid=host \
  --privileged \
  -v "$(pwd)/config:/config" \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -e QT_X11_NO_MITSHM=1 \
  go2
```

Swap `go2` for `ros2` to run the lighter image.

## Docker cheatsheet

### Containers

```bash
docker ps                        # running containers
docker ps -a                     # all containers, including stopped
docker start -ai go2-container   # restart a stopped container, attached
docker exec -it go2-container bash   # extra shell in a running container
docker stop go2-container        # graceful stop
docker rm go2-container          # delete a stopped container
docker rm -f go2-container       # force-delete, running or not
docker logs -f go2-container     # follow stdout/stderr
docker cp go2-container:/root/file.txt .   # copy a file out
```

`docker exec` is the one you want for a second terminal into a live session —
`docker run` would start a brand new container instead.

### Images

```bash
docker images                    # list images
docker rmi ros2                  # delete an image by name or ID
docker build --no-cache -f Dockerfile.go2 -t go2 .   # rebuild ignoring cache
docker history go2               # per-layer size breakdown
```

### Disk usage and cleanup

```bash
docker system df                 # what is using disk: images, containers, volumes, cache
docker system df -v              # same, itemised
```

Cleanup, least to most destructive:

```bash
docker container prune           # remove stopped containers
docker image prune               # remove dangling (untagged) images
docker builder prune             # remove build cache
docker image prune -a            # remove every image not used by a container
docker system prune              # stopped containers + unused networks + dangling images + build cache
docker system prune -a --volumes # everything unused, volumes included
```

Long ROS builds leave a lot of build cache behind. `docker builder prune` usually
reclaims the most space and only costs you a slower next build.

> `prune -a` will delete the `go2` and `ros2` images if nothing is running from
> them — a full rebuild takes a while, so check `docker images` first.

### Reset a name collision

`docker run --name` fails if the name is already taken. Either reuse the
existing container with `docker start -ai`, or drop it:

```bash
docker rm -f go2-container
```

Add `--rm` to `docker run` for throwaway sessions that clean themselves up on
exit — but note that anything written outside `/config` is lost when the
container exits.
