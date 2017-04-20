# mav_trajectory_generation
This repository contains tools for polynomial trajectory generation and optimization based on methods described in [1].
These techniques are especially suitable for rotary-wing micro aerial vehicles (MAVs).
This README provides a brief overview of our trajectory generation utilities with some examples.

**Authors**: Markus Achtelik, Michael Burri, Helen Oleynikova, Rik Bähnemann, Marija Popović  
**Maintainer**: Rik Bähnemann, brik@ethz.ch  
**Affiliation**: Autonomous Systems Lab, ETH Zurich

## Basics
A **vertex** describes the properties of a support point of a **polynomial** path. Pairs of vertices are connected together to form **segments**.
Each vertex has a set of constraints: the values of position derivatives that must be matched during optimization procedures.
Typically, only the positions are specified for vertices along a path, while start and end vertices have all derivatives of position set to zero.
```
  x----------x-----------------x
vertex            segment
```
In the case of multi-dimensional vertices, the derivative constraints exist in all dimensions, with possibly different values.

## Linear Optimization
In this section, we consider how to generate polynomial segments passing through a set of arbitrary vertices using the unconstrained **linear** optimization approach described in [1].

Necessary includes:
```c++
#include <mav_trajectory_generation/polynomial_optimization_linear.h>
```

1. Create a list of three (x,y,z) vertices to fly through, e.g. (0,0,1) -> (1,2,3) -> (2,1,5), and define some parameters. The ``dimension`` variable denotes the spatial dimension of the path (here, 3D). The ``derivative_to_optimize`` should usually be set to the last derivative that should be continuous (here, snap).

```c++
mav_trajectory_generation::Vertex::Vector vertices;
const int dimension = 3;
const int derivative_to_optimize = mav_planning_utils::derivative_order::SNAP;
mav_trajectory_generation::Vertex start(dimension), middle(3dimension end(dimension);
```

2. Add constraints to the vertices.

```c++
start.makeStartOrEnd(Eigen::Vector3d(0,0,1), derivative_to_optimize);
vertices.push_back(start);

middle.addConstraint(mav_trajectory_generation::derivative_order::POSITION, Eigen::Vector3d(1,2,3));
vertices.push_back(middle);

end.makeStartOrEnd(Eigen::Vector3d(2,1,5), derivative_to_optimize);
vertices.push_back(end);
```

3. Compute the segment times.

```c++
std::vector<double> segment_times;
const double v_max = 2.0;
const double a_max = 2.0;
const double magic_fabian_constant = 6.5; // A tuning parameter.
segment_times = estimateSegmentTimes(vertices, v_max, a_max, magic_fabian_constant);
```

4. Create an optimizer object and solve. The template parameter (N) denotes the number of coefficients of the underlying polynomial, which has to be even. If we want the trajectories to be snap-continuous, N needs to be at least 10.

```c++
const int N = 10;
mav_trajectory_generation::PolynomialOptimization<N> opt(dimension);
opt.setupFromVertices(vertices, segment_times, derivative_to_optimize);
opt.solveLinear();
```

5. Obtain the polynomial segments.

```c++
mav_trajectory_generation::Segment::Vector segments;
opt.getSegments(&segments);
```

## Nonlinear Optimization
In this section, we consider how to generate polynomial segments passing through a set of arbitrary vertices using the unconstrained **nonlinear** optimization approach described in [1]. The same approach is followed as in the previous section.

Necessary includes:

```c++
#include <mav_trajectory_generation/polynomial_optimization_nonlinear.h>
```

1. Set up the problem by following Steps 1-3 in the **Linear Optimization** section.

2. Set the parameters for nonlinear optimization. Below is an example, but the default parameters should be reasonable enough to use without fine-tuning.

```c++
NonlinearOptimizationParameters parameters;
parameters.max_iterations = 1000;
parameters.f_rel = 0.05;
parameters.x_rel = 0.1;
parameters.time_penalty = 500.0;
parameters.initial_stepsize_rel = 0.1;
parameters.inequality_constraint_tolerance = 0.1;
```

3. Create an optimizer object and solve. The third argument of the optimization object (true/false) specifies whether the optimization is run over the segment times only.

```c++
const int N = 10;
PolynomialOptimizationNonLinear<N> opt(dimension, parameters, false);
opt.setupFromVertices(vertices, segment_times, derivative_to_optimize);
opt.addMaximumMagnitudeConstraint(mav_trajectory_generation::derivative_order::VELOCITY, v_max);                                opt.addMaximumMagnitudeConstraint(mav_trajectory_generation::derivative_order::ACCELERATION, a_max);
opt.optimize();
```

4. Obtain the polynomial segments.

```c++
mav_trajectory_generation::Segment::Vector segments;
opt.getPolynomialOptimizationRef().getSegments(&segments);
```
## Sampling Trajectories
In this section, we consider methods of evaluating the trajectory at particular instances of time. To do this, we first need to convert our optimization result into the Trajectory class:

```c++
#include <mav_trajectory_generation/trajectory.h>

mav_trajectory_generation::Trajectory trajectory;
opt.getTrajectory(&trajectory);
```

We then have two options to sample the trajectory.

1. By evaluating directly from the class:

```c++
// Single sample:
double sampling_time = 2.0;
int derivative_order = mav_trajectory_generation::derivative_order::POSITION;
Eigen::VectorXd sample = trajectory.evaluate(sampling_time, derivative_order);

// Sample range:
double t_start = 2.0;
double t_end = 10.0;
double dt = 0.01;
std::vector<Eigen::VectorXd> result;
std::vector<double> sampling_times; // Optional.
trajectory.evaluateRange(t_start, t_end, dt, derivative_order, &result, &sampling_times);
```

2. By conversion to ```mav_msgs::EigenTrajectoryPoint``` state(s):

```c++
#include <mav_trajectory_generation_ros/trajectory_sampling.h>

mav_msgs::EigenTrajectoryPoint state;
mav_msgs::EigenTrajectoryPointVector states;

// Single sample:
double sampling_time = 2.0;
bool success = mav_trajectory_generation::sampleTrajectoryAtTime(trajectory, sample_time, &state);

// Sample range:
double t_start = 2.0;
double duration = 10.0;
double dt = 0.01;
success = mav_trajectory_generation::sampleTrajectoryInRange(trajectory, t_start, duration, dt, &states);

// Whole trajectory:
double sampling_interval = 0.01;
success = mav_trajectory_generation::sampleWholeTrajectory(trajectory, sampling_interval, &states);
```

## Bibliography
[1] C. Richter, A. Bry, and N. Roy, “Polynomial trajectory planning for quadrotor flight,” in International Conference on Robotics and Automation. Singapore: Springer, 2013.
