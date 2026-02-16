# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Docker development environment for the **Mulinex** omnidirectional robot, targeting an Intel NUC running ROS 2 Humble. The container is based on `osrf/ros:humble-desktop-full` and provides a full ROS 2 workspace with GUI support (X11 forwarding), NVIDIA GPU passthrough, and tools like PlotJuggler, Gazebo (Ignition Fortress), and rviz2.

## Build and Run

```bash
# Build the Docker image (pulls base image, creates src/build/log/install dirs)
./docker/build.bash

# Rebuild without cache (to update base image and packages)
./docker/build.bash -r

# Run a container (interactive shell, auto-removed on exit)
./docker/run.bash
```

The build script creates host directories (`build/`, `log/`, `src/`, `install/`) and maps the host user UID/GID into the container. The `run.bash` script sets up X11 auth, mounts the workspace, and enables shared memory for ROS 2 DDS communication.

## Architecture

### Docker (`docker/`)
- **DockerFile** - Multi-stage build from `osrf/ros:humble-desktop-full`. Creates a non-root user matching the host UID/GID. Installs ROS 2 packages (ros2_control, pinocchio, CycloneDDS, PlotJuggler, Gazebo), plus system tools.
- **compose.yaml** - Docker Compose config. Uses host networking (`network_mode: host`), ROS_DOMAIN_ID=10, CycloneDDS as RMW. Mounts `./` into `/home/ros/mulinex_ws/`. Also defines macvlan networks for cabled (100.100.100.0/24) and WiFi (192.168.88.0/24) connections (currently commented out).
- **build.bash / run.bash** - Build and run helpers. The image is `mulinex:humble`, container name is `mulinex`.
- **.config/** - Container entrypoint script and bashrc setup that sources ROS 2 and the workspace overlay.

### ROS 2 Messages (`pi3hat_moteus_int_msgs/`)
An ament_cmake package defining custom ROS 2 message types for the pi3hat/moteus motor control interface:
- **JointsCommand.msg** - Joint position/velocity/effort commands with kp/kd scale factors
- **JointsStates.msg** - Joint feedback (position, velocity, effort, current, temperature, secondary encoder)
- **OmniMulinexCommand.msg** - Base velocity command (v_x, v_y, omega, height_rate)
- **PacketPass.msg** - Communication quality metrics (packet loss, cycle duration)
- **DistributorsState.msg** - Power distributor state (current, voltage, temperature)

Build inside the container with: `colcon build --packages-select pi3hat_moteus_int_msgs`

### Network Configuration (`net_conf/`)
Netplan YAML configs for the NUC's networking:
- **wifi_conf.yaml** - Sets up a WiFi access point ("Mulinex_3" on 5GHz) for direct PC-to-NUC connection, with static wired IP (100.100.100.3/24)
- **wired_conf.yaml** - Standard config with WiFi client mode (DHCP) for internet access, plus static wired IP

Apply with `sudo cp net_conf/<file> /etc/netplan/<target> && sudo netplan apply --debug` (see README for exact paths).

## Key Configuration Details

- **ROS_DOMAIN_ID**: 10
- **RMW**: CycloneDDS (`rmw_cyclonedds_cpp`)
- **Workspace path** (inside container): `/home/ros/mulinex_ws`
- **Source mount**: host `./` maps to container `/home/ros/mulinex_ws/` (compose) or `${PWD}:${PWD}` (run.bash)
- **NUC wired static IP**: 100.100.100.3/24
