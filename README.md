# Movel AI – Control Systems Assignment

This repository contains the solutions for **PART B** (Programming Test) and **PART C** (Algorithmic Test) of the Movel AI Control Systems Assignment, as well as a video on the Seirios RNS installation test.

The setup strictly follows the assignment footnote:

> **Simulation/rosbag runs on the host**, while the **navigation program runs inside Docker**.

I provide two components:

- **`movel_waypoints_autorun/`** — Dockerized waypoint follower (runs in the container).
- **`host_sim_minimal/`** — Minimal host-side simulation that provides `/map`, `/odom`, `/tf` and consumes `/cmd_vel`.

You can also distribute these as two archives:
`movel_waypoints_autorun` and `host_sim_minimal`.

The specific code should be downloaded via a Google Drive link.
---

## 1) Repository Layout

```
.
├── movel_waypoints_autorun/         # Dockerized waypoint follower (ROS Noetic)
│   ├── docker/                      # Dockerfile + entrypoint (run_inside.sh)
│   ├── docker-compose.yml           # Uses host network; auto-waits for host roscore
│   ├── src/waypoint_follower/
│   │   ├── config/                  # move_base & costmap configs + sample waypoints
│   │   ├── launch/                  # auto-run launch files
│   │   └── scripts/
│   │       ├── waypoint_follower.py # PART B: sends goals in sequence
│   │       └── simplify_path.py     # PART C: path simplification to N points
│── video  
│
├── host_sim_minimal/                # Host-side minimal sim
│   ├── maps/mini_map.pgm/yaml       # Small static map
│   ├── scripts/unicycle_sim.py      # Integrates /cmd_vel → publishes /odom + /tf
│   ├── launch/minimal_sim.launch    # Map server + static TF + unicycle sim
│   └── rviz/sim.rviz                # RViz layout 
│
├── data/
│   └── path_test.bag                # Example bag
└── README.md
```

---

## 2) Prerequisites

### Host (Ubuntu 20.04 + ROS Noetic)
- ROS master on host (`roscore`)
- `rviz`, `map_server`
- Docker Engine + Docker Compose
- The container connects to the host’s ROS master (compose file uses **host network**).  

### Container (ROS Noetic)
Image based on `ros:noetic-ros-base-focal`, includes:
- `ros-noetic-navigation`, `ros-noetic-map-server`, `ros-noetic-amcl`
- `ros-noetic-dwa-local-planner` and `ros-noetic-teb-local-planner`
- PART C deps: `python3-rosbag`, `ros-noetic-rosbag`, `python3-rospy`, `python3-roslib`, `python3-genpy`, `python3-genmsg`

The entrypoint waits for the host ROS master to be available before launching `move_base` and the follower.

---

## 3) Start

Open two terminals.

### (A) Host: start the minimal simulation
```bash
unzip host_sim_minimal.zip -d ~/host_ws/src/
cd ~/host_ws
catkin_make
source devel/setup.bash
roslaunch host_sim_minimal host_sim.launch
```
This publishes `/map`, `/odom`, `/tf`. It also integrates `/cmd_vel` (from the container) into a unicycle model to move the robot.

### (B) Container: build & run the waypoint follower
```bash
unzip movel_waypoints_autorun.zip -d ~/projects/
cd ~/projects/movel_waypoints_autorun
sudo docker compose build
sudo docker compose up
```
The container will wait for the host’s ROS master and then launch `move_base` + the waypoint follower.  

---

## 4) PART B — Waypoint Following (in Docker)

### What it does
- Implements a waypoint-follower node that drives a robot along a predefined route.
- Supports different local planners. Local planner: **DWA** (default) or **TEB** (optional).
- `waypoint_follower.py` reads a YAML of waypoints and sends goals to `/move_base` in order.
- Runs inside a Docker container with auto-start enabled.

### Select the waypoint file
By default the container looks for a waypoints YAML in `src/waypoint_follower/config/`.  
You can override with an environment variable:

```bash
# Use a file inside the container (for example), or another motion path file.    
sudo PLANNER=teb \
     WAYPOINT_FILE=/root/ws/src/waypoint_follower/config/waypoints.yaml \
     docker compose up   
```

### Switch local planner
- **DWA (default)** — configure in `config/dwa_local_planner.yaml` and `move_base_common.yaml`.
- **TEB** — enable via environment/launch arg (e.g., `use_teb=true`) and provide `teb_local_planner.yaml`.

---

## 5) PART C — Path Simplification from ROS Bag (in Docker)

The provided bag file contains ~1000 raw path points (geometry_msgs/PoseStamped).
This script simplifies the path to a manageable number of points (default: 200) while:
- Preserving the overall trajectory shape.
- Smoothing local noise.
- Allowing adjustable number of output points (--n).

### Algorithm (inside `simplify_path.py`)
1. **Smooth**: Moving average (`--smooth avg:W`).
2. **Simplify**: Douglas–Peucker (RDP) to drop redundant bends.
3. **Resample**: Equal arc-length to get an exact `N`.
4. **Export**: YAML compatible with the waypoint follower.

### Commands

**Copy the bag into the container:**
```bash
sudo docker compose up -d
sudo docker cp data/path_test.bag movel_waypoints:/root/in.bag
```

**Run the script in the container:**
```bash
sudo docker exec -it movel_waypoints bash -lc \
 'source /opt/ros/noetic/setup.bash && \
  python3 /root/ws/src/waypoint_follower/scripts/simplify_path.py \
    --bag /root/in.bag --topic /vslam2d_pose --n 200 --smooth avg:9 --out /root/out.yaml'
```

**Copy the result back to host:**
```bash
sudo docker cp movel_waypoints:/root/out.yaml ./src/waypoint_follower/config/simplified_waypoints.yaml
```

**Run the simplified waypoints:**
```bash
# Option A: point the container to /root/out.yaml directly
sudo WAYPOINT_FILE=/root/out.yaml docker compose up

# Option B: copy to the package config and start normally
sudo PLANNER=dwa \
     WAYPOINT_FILE=/root/ws/src/waypoint_follower/config/simplified_waypoints.yaml \
     docker compose up

sudo PLANNER=teb \
     WAYPOINT_FILE=/root/ws/src/waypoint_follower/config/simplified_waypoints.yaml \
     docker compose up
```

---

## 6) RViz – Useful Displays

A ready-to-use layout is provided at `host_sim_minimal/rviz/sim.rviz`.

---

## 7) Demo Video

- **PART B – Waypoint Following Demo**  
  <video src="video/partb.webm" controls width="600"></video>

- **PART C – Path Simplification Demo**  
  <video src="video/partc.webm" controls width="600"></video>

- **Seirios RNS Installation Test**  
  **Driving Test**  
  <video src="video/drive.mp4" controls width="600"></video>  

  **Trails Test**  
  <video src="video/trails.mp4" controls width="600"></video>  

  **Waypoints Test**  
  <video src="video/waypoints.mp4" controls width="600"></video>  

---

## 8) Submission Checklist

-  Code builds and runs **inside Docker** (ROS Noetic)
-  Host provides `/map`, `/odom`, `/tf`
-  Waypoint follower runs and reaches goals (PART B)
-  Path simplification script with exact-N output (PART C)
-  README covers steps, tuning, and pitfalls
-  Videos prepared

---

## 9) License

This assignment solution is provided for educational and evaluation purposes. Please include attribution if you reuse or extend it.
