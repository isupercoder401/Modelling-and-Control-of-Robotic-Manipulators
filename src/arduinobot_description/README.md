# arduinobot_description

The robot model of **Arduinobot**, a robotic arm with a rotating base, two arm joints and a two-finger gripper. It is described in URDF/Xacro with STL meshes. One launch file shows it in RViz, and another simulates it in Gazebo.

## Contents

```
arduinobot_description/
├── urdf/arduinobot.urdf.xacro   # robot model (links, joints, limits, collision, inertia)
├── meshes/*.STL                 # 3D parts used for visual and collision geometry
├── launch/display.launch.py     # starts robot_state_publisher + joint sliders + RViz
├── launch/gazebo.launch.py      # Gazebo simulation
└── rviz/display.rviz            # saved RViz configuration
```

The `CMakeLists.txt` installs all four folders to `share/arduinobot_description` so the launch file can find them with `get_package_share_directory()`.

## Quick start

```bash
cd ~/arduinobot_ws
colcon build --packages-select arduinobot_description
source install/setup.bash
ros2 launch arduinobot_description display.launch.py
```

RViz opens with the arm and a **Joint State Publisher** window. Move the sliders to move the joints.

## The robot model

The URDF is written with **Xacro** (XML macros), which lets the file use variables:

| Xacro property | Value | Used for |
|---|---|---|
| `PI` | `3.14159` | joint limits and rotations |
| `effort` | `30.0` | max joint effort |
| `velocity` | `10.0` | max joint velocity |

The meshes are scaled by `0.01` in each axis.

### Visual, collision and inertial

Every link with geometry has three parts:

| Tag | Used by | In this model |
|---|---|---|
| `<visual>` | RViz, Gazebo (what you **see**) | STL mesh of the part |
| `<collision>` | Gazebo, MoveIt (what can **hit** things) | An exact copy of the visual: the same mesh, origin and scale |
| `<inertial>` | Gazebo physics (how the link **moves**) | Created by the `default_inertial` macro |

The `default_inertial` macro takes the link's mass in **kilograms**:

```xml
<xacro:macro name="default_inertial" params="mass">
    <inertial>
        <mass value="${mass}"/>
        <inertia ixx="1.0" ixy="0.0" ixz="0.0" iyy="1.0" iyz="0.0" izz="1.0"/>
    </inertial>
</xacro:macro>

<xacro:default_inertial mass="0.1"/>   <!-- used inside a link -->
```

| Link | Mass (kg) |
|---|---|
| `base_link` | 1.0 |
| `base_plate`, `forward_drive_arm`, `horizontal_arm` | 0.1 |
| `claw_support` | 0.05 |
| `gripper_right`, `gripper_left`, `rgb_camera` | 0.01 |

> The inertia values (`1.0` kg·m²) are placeholders. Real values for parts this small would be much smaller. RViz ignores inertia; only the physics simulation uses it.

To see the collision shapes in RViz, open **RobotModel** and tick **Collision Enabled**.

### Joints

| Joint | Type | Parent → Child | Axis | Limits (rad) |
|---|---|---|---|---|
| `virtual_joint` | fixed | `world` → `base_link` | – | – |
| `joint_1` | revolute | `base_link` → `base_plate` | z | −π/2 … π/2 (base rotation) |
| `joint_2` | revolute | `base_plate` → `forward_drive_arm` | x | −π/2 … π/2 (shoulder) |
| `joint_3` | revolute | `forward_drive_arm` → `horizontal_arm` | x | −π/2 … π/2 (elbow) |
| `horizontal_arm_to_claw_support` | fixed | `horizontal_arm` → `claw_support` | – | – |
| `joint_4` | revolute | `claw_support` → `gripper_right` | z | −π/2 … 0 (gripper) |
| `joint_5` | revolute (**mimic**) | `claw_support` → `gripper_left` | z | 0 … π/2, follows `joint_4 × −1` |
| `rgb_camera_joint` | fixed | `base_link` → `rgb_camera` | – | – |

Because of `<mimic>`, the two gripper fingers open and close together: you move `joint_4` and `joint_5` mirrors it.

### Link tree

```
world
└── base_link
    ├── base_plate                 (joint_1)
    │   └── forward_drive_arm      (joint_2)
    │       └── horizontal_arm     (joint_3)
    │           └── claw_support   (fixed)
    │               ├── gripper_right  (joint_4)
    │               └── gripper_left   (joint_5, mimic)
    └── rgb_camera                 (fixed)
```

Check the model from the terminal:

```bash
xacro src/arduinobot_description/urdf/arduinobot.urdf.xacro > /tmp/arduinobot.urdf
check_urdf /tmp/arduinobot.urdf          # from the liburdfdom-tools package
```

## How the launch file works

[`display.launch.py`](launch/display.launch.py) starts three nodes:

