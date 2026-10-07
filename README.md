# ros2-internals-notes

Reverse-engineering how a ROS 2 message actually travels from `publish()` to a
subscriber's callback, by **observing** it rather than trusting the docs. Built
while preparing to work on robot telemetry and operator-command services.

> **Status:** the experiments below are scripts and procedures. The tables marked
> `TODO` are filled in from *my own runs*. Do not trust a number until it has
> your machine, ROS distro and RMW next to it.

## Questions this repo answers

1. What layers does a message cross, and where can I see each one?
2. How does discovery work, and what does it put on the wire?
3. Which QoS combinations silently fail to connect, and how do I detect that?
4. What does the executor do when a callback is slow?
5. Which QoS profile belongs on telemetry versus operator commands?

## The stack (what I traced)

```
your node (rclpy / rclcpp)
   └── rcl            C client library: nodes, publishers, timers, wait sets
        └── rmw       middleware interface: one API, many implementations
             └── rmw_fastrtps_cpp | rmw_cyclonedds_cpp   (chosen by RMW_IMPLEMENTATION)
                  └── DDS / RTPS   discovery + data over UDP
```

Switching RMW needs no code change, only an environment variable:

```bash
RMW_IMPLEMENTATION=rmw_cyclonedds_cpp ros2 run demo_nodes_cpp talker
```

## Setup

```bash
# ROS 2 Humble or Jazzy, Ubuntu 22.04/24.04
source /opt/ros/$ROS_DISTRO/setup.bash
sudo apt install wireshark ros-$ROS_DISTRO-demo-nodes-py
echo $ROS_DOMAIN_ID; ros2 doctor --report | head -40   # record distro + RMW here
```

Record your environment: `distro=__  rmw=__  cpu=__  os=__`

---

## Experiment 1: watch discovery on the wire

```bash
ros2 run demo_nodes_py talker &        # terminal 1
ros2 run demo_nodes_py listener        # terminal 2
sudo wireshark -k -i lo -Y rtps        # or capture on your LAN interface
```

What to look for:
- **SPDP** packets (participant discovery) sent to a multicast group, then **SEDP**
  packets (endpoint discovery) that advertise each topic, type and QoS.
- RTPS port numbers are derived from `ROS_DOMAIN_ID` by formula from the RTPS spec:
  multicast discovery port = `7400 + 250 * domain_id`. Change the domain and
  confirm the port moves.
- Run `ros2 topic info /chatter -v` and match its QoS fields to the SEDP data.

Result: `TODO: screenshot + the ports you observed for domain 0 and domain 5`

## Experiment 2: QoS compatibility matrix

Write a publisher and subscriber pair (`qos_probe.py`) that take their QoS from
CLI flags, then run every combination.

Expected rules to **verify**, not assume:

| Publisher | Subscriber | Expected |
|---|---|---|
| RELIABLE | BEST_EFFORT | connects |
| BEST_EFFORT | RELIABLE | does **not** connect |
| VOLATILE | TRANSIENT_LOCAL | does **not** connect |
| TRANSIENT_LOCAL | VOLATILE | connects |

Detect a mismatch without reading logs by registering an incompatible-QoS event
callback (`QoSIncompatibleQoSEventCallback` in `rclpy.event_handler`) and printing
which policy failed.

Result: `TODO: fill the observed column and note any row that surprised you`

## Experiment 3: executor behaviour with a slow callback

```python
# slow_listener.py: a subscriber whose callback sleeps 500 ms
# publish at 10 Hz and count dropped / delayed messages
```

Run it three ways and compare:
1. `SingleThreadedExecutor` (default `rclpy.spin`)
2. `MultiThreadedExecutor` with all callbacks in one callback group
3. `MultiThreadedExecutor` with the slow callback in its own `ReentrantCallbackGroup`

Measure: messages received per second and worst-case delay of a *second, fast*
subscription in the same node.

Result: `TODO: table of msgs/s and max delay for the three setups`

## Experiment 4: latency vs payload size and RMW

Publish a byte array of 1 KB, 64 KB, 1 MB. Timestamp on publish, diff on receive
(same machine, so the clocks match). Repeat for Fast DDS and Cyclone DDS.

Result: `TODO: median and p99 latency table; one sentence on what you conclude`

---

## Takeaways for a robot control stack

Fill these in from your own results, then keep only what you can defend.

- **Telemetry** (battery, pose at high rate): `qos_profile_sensor_data`
  (best effort, shallow queue). A late sample is worthless; do not retry it.
- **Operator commands / job assignments**: reliable, deeper queue. Losing one
  is worse than delaying it.
- **Latched state** (current mode): transient local, so a late-joining node
  gets the last value.
- A slow callback in a single-threaded executor delays *every* other callback
  in that node: put blocking work in its own callback group or thread.

## Repo layout

```
ros2-internals-notes/
├── README.md
├── qos_probe.py        # Experiment 2
├── slow_listener.py    # Experiment 3
├── latency_bench.py    # Experiment 4
└── captures/           # .pcapng files + screenshots from Experiment 1
```

## References

- ROS 2 docs: *About Quality of Service settings*, *About executors*, *About discovery*
- OMG DDSI-RTPS specification (port mapping formula, SPDP/SEDP)
- `rcl`, `rmw` and `rmw_fastrtps` source on GitHub (read alongside the traces)
