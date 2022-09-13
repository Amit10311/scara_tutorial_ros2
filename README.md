# Scara tutorial ROS2

This tutorial is made to understand the basic concepts of controlling a robot using ros2_control. In particular, it describes how to :
- Write a URDF description of a simple SCARA manipulator 
- Write a custom hardware interface for the SCARA robot
- Launch and interact with the SCARA robot
- Write a custom velocity controller in joint-space and cartesian-space
- Set up the SCARA manipulator to run with ros2_control and Gazebo 

![scara model](resources/scara_model.png)

## Writing the URDF description of the SCARA robot
The URDF file is a standard XML based file used to describe characteristic of a robot. It can represent any robot with a tree structure, except those with cycles. Each link must have only one parent. For ros2_control, there are three primary tags: `link`, `joint`, and `ros2_control`. The `joint` tag define the robot's kinematic structure, while the `link` tag defines the dynamic properties and 3D geometry. The `ros2_control` defines the hardware and controller configuration.

A good practice in ROS2 is to specify the description of the used robot in a dedicated package. In this tutorial, the package is named in a standard way `scara_description`. In this package you can find different folders containing the configuration of the used system for different ROS2 components. 

### Global URDF description using Xacro
In order to simplify the setup of the robot we often build the robot URDF description using `xacro`. Xacro (XML Macros) is an XML macro language. With xacro, you can construct shorter and more readable XML files by using macros that expand to larger XML expressions. Using xacro allows to include smaller segments of the system description for better readability. For example, in the case of the scara robot, the global urdf is defined using the [scara.config.xacro](scara_description/config/scara.config.xacro) file, formatted as follows: 
```xml 
<?xml version="1.0"?>
<!-- Scara manipulator -->
<robot xmlns:xacro="http://www.ros.org/wiki/xacro" name="scara">

    <!-- Import scara urdf file -->
    <xacro:include filename="$(find scara_description)/urdf/scara.urdf" />

    <!-- Import scara materials file -->
    <xacro:include filename="$(find scara_description)/config/materials.urdf" />

    <!-- Import scara ros2_control description -->
    <xacro:include filename="$(find scara_description)/ros2_control/scara.ros2_control.urdf" />

</robot>
```
In this xml description:
* The `robot` tag encloses all contents of the URDF file. It has a name attribute which must be specified.
* The `xacro:include` tag is used to import the geometric description, the materials file and the ros2_control description.

In the next sections, let's focus more in details on the included description files. 

### Geometry and Dynamics

In this section, let's focus on the [`scara.urdf`](scara_description/urdf/scara.urdf) description file. The URDF file describes in details the geometry of the robot as well as some additional parameters such as its visual and collision meshes, dynamics and others. 

The URDF description file is generally formatted as follows: 

