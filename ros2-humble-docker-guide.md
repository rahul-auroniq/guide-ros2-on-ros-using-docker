# Running ROS2 Humble (Docker) on a ROS1 Host Machine

This guide covers: Docker basics/cheat sheet, running the ROS2 Humble container correctly on a ROS1 machine, building and using a ROS2 workspace inside the container, and bridging ROS1 ↔ ROS2 topics (the piece most people need in this exact setup).

---

## 1. The Setup, Conceptually

- **Host OS**: has ROS1 installed natively (e.g. Noetic).
- **Container**: `osrf/ros:humble-desktop` (or similar) running ROS2 Humble, isolated from the host's ROS1 install.
- Both can talk to each other over the network (`--network host`) since ROS1 and ROS2 use different discovery/transport (ROS1 = TCPROS/XMLRPC via a master, ROS2 = DDS). They do **not** automatically see each other's topics — for that you need the **ros1_bridge** (Section 6).

---

## 2. Docker Command Cheat Sheet

### Images
```bash
docker images                          # list local images
docker pull osrf/ros:humble-desktop    # pull ROS2 Humble desktop image
docker rmi <image_id>                  # remove an image
docker build -t my-ros2-ws .           # build image from a Dockerfile in current dir
```

### Containers
```bash
docker ps                              # list running containers
docker ps -a                           # list all containers (incl. stopped)
docker start <container_name>          # start a stopped container
docker stop <container_name>           # stop a running container
docker rm <container_name>             # remove a container
docker exec -it <container_name> bash  # open a new shell in a running container
docker attach <container_name>         # attach to container's main process
docker logs -f <container_name>        # follow container logs
docker cp file.txt <container>:/path   # copy file into container
docker cp <container>:/path file.txt   # copy file out of container
```

### Cleanup
```bash
docker system df                       # see disk usage
docker system prune                    # remove unused containers/networks/images
docker volume prune                    # remove unused volumes
```

---

## 3. Running the ROS2 Humble Container

### Basic run (quick test, disposable)
```bash
docker run -it --rm \
  --network host \
  osrf/ros:humble-desktop \
  bash
```
- `-it` → interactive terminal
- `--rm` → auto-remove container on exit (good for throwaway testing)
- `--network host` → shares host's network stack, so ROS2 nodes in the container can be reached at the same IP as the host (needed for talking to ROS1 side / other machines)

### Persistent, workspace-mounted run (recommended for real work)
```bash
docker run -it \
  --name ros2_humble \
  --network host \
  --ipc host \
  -v ~/ros2_ws:/root/ros2_ws \
  -v /dev/shm:/dev/shm \
  -e ROS_DOMAIN_ID=0 \
  osrf/ros:humble-desktop \
  bash
```
- `--name ros2_humble` → so you can `docker start`/`exec` into it later instead of recreating it
- `-v ~/ros2_ws:/root/ros2_ws` → mounts your workspace folder from host into container (edits persist, survive container removal)
- `--ipc host` → needed for some RViz/Gazebo shared-memory use cases
- `-e ROS_DOMAIN_ID=0` → keep this consistent across anything that needs to discover the same DDS domain

### With GUI support (RViz2, rqt, Gazebo)
On the **host**, first allow the container to use your X server:
```bash
xhost +local:docker
```
Then run with the display forwarded:
```bash
docker run -it \
  --name ros2_humble \
  --network host \
  --ipc host \
  -v ~/ros2_ws:/root/ros2_ws \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  osrf/ros:humble-desktop \
  bash
```
Revoke access when done (optional, more secure):
```bash
xhost -local:docker
```

### With GPU (for Gazebo/RViz with NVIDIA GPU)
```bash
docker run -it \
  --name ros2_humble \
  --network host \
  --gpus all \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v ~/ros2_ws:/root/ros2_ws \
  osrf/ros:humble-desktop \
  bash
```
Requires the NVIDIA Container Toolkit installed on the host.

### Re-entering a container you already created
```bash
docker start ros2_humble
docker exec -it ros2_humble bash
```

---

## 4. Setting Up the ROS2 Workspace (inside the container)

```bash
# Source the ROS2 underlay first
source /opt/ros/humble/setup.bash

# Create workspace structure (if not already mounted from host)
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src

# Clone or copy your packages here
git clone <your_ros2_package_repo>

# Install dependencies (from workspace root)
cd ~/ros2_ws
rosdep update
rosdep install --from-paths src --ignore-src -r -y

# Build
colcon build --symlink-install

# Source the overlay
source install/setup.bash
```

