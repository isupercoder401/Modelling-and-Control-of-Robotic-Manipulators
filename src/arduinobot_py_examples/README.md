# arduinobot_py_examples

Simple ROS 2 nodes written in Python (`rclpy`). They cover the core ideas: **nodes**, **topics**, **timers** and **parameters**.

## Contents

| Executable | Source file | Concept |
|---|---|---|
| `simple_publisher` | [`simple_publisher.py`](arduinobot_py_examples/simple_publisher.py) | Publishing on a topic with a timer |
| `simple_subscriber` | [`simple_subscriber.py`](arduinobot_py_examples/simple_subscriber.py) | Subscribing to a topic with a callback |
| `simple_parameter` | [`simple_parameters.py`](arduinobot_py_examples/simple_parameters.py) | Declaring parameters and checking changes made at runtime |

The executable names come from `entry_points` in [`setup.py`](setup.py).

## Build

```bash
cd ~/arduinobot_ws
colcon build --packages-select arduinobot_py_examples
source install/setup.bash
```

> Tip: `colcon build --symlink-install` makes Python edits take effect without rebuilding.

---

## 1. Simple publisher

**Node name:** `simple_publisher`
**Publishes:** `/chatter` (`std_msgs/msg/String`), queue size 10

How it works:
1. The node creates a publisher with `create_publisher(String, "chatter", 10)`.
2. `create_timer(1.0, callback)` calls the callback once every second (1 Hz).
3. Each time the callback runs, it publishes `"Hello Ros2 - ounter <n>"` and adds one to the counter.

```bash
ros2 run arduinobot_py_examples simple_publisher
```

Check it from another terminal:

```bash
ros2 topic echo /chatter
ros2 topic hz /chatter        # should report ~1.0 Hz
```

## 2. Simple subscriber

**Node name:** `simple_subscriber`
**Subscribes:** `/chatter` (`std_msgs/msg/String`), queue size 10

How it works: `create_subscription(String, "chatter", msgCallback, 10)` registers a callback. ROS calls it for every message that arrives, and the callback logs `i heard <message>`.

```bash
ros2 run arduinobot_py_examples simple_subscriber
```

You can test it without the publisher by sending a message by hand:

```bash
ros2 topic pub /chatter std_msgs/msg/String "{data: 'hello from the terminal'}"
```

```
simple_publisher ──/chatter (std_msgs/String)──▶ simple_subscriber
```

## 3. Simple parameters

**Node name:** `simple_parameter`

| Parameter | Type | Default |
|---|---|---|
| `simple_int_param` | integer | `28` |
| `simple_string_param` | string | `"Muhammad"` |

How it works:
1. `declare_parameter(name, default)` declares each parameter. The type comes from the default value.
2. `add_on_set_parameters_callback(...)` registers a callback that runs **before** a change is accepted.
3. The callback returns a `SetParametersResult`. If `successful` is `True`, the change is accepted. Otherwise it's rejected. This node accepts a change only when the value has the right type.

Run with the default values:

```bash
ros2 run arduinobot_py_examples simple_parameter
```

Override values at startup:

```bash
ros2 run arduinobot_py_examples simple_parameter --ros-args \
  -p simple_int_param:=30 -p simple_string_param:="Sohaib"
```

Read and change parameters while the node is running:

```bash
ros2 param list /simple_parameter
ros2 param get  /simple_parameter simple_int_param
ros2 param set  /simple_parameter simple_int_param 42        # accepted, logs the change
ros2 param set  /simple_parameter simple_string_param "ROS"  # accepted
ros2 param set  /simple_parameter simple_int_param "oops"    # rejected: wrong type
```

---

## Dependencies

`rclpy`, `std_msgs`, `rcl_interfaces` (see [`package.xml`](package.xml)).

## Tests

The package includes the standard ament linters (flake8, pep257, copyright):

```bash
colcon test --packages-select arduinobot_py_examples
colcon test-result --verbose
```
