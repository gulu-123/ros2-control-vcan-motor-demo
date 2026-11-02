# Jazzy adaptation progress

Snapshot date: 2026-11-02

This file records verified progress only. Passing a test in this workspace does
not by itself establish full ROS 2 Jazzy support. The project documentation and
CI configuration still identify Humble as the supported distribution.

## Repository state

- Branch: `jazzy-bringup`
- Commit: `d7ffc56` (`fix: keep CAN local loopback enabled on vcan`)
- Initial status for this review: clean and tracking `origin/jazzy-bringup`
- Test-source change reviewed here: fallback for
  `socket.CAN_RAW_ERR_FILTER` in `test/test_virtual_motor_node.py`

## Environment

| Component | Version |
| --- | --- |
| Ubuntu | 24.04.5 LTS (Noble), x86_64 |
| Linux kernel | 7.0.0-38-generic |
| ROS 2 distribution | Jazzy |
| `ros2cli` | 0.32.13 |
| `ros2_control` | 4.48.1 |
| `ros2_controllers` | 4.42.1 |
| `ros2_socketcan` | 1.4.0 |
| `xacro` | 2.1.1 |
| CMake | 3.28.3 |
| GCC/G++ | 13.3.0 |
| Python | 3.12.3 |

## Commands used

Full package build:

```bash
source /opt/ros/jazzy/setup.bash
colcon build --packages-select vcan_diffbot_demo \
  --event-handlers console_direct+
source install/setup.bash
```

Tests that do not require a vcan interface:

```bash
ctest --test-dir build/vcan_diffbot_demo -I 1,10 --output-on-failure
```

Launch tests that create per-process vcan interfaces:

```bash
pkexec --keep-cwd bash -c '
source /opt/ros/jazzy/setup.bash
source install/setup.bash
ctest --test-dir build/vcan_diffbot_demo -I 11,17 --output-on-failure
'
```

The test helper normally uses `sudo -n`. Running the launch tests through
polkit avoids requiring a passwordless sudo rule.

Only the previously failing test was rerun after the Python socket constant
fix:

```bash
pkexec --keep-cwd bash -c '
source /opt/ros/jazzy/setup.bash
source install/setup.bash
ctest --test-dir build/vcan_diffbot_demo \
  -R test_test_virtual_motor_node.py --output-on-failure
'
```

Final result collection:

```bash
source /opt/ros/jazzy/setup.bash
source install/setup.bash
colcon test-result --all --verbose
```

## Build status

The complete `vcan_diffbot_demo` package was built successfully, not only a
subset of its modules. The successful CMake build and install commands returned
exit code 0. The install tree contains the hardware plugin, virtual motor
executable, launch/config/URDF resources, exported headers, plugin description,
and CMake exports.

