# arduinobot_description

The robot model of **Arduinobot**, a robotic arm with a rotating base, two arm joints and a two-finger gripper. It is described in URDF/Xacro with STL meshes. A launch file shows it in RViz.

## Contents

```
arduinobot_description/
├── urdf/arduinobot.urdf.xacro   # robot model (links, joints, limits)
├── meshes/*.STL                 # 3D parts used as visual geometry
├── launch/display.launch.py     # starts robot_state_publisher + joint sliders + RViz
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

## Dependencies

At runtime this package uses `xacro`, `robot_state_publisher`, `joint_state_publisher_gui`, `rviz2` and `ros2launch`.