```
            robot_description (parameter: xacro → URDF string)
                         │
joint_state_publisher_gui ──/joint_states──▶ robot_state_publisher ──/tf, /tf_static──▶ rviz2
        (sliders)                                                   /robot_description ──▶ (RobotModel display)
```

1. **`model` launch argument**: the path to the Xacro file. By default it's the installed `urdf/arduinobot.urdf.xacro`.
2. **`robot_description`**: `Command(['xacro ', LaunchConfiguration('model')])` runs xacro when the launch starts. Wrapping it in `ParameterValue(..., value_type=str)` makes sure ROS treats the output as plain text.
3. **`robot_state_publisher`**: reads the URDF and the joint angles, then publishes the pose of every link on TF.
4. **`joint_state_publisher_gui`**: shows a slider for each movable joint and publishes `/joint_states`.
5. **`rviz2`**: opens with the saved config in `rviz/display.rviz` (`-d` argument).

Use a different model file:

```bash
ros2 launch arduinobot_description display.launch.py model:=/absolute/path/to/robot.urdf.xacro
```

Inspect what's running:

```bash
ros2 topic echo /joint_states
ros2 run tf2_tools view_frames      # saves frames.pdf with the TF tree
```

## Gazebo simulation

[`gazebo.launch.py`](launch/gazebo.launch.py) will run the robot in **Gazebo Harmonic**, the simulator used with ROS 2 Jazzy. It talks to ROS through the `ros_gz` packages.

> **Status:** working. Gazebo starts, and the robot spawns in an empty world. The joints aren't controlled yet, so the arm falls under gravity.

### What each part does

| Action | Package / executable | Purpose |
|---|---|---|
| `model` argument | – | Path to the Xacro file (the same default as `display.launch.py`) |
| `SetEnvironmentVariable` | – | Adds the parent of the package's `share/` folder to Gazebo's resource path, so `package://arduinobot_description/meshes/...` can be found |
| `robot_state_publisher` | `robot_state_publisher` | Publishes `/robot_description` and TF. `use_sim_time: True` makes it follow Gazebo's clock |
| `gazebo` (included launch) | `ros_gz_sim` → `gz_sim.launch.py` | Starts Gazebo with `gz_args: -v 4 -r empty.sdf` (verbose logs, start running, empty world) |
| `gz_spwan_entity` | `ros_gz_sim` → `create` | Spawns the robot in Gazebo from the `robot_description` topic, with the name `arduinobot` |
| `gz_ros2_bridge` | `ros_gz_bridge` → `parameter_bridge` | Bridges Gazebo's `/clock` to ROS (`rosgraph_msgs/msg/Clock`) so ROS nodes can use simulation time |

```
                    robot_description
robot_state_publisher ───────────────▶ create (spawns the robot) ──▶ Gazebo
          ▲                                                           │
          └────────────── /clock ◀── parameter_bridge ◀───────────────┘
```

### Ways of building paths used in this file

| Code | Result |
|---|---|
| `get_package_share_directory('arduinobot_description')` | `install/arduinobot_description/share/arduinobot_description` |
| `os.path.join(dir, "urdf", "arduinobot.urdf.xacro")` | A file inside that folder |
| `Path(dir).parent.resolve()` | The folder one level up (`.../share`), as an absolute path |
| `[path, "/file.launch.py"]` (a list) | Joined together by the launch system when the launch starts. A list can also contain values like `LaunchConfiguration` |

### Fixes made to get it running

- [x] Return the actions: `LaunchDescription([model_Arg, gazebo_resource_path, robot_state_publisher, gazebo, gz_spwan_entity, gz_ros2_bridge])`
- [x] Fix the included file path: `os.path.join(get_package_share_directory('ros_gz_sim'), 'launch', 'gz_sim.launch.py')`
- [x] Rename the environment variable to `GZ_SIM_RESOURCE_PATH`, which is what Gazebo Harmonic reads
- [x] Fix the spawn arguments to `["-topic", "robot_description", "-name", "arduinobot"]` (no trailing space after `-topic`, and `-name` needs its dash)

Run it with:

```bash
ros2 launch arduinobot_description gazebo.launch.py
```

## Dependencies

Declared as `<exec_depend>` in [`package.xml`](package.xml):

| Package | Used for |
|---|---|
| `xacro` | Turning the `.urdf.xacro` file into URDF |
| `urdf` | URDF parsing |
| `robot_state_publisher` | Publishing the robot description and TF |
| `joint_state_publisher_gui` | Joint sliders in `display.launch.py` |
| `rviz2` | Visualisation |
| `ros2launch` | `ros2 launch` |
| `ros_gz_sim` | Starting Gazebo and spawning the robot |
| `ros_gz_bridge` | Bridging topics between Gazebo and ROS |