The build emitted deprecation and test warnings described under
[Remaining issues](#remaining-issues). No compiler errors occurred.

No tracked source relevant to the C++ build changed after the build completed,
so the existing full build remains current. The later Python test-only change
does not require rebuilding the C++ targets.

## Test results

Current aggregate result:

```text
46 tests, 0 errors, 0 failures, 0 skipped
```

| CTest entry | Result | What was verified |
| --- | --- | --- |
| `test_can_protocol` | 7 passed | CAN IDs, little-endian signed velocity, saturation, reserved-byte checks, feedback/ACK round trips, and DLC validation |
| `test_motor_state` | 5 passed | Configuration validation, acceleration limit, encoder integration, disable behavior, watchdog stop, and protocol fault status |
| `test_hardware_health` | 5 passed | Matching/rejected/unexpected ACK handling, feedback age, safe-diagnostic bounds, ACK timeout, and safe-stop ACK handling |
| `test_can_filters` | 2 passed | Hardware and virtual-motor CAN filter acceptance |
| `test_can_error_policy` | 5 passed | SocketCAN error classification, text descriptions, controller state bits, and warning/passive distinction |
| `test_can_sender` | 4 passed | Loopback-capability detection and sender construction across upstream API variants |
| `test_urdf_xacro` | 2 passed | Xacro generation, two-wheel hardware description, and shared CAN configuration |
| `test_demo_launch_arguments` | 1 passed | Virtual motor enable/disable conditions and controller spawn conditions |
| `test_vcan_test_utils` | 3 passed | Unique interface names and root/non-root privileged command construction |
| `test_hardware_plugin` | 4 passed | pluginlib loading, command clearing on failed activation, and deactivation/shutdown errors |
| `test_test_virtual_motor_node.py` | 1 passed | Command ACK, positive feedback, node filtering, watchdog stop, CAN error frame, and malformed feedback |
| `test_test_demo_launch.py` | 1 passed | Normal closed loop from `/cmd_vel` through `ros2_control` and vcan to virtual-motor feedback and odometry |
| `test_test_feedback_timeout_launch.py` | 1 passed | Single-side feedback timeout with node 2 feedback dropped and bounded disabled zero commands |
| `test_test_ack_timeout_launch.py` | 1 passed | Missing command ACK leads to hardware fault and a disabled zero command |
| `test_test_can_busoff_launch.py` | 1 passed | BUS-OFF stops both motors with bounded safe-stop traffic |
| `test_test_invalid_fault_selector_launch.py` | 2 passed | Valid custom node selector and rejection of an invalid selector |
| `test_test_diagnostics_launch.py` | 1 passed | Healthy diagnostics followed by right-motor feedback timeout and `safe_stop` reporting |

No tests were skipped in the current result set.

## Requested stage status

### Complete package build

Complete for the current clean C++ source revision. Both compile and install
returned success, and all configured build targets were created.

### Normal closed loop

Complete as automated launch-test evidence. The test publishes a stamped
forward velocity command, requires both wheel velocities above `0.1 rad/s`,
requires odometry position to move beyond `0.01 m`, then requires both wheel
velocities to fall below `0.05 rad/s` within 3 seconds after commands stop.

Relevant flow:

`geometry_msgs/TwistStamped` -> `DiffDriveController` -> wheel velocity command
interfaces -> `CanMotorHardware::write()` -> vcan -> `VirtualMotorNode` ->
feedback/ACK -> `CanMotorHardware::read()` -> `JointState` and odometry.

### Single-side feedback timeout

Complete as automated launch-test evidence. The feedback-timeout test launches
with `drop_feedback_node_id:=2`, waits for `Feedback timeout for motor node 2`,
and requires two to four disabled zero-velocity command frames without a later
re-enable.

The diagnostics test also verifies:

- bus state becomes `safe_stop`;
- stop reason is `feedback_timeout_node_2`;
- both motor diagnostics become errors;
- the right motor reports `feedback_timeout=true`;
- motor diagnostic fields remain stable.

## Remaining issues

These issues remain even though the current test suite passes:

1. The hardware class still uses deprecated Jazzy initialization/export
   interfaces. Jazzy currently provides backward compatibility, but this
   should be migrated before relying on future releases.
2. `test_hardware_plugin.cpp` ignores the `[[nodiscard]]` result of
   `set_value()` and still uses deprecated `get_value()`.
3. Failure/deactivation launch logs contain controller-manager warnings about
   command/state interfaces not being available and multiple cleanup calls.
   The assertions pass, but lifecycle cleanup behavior needs separate review.
4. README files, badges, installation commands, and CI still target Humble and
   Ubuntu 22.04. They must not be used as evidence of Jazzy support.
5. `vcan0` does not persist across VM restart. Launch tests create isolated
   interfaces and clean them up; manual runs must recreate `vcan0`.
6. No physical SocketCAN hardware/HIL test has been performed.
7. No long-duration stress test, external protocol fuzz test, or manual RViz
   visualization has been recorded.

## Next recommended task

Review and resolve the lifecycle cleanup warnings observed during failed
feedback and deactivation. This is a separate behavior change and should not
be mixed with the completed Jazzy test-constant portability fix.