``` xml
<?xml version = "1.0"?>
<robot name = "scara" xmlns:xacro="http://www.ros.org/wiki/xacro">

  <!-- Used for fixing robot to 'base_link' -->
  <link name="world"/>

  <!-- Base Link -->
  <link name = "base_link">
    <visual>
      <geometry>
        <box size = "0.12 0.12 0.05"/>
      </geometry>
      <material name = "blue"/>
      <origin xyz = "0 0 0.025"/>
    </visual>
    <inertial>
      <mass value="1" />
      <origin xyz="0 0 0.025" rpy="0 0 0" />
      <inertia ixx="1" ixy="0" ixz="0" iyy="1" iyz="0" izz="1" />
    </inertial>
    <collision>
      <geometry>
        <box size = "0.12 0.12 0.05"/>
      </geometry>
      <material name = "blue"/>
      <origin xyz = "0 0 0.025"/>
    </collision>
  </link>

  <joint name="base2world" type="fixed">
    <parent link="world"/>
    <child link="base_link"/>
  </joint>

  <!-- revolute1 1 -->
  <link name = "revolute1">
    <visual>
      <geometry>
        <cylinder length = "0.45" radius = "0.05"/>
      </geometry>
      <material name = "green"/>
      <origin xyz = "0 0 0.225"/>
    </visual>
    <inertial>
      <mass value="0.5" />
      <origin xyz="0 0 0.05" rpy="0 0 0" />
      <inertia ixx="0.5" ixy="0" ixz="0" iyy="0.5" iyz="0" izz="0.5" />
    </inertial>
    <collision>
      <geometry>
        <cylinder length = "0.45" radius = "0.05"/>
      </geometry>
      <material name = "green"/>
      <origin xyz = "0 0 0.225"/>
    </collision>
  </link>

  <joint name = "joint1" type="revolute">
    <parent link = "base_link"/>
    <child link = "revolute1"/>
    <origin xyz = "0 0 0.05"/>
    <limit effort="1000.0" lower="-1.57" upper="1.57" velocity="0.5"/>
    <axis xyz = "0 0 1"/>
    <dynamics damping="0.2" friction="0.1" />
  </joint>

  <!-- additional links and joints -->
 
</robot>
```
In this xml description:
* The `robot` tag encloses all contents of the URDF file. It has a name attribute which must be specified.
* The `link` tag defines the robot's geometry and inertia properties. It has a name attribute which will be referred to by the `joint` tags.
* The `visual` tag specifies the rotation and translation of the visual shapes. The shapes require to set the `origin` tag to fit the desired link shape.
* The `geometry`, `box` and `cylinder` tags specify the geometry of the robot link. Alternatively, you can also use the `mesh` tag to specify the location of the 3D mesh file relative to a specified ROS 2 package.
* The `collision` tag is equivalent to the `visual` tag, except the specified mesh is used for collision checking in some applications.
* The `inertial` tag specifies mass and inertia for the link. The origin tag specifies the link's center of mass. These values are used to calculate forward and inverse dynamics. Since our application does not use dynamics, uniform arbitrary values are used.
* The `<!-- additional links ... -->` comments indicates that many consecutive `link` tags will be defined, one for each link.
* The `<link name="world"/>` and `<link name="tool0"/>` elements are not required. However, it is convention to set the link at the tip of the robot to  tool0 and to define the robot's base link relative to a world frame.
* The `joint` tag specifies the kinematic structure of the robot. It two required attributes: name and type. The type specifies the viable motion between the two connected links. The subsequent `parent` and `child` links specify which two links are joined by the joint.
* The `axis` tag species the joint's axis of rotation. If the meshes were process as described previously, then the axis value is always `"0 0 1"`.
* The `limits` tag specifies kinematic and dynamic limits for the joint.
* The `dynamics` tag specifies some dynamics properties of the joint such as its damping or friction coefficients.

## Hardware setup for ros2_control

In this section, let's focus on the [`scara.ros2_control.urdf`](scara_description/ros2_control/scara.ros2_control.urdf) description file. This description file is used to set up the ros2_control hardware that will be used to specify the `command_interface` and `state_interface` for each `joint`, `sensor` and/or `gpio`. 

The ros2_control description is generally formatted as follows:  

```xml
<?xml version="1.0"?>
<robot name = "scara" xmlns:xacro="http://www.ros.org/wiki/xacro">

    <ros2_control name="scara" type="system">

        <hardware>
            <plugin>scara_hardware/ScaraRobot</plugin>
        </hardware>

        <joint name="joint1">
            <command_interface name="position" />
            <state_interface name="position">
                <param name="initial_value">0.0</param>
                <param name="min">-1.57</param>
                <param name="max">1.57</param>
            </state_interface>
            <state_interface name="velocity"> 
                <param name="initial_value">0.0</param> 
            </state_interface>
        </joint>
        
        <!-- additional joints -->

    </ros2_control>

</robot>
```
In this xml description:
* The `robot` tag encloses all contents of the URDF file. It has a name attribute which must be specified.
* The `hardware` and `plugin` tags instruct the ros2_control framework to dynamically load a hardware driver conforming to `HardwareInterface` as a plugin. The plugin is specified as `{Name_Space}/{Class_Name}`.
* The`joint` tag specifies the state and command interfaces that the loaded plugin is will offer. The joint is specified with the name attribute. The `command_interface` and `state_interface` tags specify the interface type, usually position, velocity, acceleration, or effort. Additionally, for each interface additional parameters such as `min`, `max` and `initial_value` can be set.  

