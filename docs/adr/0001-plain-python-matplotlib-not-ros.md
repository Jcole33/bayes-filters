# Plain Python and Matplotlib, not ROS2 or Gazebo

Status: accepted (provisional, pending the stack comparison in requirements A1)

The current plan is a single-process Python program: NumPy for the simulation, Matplotlib for the GUI, and no ROS2, turtlesim or Gazebo. ROS2 was the user's first thought, but it costs a lot here for little gain.

- **Setup:** ROS2 needs a ROS install and a colcon workspace, and every filter has to be wrapped in node boilerplate.
- **Determinism:** seeded runs can be made repeatable in ROS2, using a single-threaded executor, synchronous calls or bag replay, but only with machinery a plain loop doesn't need.
- **Belief drawing:** turtlesim can't draw beliefs at all. RViz can (occupancy grids, PoseArray, pose with covariance, MarkerArray), but each belief type and every plot needs its own markers or messages.
- **What ROS is for:** real robots and sensor drivers, neither of which matters for learning filters.

The next stage compares stacks in full and may replace this choice if an alternative is clearly better on setup, belief visualization, ease of plugging in a filter and repeatability. The comparison is not settled in advance.
