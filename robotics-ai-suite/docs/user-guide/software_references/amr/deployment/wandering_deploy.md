# Deploying `wandering`

This software reference deploys the `wandering` workflow on a Clearpath Jackal with an
Intel® RealSense™ camera. The application uses RTAB-Map visual SLAM, Nav2 navigation,
and ADBSCAN 3D obstacle perception to explore and map the environment autonomously.

## Architecture

The deployment pipeline integrates Clearpath Jackal base services, Intel® RealSense™
depth sensing, RTAB-Map visual SLAM, Nav2 navigation with the ADBSCAN costmap layer,
and autonomous frontier exploration.

```{mermaid}
flowchart TD
    subgraph Sensors["Hardware Sensors & Base"]
        RS["Intel® RealSense™ Camera\n(RGB-D & PointCloud2)"]
        Depth2Scan["depthimage_to_laserscan\n(Surrogate /scan)"]
        JackalBase["Clearpath Jackal Base Driver\n(<robot_namespace>/cmd_vel)"]
        RS --> Depth2Scan
    end

    subgraph Perception["Perception & SLAM"]
        Depth2Scan -->|"/scan"| SLAM["RTAB-Map SLAM\n(Visual SLAM & TF)"]
        RS -->|"RGB-D Streams"| SLAM
        RS -->|"PointCloud2"| Fusion["adbscan_sensor_fusion\n(Sensor Fusion & Downsampling)"]
        Depth2Scan -->|"/scan"| Fusion
        Fusion -->|"/adbscan/points"| ADBSCAN["ADBSCAN Node\n(3D Clustering)"]
    end

    subgraph Nav["Nav2 Navigation Stack"]
        ADBSCAN -->|"/obstacle_array"| Costmaps["Costmaps\n(ADBScanLayer + Standard Layers)"]
        Depth2Scan -->|"/scan"| Costmaps
        SLAM -->|"/map & TF"| Nav2["Nav2 Planner & Controller\n(NavigateToPose Server)"]
        Costmaps --> Nav2
    end

    subgraph App["Wandering Application & UI"]
        Costmaps -->|"/global_costmap/costmap"| WanderApp["wandering_app\n(Frontier Exploration)"]
        RVizPanel["RViz Wandering Control Panel\n(Manual / Autonomous Mode)"]
        RVizPanel -->|"/pause_wandering"| WanderApp
        WanderApp -->|"NavigateToPose Goal"| Nav2
    end

    Nav2 --> JackalBase
```

## Components

- `depthimage_to_laserscan` derives a 2D `/scan` topic from the RealSense depth image.
- `adbscan_sensor_fusion` time-synchronizes and filters 2D `/scan` and RealSense 3D point clouds (or Velodyne Puck LiDAR).
- `adbscan_ros2` identifies and clusters 3D obstacle objects from the fused point cloud.
- `nav2_adbscan_layer` marks object-sized ADBSCAN obstacle clusters into Nav2 local and global costmaps.
- RTAB-Map creates and updates the 3D environment map and robot localization.
- Nav2 plans and executes movement toward exploration goals using enhanced costmaps.
- `wandering_app` (`WanderingMapper` and `GoalCatcher`) selects unvisited frontiers and dispatches `NavigateToPose` goals.
- `wandering_rviz_panel` allows operators to toggle between autonomous wandering and manual navigation.

## Prerequisites

Complete the [Clearpath Robotics Jackal setup](../../../hardware_blueprints/amr/clearpath-jackal.md)
and verify motor control and sensor topics before continuing.

## Install the Deployment Package

<!--hide_directive::::{tab-set}hide_directive-->
<!--hide_directive:::{tab-item}hide_directive--> **Jazzy**
<!--hide_directive:sync: jazzyhide_directive-->

```bash
sudo apt update
sudo apt install ros-jazzy-wandering
```

<!--hide_directive:::hide_directive-->
<!--hide_directive:::{tab-item}hide_directive--> **Humble**
<!--hide_directive:sync: humblehide_directive-->

```bash
sudo apt update
sudo apt install ros-humble-wandering
```

<!--hide_directive:::hide_directive-->
<!--hide_directive::::hide_directive-->

## Run the Deployment

### Autonomous exploration mode

Log in as the `administrator` user (or user with robot access permissions) and run:

<!--hide_directive::::{tab-set}hide_directive-->
<!--hide_directive:::{tab-item}hide_directive--> **Jazzy**
<!--hide_directive:sync: jazzyhide_directive-->

```bash
export ROBOT_NAMESPACE=/j100_0812
ros2 launch wandering_bringup wandering_jackal.launch.py
```

<!--hide_directive:::hide_directive-->
<!--hide_directive:::{tab-item}hide_directive--> **Humble**
<!--hide_directive:sync: humblehide_directive-->

```bash
export ROBOT_NAMESPACE=/j100_0812
ros2 launch wandering_bringup wandering_jackal.launch.py
```

<!--hide_directive:::hide_directive-->
<!--hide_directive::::hide_directive-->

After startup, the robot begins autonomous exploration and RTAB-Map builds a map from the
RealSense camera input while Nav2 avoids obstacles using the ADBSCAN perception layer.
Stop the workflow with `Ctrl-c`.

### Interactive manual override mode

To retain autonomous SLAM and ADBSCAN costmap protection while allowing an operator to pause exploration and send manual navigation goals via RViz:

```bash
export ROBOT_NAMESPACE=/j100_0812
ros2 launch wandering_bringup wandering_jackal_manual_nav.launch.py
```

- Click **Manual mode** in the **Wandering Control** panel to cancel the active exploration goal.
- Use Nav2's **Goal** tool in the RViz toolbar to click-drag a waypoint on the map.
- Click **Autonomous mode** to resume autonomous exploration.

## Troubleshooting

For general robot issues, see the [troubleshooting guide](../../../resources/troubleshooting).