The hardware interface that can be loaded here as a plugin is dependant of the type of robot that is controlled and its control mode. It is an interface between ros2_control and the robot driver. For robots that support ros2_control, the interface is often given either by the manufacturer or the community. A non exhaustive list of available hardware interfaces can be found [here](https://control.ros.org/master/doc/supported_robots/supported_robots.html). 

In the case if the hardware interface is not available or not suited for the desired application, it can be developed in a custom way, what is the topic of the next section.

## Writing of a hardware interface

In ros2_control, hardware system components are integrated via user defined driver plugins that conform to the `HarwareInterface` public interface. Hardware plugins specified in the URDF are dynamically loaded during initialization using the `pluginlib` interface. More information about creating and using plugins can be found [here](https://docs.ros.org/en/humble/Tutorials/Beginner-Client-Libraries/Pluginlib.html).

For the purpose of this tutorial, let's create the custom interface plugin `ScaraRobot` that will be used to simulate a scara robot. We want the simulated system to be controlled in joint position and provide information about its current position and velocity, as described in the ros2_control description [file](scara_description/ros2_control/scara.ros2_control.urdf).  

To do so, in the `scara_hardware` package, let's first define the hardware plugin called `ScaraRobot` that inherits from  `hardware_interface::SystemInterface`. The `SystemInterface` is one of the offered hardware interfaces designed for a complete robot system. For example, The UR5 uses this interface. The `ScaraRobot` must implement five public methods:
1. `on_init`
2. `export_state_interfaces`
3. `export_command_interfaces`
4. `read`
5. `write`

These methods are defined in the [scara_robot.hpp](scara_hardware/include/scara_hardware/scara_robot.hpp) header file as follows: 

```c++
using CallbackReturn = rclcpp_lifecycle::node_interfaces::LifecycleNodeInterface::CallbackReturn;
#include "hardware_interface/types/hardware_interface_return_values.hpp"

class HARDWARE_INTERFACE_PUBLIC ScaraRobot : public hardware_interface::SystemInterface {
    public:
    CallbackReturn on_init(const hardware_interface::HardwareInfo &info) override;
    std::vector<hardware_interface::StateInterface> export_state_interfaces() override;
    std::vector<hardware_interface::CommandInterface> export_command_interfaces() override;
    return_type read(const rclcpp::Time &time, const rclcpp::Duration &period) override;
    return_type write(const rclcpp::Time & /*time*/, const rclcpp::Duration & /*period*/) override;
    // private members
    // ...
}
```

### Initializing the hardware
Let's first have a look at the initialization `on_init` method. The `on_init` method is called once during ros2_control initialization if the `ScaraRobot` was specified in the URDF. This method should:
- Check the validity of the requested `command_interfaces` and `state_interfaces` w.r.t. the loaded driver
- Instantiate the communication with the robot hardware
- Allocate memory

Since in this tutorial the robot is simulated, no communication need to be established. Instead, vectors will be initialized that represent the state all the hardware using the initial values from the description file. The definition of this method is as follows: 

```c++
CallbackReturn ScaraRobot::on_init(const hardware_interface::HardwareInfo &info) {
    if (hardware_interface::SystemInterface::on_init(info) != CallbackReturn::SUCCESS) {
        return CallbackReturn::ERROR;
    }

    // allocate memory 
    hw_states_position_.resize(info_.joints.size() std::numeric_limits<double>::quiet_NaN());
    //...
    // check the validity of the description
    for (const hardware_interface::ComponentInfo & joint : info_.joints) {
        if (joint.command_interfaces[0].name != hardware_interface::HW_IF_POSITION)
        {
            return CallbackReturn::ERROR;
        }
    }
    // ...
    // initialize states
    for (uint i = 0; i < info_.joints.size(); i++) {
        hw_states_position_[i] = std::stod(info_.joints[i].state_interfaces[0].initial_value);
    }
    // ...
    return CallbackReturn::SUCCESS;
}
```
Note that the behavior of `on_init` is expected to vary depending on the URDF description file. The `SystemInterface::on_init(info)` call fills out the `info` object with specifics from the URDF. The `info` object has fields for joints, sensors, gpios, and more. This allows to check if the interface that is called is compatible with the description and use to parameters from the description file to set up the hardware. 

## Exporting interfaces

Next, `export_state_interfaces` and `export_command_interfaces` methods are called in succession. Their purpose is to create a handle to link the internal state/command variable with the ros2_control framework so that it can be accessed from any method. The `export_state_interfaces` method returns a vector of `StateInterface` describing the `state_interfaces` for each joint. The `StateInterface` objects are read only data handles that contain the interface name, interface type, and a pointer to a double data value of the internal state variable. For the `ScaraRobot`, the `export_state_interfaces` references `hw_states_position_` vector with the position state interface and is defined as follows: 
```c++
std::vector<hardware_interface::StateInterface> ScaraRobot::export_state_interfaces() {
    std::vector<hardware_interface::StateInterface> state_interfaces;
    for (uint i = 0; i < info_.joints.size(); i++) {
        state_interfaces.emplace_back(
            hardware_interface::StateInterface(
                info_.joints[i].name, hardware_interface::HW_IF_POSITION, &hw_states_position_[i]));
    }
    // ...
    return state_interfaces;
}
```
The `export_command_interfaces` method is nearly identical to the previous one. The difference is that a vector of `CommandInterface` is returned. The vector contains objects describing the `command_interfaces` for each joint. For the `ScaraRobot`, the `export_command_interfaces` references the `hw_commands_position_` vector with the position command interface and is defined as follows: 
```c++
std::vector<hardware_interface::CommandInterface> ScaraRobot::export_command_interfaces() {
    std::vector<hardware_interface::CommandInterface> command_interfaces;
    for (uint i = 0; i < info_.joints.size(); i++) {
        command_interfaces.emplace_back(
            hardware_interface::CommandInterface(
                info_.joints[i].name, hardware_interface::HW_IF_POSITION, &hw_commands_position_[i]));
    }
    return command_interfaces;
}
```
Now that the Hardware Interface is initialized, connected to the robot and that the internal variables are connected to the ros2_control framework, let's focus on the main control loop. 

## Reading and writing from the hardware
In ros2_control the main control loop consists in successive calls of the hardware `read` method, followed by the controller `update` method, followed by the hardware `write` method. In the read phase of the main loop, ros2_control loops over all hardware components that where loaded to call their `read` method. It is executed on the realtime thread, hence the method must obey by realtime constraints. The `read` method is responsible for accessing the robot current state and updating the data values of the `state_interfaces`. 
In this tutorial, as we only want to simulate the robot, we compute its current velocity as follows:  
```c++
hardware_interface::return_type ScaraRobot::read(const rclcpp::Time & time, const rclcpp::Duration &period) {
    // read hardware values for state interfaces, e.g joint encoders and sensor readings
    for (uint i = 0; i < info_.joints.size(); i++) {
        hw_states_velocity_[i] = (hw_states_position_[i] - hw_states_previous_position_[i])/(period.nanoseconds()*1e-9);

        hw_states_previous_position_[i] = hw_states_position_[i];
    }
    return hardware_interface::return_type::OK;
}
```
In the same way, during the write phase of the main loop, the `write` method of all loaded hardware components is called after the controller `update` in the realtime loop. For this reason, the `write` method must also obey by realtime constraints. The `write` method is responsible for updating the data values of the `command_interfaces`. 
In the case of our scara robot, the methods is defined as follows:
```c++
hardware_interface::return_type write(const rclcpp::Time & time, const rclcpp::Duration & period) {
    // send command interface values to hardware, e.g joint set joint velocity
    bool isNan = false;
    for (auto i = 0ul; i < hw_commands_position_.size(); i++) {
        if (hw_commands_position_[i] != hw_commands_position_[i]) isNan = true;
    }

    if (!isNan) {
        for (uint i = 0; i < info_.joints.size(); i++) {
            double min_position = std::stod(info_.joints[i].state_interfaces[0].min);
            double max_position = std::stod(info_.joints[i].state_interfaces[0].max);

            hw_states_position_[i] = hw_commands_position_[i];

            if(hw_states_position_[i] > max_position) hw_states_position_[i] = max_position;
            if(hw_states_position_[i] < min_position) hw_states_position_[i] = min_position;
        }
    }

    return hardware_interface::return_type::OK;
}
```
Notice here that in the first part of the methods we check if the value command is valid (i.e. not NAN), that means that a command was received through the `command_interface`. Also, notice that we can use some information from the ros2_control description file such as position `min` and `max` parameters to prevent the robot to go outside of the its limits.

Now that we have defined our hardware interface, let's focus on building it in the next section.

## Building the Hardware Interface plugin
Building the Hardware Interface plugin is done with the following steps:
* Adding C++ export macro 
* Creating the plugin description file
* Exporting the CMake library

### Adding C++ export macro
In order to reference the previously defined hardware interface as a ros2_control plugin, we need to add the following two lines of code at the end of the [scara_robot.cpp](scara_hardware/src/scara_robot.cpp) file containing our method definitions:

```c++
#include "pluginlib/class_list_macros.hpp"

PLUGINLIB_EXPORT_CLASS(scara_hardware::ScaraRobot, hardware_interface::SystemInterface)
```
The `PLUGINLIB_EXPORT_CLASS` is a c++ macro that creates a plugin library using `pluginlib`. More information about it can be found [here](https://docs.ros.org/en/humble/Tutorials/Beginner-Client-Libraries/Pluginlib.html).

### Creating the plugin description file
The plugin description file is a required XML file that describes the plugin's library name, class type, namespace, description, and interface type. This file allows ROS2 to automatically discover and load plugins. It is formatted as follows:

```xml
<library path="{Library_Name}">
  <class
    name="{Namespace}/{Class_Name}"
    type="{Namespace}::{Class_Name}"
    base_class_type="hardware_interface::SystemInterface">
  <description>
    {Human readable description}
  </description>
  </class>
</library>
```

The `path` attribute of the `library` tags refers to the cmake library name of the user defined hardware plugin. See [here](scara_hardware/scara_hardware_plugin.xml) for the complete XML file used for this tutorial.

### Exporting the CMake library
The general CMake template to make a hardware plugin available in ros2_control is shown below:
```cmake
add_library(
    scara_hardware
    SHARED
    src/scara_robot.cpp
)

# include and link dependencies
# ...

# Causes the visibility macros to use dllexport rather than dllimport, which is appropriate when building the dll but not consuming it.
target_compile_definitions(scara_hardware PRIVATE "HARDWARE_PLUGIN_DLL")
# export plugin
pluginlib_export_plugin_description_file(scara_hardware scara_hardware_plugin.xml)
# install libraries
# ...
```
Notice that a library is created using the plugin source code just like any other cmake library. In addition, an extra compile definition and cmake export macro (`pluginlib_export_plugin_description_file`) need to be added. See [here](scara_hardware/CMakeLists.txt) for the complete `CMakeLists.txt` file used for this tutorial. 

Now that our scara robot's hardware is ready to be loaded as a plugin let's run our scara robot!

## Launching and interacting with the Scara robot


## Acknowledgments 
This tutorial is partially inspired from [pac48](https://github.com/pac48/ros2_control_demos/tree/full-example-tutorial)'s tutorial.