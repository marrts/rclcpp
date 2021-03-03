# Notes on statically typed parameters

## Introduction

Up to ROS 2 Foxy, all parameters could be changed of type anytime, except the user installed a parameter callback to reject that change.
This could generate confusing errors, for example:

```
$ ros2 run demo_nodes_py listener &
$ ros2 param set /listener use_sim_time not_a_boolean
[ERROR] [1614712713.233733147] [listener]: use_sim_time parameter set to something besides a bool
Set parameter successful
$ ros2 param get /listener use_sim_time
String value is: not_a_boolean
```

For most use cases, having a static parameter types is enough.
This notes document some of the decisions that were made when implementing static parameter types enforcement in:

* https://github.com/ros2/rclcpp/pull/1522
* https://github.com/ros2/rclpy/pull/683

## Allowing dynamically typed parameters

There might be some scenarios were dynamic typing is desired, so a `dynamic_typing` field was added to the [parameter descriptor](https://github.com/ros2/rcl_interfaces/blob/09b5ed93a733adb9deddc47f9a4a8c6e9f584667/rcl_interfaces/msg/ParameterDescriptor.msg#L25).
It defaults to `false`.

For example:

```cpp
rcl_interfaces::msg::ParameterDescriptor descriptor;
descriptor.dynamic_typing = true;

node->declare_parameter("dynamically_typed_parameter", rclcpp::ParameterValue{}, descriptor);
```

```py
rcl_interfaces.msg.ParameterDescriptor descriptor;
descriptor.dynamic_typing = True;

node.declare_parameter("dynamically_typed_parameter", None, descriptor);
```

## How is the parameter type specified?

The parameter type will be inferred from the default value provided when declaring it.

## Statically typed parameters when allowing undeclared parameters

When undeclared parameters are allowed, the descriptor of undeclared parameters will have the dynamic typing field set.
That's because in that case there is no declaration, and it doesn't make much sense to enforce the type of the first value set.

TBD: Should parameters that were declared with the dynamic typing option off enforce that the parameter doesn't change the type even if allow undeclared parameters is set?
In the current implementation that isn't enforce, but it might make sense to do it for parameters that were declared.

## Declaring a parameter without a default value

There might be cases were a default value does not make sense and the user must always provide an override.
In those cases, the parameter type can be specified explicetly:

```cpp
// method signature
template<typename T>
Node::declare_parameter<T>(std::string name, rcl_interfaces::msg::ParameterDescriptor = rcl_interfaces::msg::ParameterDescriptor{});
// or alternatively
Node::declare_parameter(std::string name, rclcpp::ParameterType type, rcl_interfaces::msg::ParameterDescriptor = rcl_interfaces::msg::ParameterDescriptor{});

// examples
node->declare_paramter<int64_t>("my_integer_parameter");  // declare an integer parameter
node->declare_paramter("another_integer_parameter", rclcpp::ParameterType::PARAMETER_INTEGER);  // another way to do the same
```

```py
# method signature
Node.declare_parameter(name: str, param_type: rclpy.Parameter.Type, descriptor: rcl_interfaces.msg.ParameterDescriptor = rcl_interfaces.msg.ParameterDescriptor())

# example
node.declare_paramter('my_integer_parameter', rclpy.Parameter.Type.INTEGER);  # declare an integer parameter
```

If the parameter is optional, e.g.: only needed depending on the value of another parameter, the recommended approach is to conditionally declare the parameter:

```cpp
auto mode = node->declare_parameter("mode", "modeA");  // "mode" parameter is an string
if (mode == "modeB") {
    node->declare_parameter<int64_t>("param_needed_when_mode_b");  // when "modeB", the user must provide "param_needed_when_mode_b"
}
```

## Other migration notes

Declaring a parameter with only a name is deprecated:

```cpp
node->declare_parameter("my_param");  // this generates a build warning
```

```py
node.declare_parameter("my_param");  # this generates a python user warning
```

Before, you could initialize a parameter with the "NOT SET" value (in cpp `rclcpp::ParameterValue{}`, in python `None`).
Now this will throw an exception in both cases:

```cpp
node->declare_parameter("my_param", rclcpp::ParameterValue{});  // not valid, will throw exception
```

```py
node.declare_parameter("my_param", None);  # not valid, will raise error
```