Add both source lines to `~/.bashrc` inside the container so new shells (`docker exec`) get them automatically:
```bash
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
```

### Running nodes
```bash
ros2 run <package_name> <executable_name>
ros2 launch <package_name> <launch_file>.launch.py
ros2 topic list
ros2 node list
```

---

## 5. Optional: A Dockerfile So You Don't Redo Setup Every Time

```dockerfile
FROM osrf/ros:humble-desktop

RUN apt-get update && apt-get install -y \
    python3-colcon-common-extensions \
    python3-rosdep \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /root/ros2_ws

# Copy your source code in (or mount it as a volume instead)
COPY src/ ./src/

RUN . /opt/ros/humble/setup.sh && \
    rosdep update && \
    rosdep install --from-paths src --ignore-src -r -y && \
    colcon build --symlink-install

RUN echo "source /opt/ros/humble/setup.bash" >> /root/.bashrc && \
    echo "source /root/ros2_ws/install/setup.bash" >> /root/.bashrc

CMD ["bash"]
```
Build and run:
```bash
docker build -t my-ros2-ws .
docker run -it --network host --name my_ros2_container my-ros2-ws
```

---

## 6. Bridging ROS1 (host) ↔ ROS2 (container)

Since your host runs ROS1, to exchange topics/services between the two, use **ros1_bridge**. Easiest path: run it in a *third* container that has both ROS1 and ROS2 installed, or install it on the host if you have ROS2 there too. The common pattern:

### Option A — dedicated bridge image (recommended)
Pull an image that has both ROS1 Noetic and ROS2 Humble, e.g. `ros:noetic-ros-base` won't have ROS2 — instead use a bridge-specific image like `osrf/ros:noetic-desktop-full` isn't sufficient either. The reliable approach:

1. Make sure `roscore` is running on the **host** (native ROS1).
2. Run the bridge container with host networking so it can reach the host's `roscore`:
```bash
docker run -it --rm \
  --network host \
  -e ROS_MASTER_URI=http://localhost:11311 \
  ros:noetic-humble-bridge \
  ros2 run ros1_bridge dynamic_bridge
```
(If a pre-built dual image isn't available, build one from a Dockerfile that installs both `ros-noetic-ros-base` and `ros-humble-ros-base` plus `ros1_bridge` — this is the standard "dual ROS" bridge image pattern used in industry.)

### Option B — build ros1_bridge yourself in the ROS2 container
Only works if ROS1 is also reachable/installed where the bridge runs:
```bash
apt-get update && apt-get install -y ros-humble-ros1-bridge
source /opt/ros/humble/setup.bash
ros2 run ros1_bridge dynamic_bridge --bridge-all-topics
```

### Key requirement either way
- `ROS_MASTER_URI` must point to your host's running `roscore` (usually `http://localhost:11311` since you're using `--network host`).
- The bridge auto-detects topics with matching types on both sides and relays messages between them.

---

## 7. Quick Troubleshooting

| Issue | Fix |
|---|---|
| `ros2 topic list` shows nothing from another machine/container | Check `ROS_DOMAIN_ID` matches on all sides; confirm `--network host` used |
| GUI apps (RViz2) won't open | Re-check `xhost +local:docker` and `DISPLAY`/X11 volume mount |
| `rosdep` fails with "cannot find rosdep" | Run `rosdep update` before `rosdep install` |
| Bridge sees no ROS1 topics | Confirm `roscore` is actually running on host and `ROS_MASTER_URI` is correct inside bridge container |
| Container loses workspace changes after removal | Always use `-v` to bind-mount your workspace, don't rely on container's writable layer |

---

## 8. Handy One-Liners Summary

```bash
# Pull image
docker pull osrf/ros:humble-desktop

# Run with workspace + networking + GUI
xhost +local:docker
docker run -it --name ros2_humble --network host --ipc host \
  -v ~/ros2_ws:/root/ros2_ws -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix osrf/ros:humble-desktop bash

# Re-enter later
docker start ros2_humble && docker exec -it ros2_humble bash

# Inside container: build workspace
cd ~/ros2_ws && colcon build --symlink-install && source install/setup.bash
```
