# arduinobot_cpp_examples

C++ (`rclcpp`) versions of the examples in [`arduinobot_py_examples`](../arduinobot_py_examples/README.md).

> **Status: work in progress.** The package skeleton exists (`CMakeLists.txt`, `package.xml`, `src/`, `include/`), but no nodes have been written yet.

## Planned nodes

- [ ] `simple_publisher`: publish `std_msgs/msg/String` on `/chatter` with a timer
- [ ] `simple_subscriber`: subscribe to `/chatter` and log each message
- [ ] `simple_parameter`: declare parameters and check changes with a callback

## Build

```bash
cd ~/arduinobot_ws
colcon build --packages-select arduinobot_cpp_examples
source install/setup.bash
```

## Adding a node (reminder)

1. Put the source file in `src/`, e.g. `src/simple_publisher.cpp`.
2. In `package.xml` add `<depend>rclcpp</depend>` and `<depend>std_msgs</depend>`.
3. In `CMakeLists.txt`:
   ```cmake
   find_package(rclcpp REQUIRED)
   find_package(std_msgs REQUIRED)

   add_executable(simple_publisher src/simple_publisher.cpp)
   ament_target_dependencies(simple_publisher rclcpp std_msgs)

   install(TARGETS simple_publisher DESTINATION lib/${PROJECT_NAME})
   ```
4. Rebuild, then run it with `ros2 run arduinobot_cpp_examples simple_publisher`.
