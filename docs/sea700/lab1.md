# Lab 1 : Intro to ROS and TurtleSim

<font size="5">
Seneca Polytechnic</br>
SEA700 Robotics for Software Engineers
</font>

## Introduction

In this course, you'll be using the [JetAuto Pro ROS Robot Car](https://www.hiwonder.com/products/jetauto-pro?variant=40040875229271&srsltid=AfmBOopGiD-7htdo9MYV2dDxaC_hu9xuS887mAu3p1SuU0YDl9iVj6Da) as the base platform for development. The JetAuto robot uses the NVIDIA Jetson Nano embedded computing boards as its controller. The robot comes with various accessories and add-ons for robotics applications and they can all be controlled using [Robot Operating System (ROS)](https://www.ros.org/) as the backbone application. However, before jumping into using the physical, we will be learning how to use ROS and [Gazebo](https://gazebosim.org/) to simulate the robot and it's environmental. During the design phase of any robotics project, testing the functionality of your code and the robot is a key step of project cycle. In some industrial settings (such as robot in remote location), physical access to the robot are limited or impossible. In such a case, simulating the robot's action and performance is the only way to ensure mission success.

### What is ROS?

Per the ROS website:

> The Robot Operating System (ROS) is a set of software libraries and tools that help you build robot applications. From drivers to state-of-the-art algorithms, and with powerful developer tools, ROS has what you need for your next robotics project. And it's all open source.

The above statement isn't exactly clear so let's take a look at an example.

![Figure 1.1 Robotics Arm](lab1-robot-arm.png)

***Figure 1.1** Robotics Arm (Source: Hiwonder)*

A robotics system usually consist of various sensors, actuators, and controllers. The system in Figure 1.1 have the following:

1. a servo gripper at the end of the arm
1. a servo revolute joint that rotate the gripper
1. a servo revolute joint for link 3-2
1. a servo revolute joint for link 4-3
1. a servo revolute joint for link 5-4
1. a servo revolute joint that rotate the base
1. a stationary camera supervise the workspace

To pick up an object, the robot might:

- Use the camera to measure the position of the object
- Command the arm to move toward the object's position
- Once in position, command the gripper to close around the object

To approach this problem, we'll need to break it down into smaller task. In robotics, that usually means having independent low-level control loops, each controlling a single task.

- A control loop for each joint that, given a position or velocity command, controls the power applied to the joint motor based on position sensor measurements at the joint.
- Another control loop that receives commands to open or close the gripper, then switches the gripper motor on and off while controlling the power applied to it to avoid crushing objects.
- A sensing loop that reads individual images from the camera

With the above structure, we then couple these low-level loops together via a single high-level module that performs supervisory control of the whole system:

- Query the camera sensing loop for a single image
- Use a vision algorithm to compute the location of the object to grasp
- Compute the joint angles necessary to move the manipulator arm to this location
- Sends position commands to each of the joint control loops telling them to move to this position
- Signal the gripper control loop to close the gripper to grab the object

An important feature of this design is that the supervisor need not know the implementation details of any of the low-level control loops: it interacts with each only through simple control messages. This encapsulation of functionality makes the system modular, making it easier to reuse code across robotic platforms.

In ROS, each individual control loop is known as a node, an individual software process that performs a specific task. Nodes exchange control messages, sensor readings, and other data by publishing or subscribing to topics or by sending requests to services offered by other nodes (these concepts will be discussed in detail later in the lab).

Nodes can be written in a variety of languages (including Python and C++), and ROS transparently handles the details of converting between different datatypes, exchanging messages between nodes, etc.

We can then visualize the communication and interaction between different software components via a computation graph, where:

![Figure 1.2 Example computation graph](lab1-computation-graph.png)

***Figure 1.2** Example computation graph*

- Nodes are represented by ovals (ie. `/usb_cam` or `/ar_track_alvar`).
- Topics are represented by rectangles (ie. `/usb_cam/camera_info` and `/usb_cam/image_raw`).
- The flow of information to and from topics and represented by arrows. In the above example, `/usb_cam publishes`
to the topics `/usb_cam/camera_info` and `/usb_cam/image_raw`, which are subscribed to by `/ar_track_alvar`.
- While not shown here, services would be represented by dotted arrows.

## Procedures

The lab procedures assume a Ubuntu Jammy 22.04 enviornment. If you are using Windows or macOS, some modification to the steps might be required.

### ROS Installation

1. Follow the [ROS 2 Documentation Installation](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html) instruction to install the ROS desktop package into your system. **You do NOT need to install the Bare Bones or the Development tools**

1. Since we want our terminal to load the ROS source everytime it start, add the `source` command to `.bashrc`.

    Use `vi`, `vim` or any editor to open `~/.bashrc` in your user's home directory then add the following code at the end.

        source /opt/ros/humble/setup.bash
        
1. After installation and setting up `.bashrc`, ensure you successfully tested the C++ talker and Python listener per the instruction by opening a NEW terminal.

### Turtlesim Test

Turtlesim is a lightweight simulator for learning ROS 2. It illustrates what ROS 2 does at the most basic level to give you an idea of what you will do with a real robot or a robot simulation later on.

The ros2 tool is how the user manages, introspects, and interacts with a ROS system. It supports multiple commands that target different aspects of the system and its operation. One might use it to start a node, set a parameter, listen to a topic, and many more. The ros2 tool is part of the core ROS 2 installation.

1. Check that the turtlesim package is installed.

        ros2 pkg executables turtlesim

    The above command should return a list of turtlesim’s executables.
    
        turtlesim draw_square
        turtlesim mimic
        turtlesim turtle_teleop_key
        turtlesim turtlesim_node

1. Start turtlesim by entering the following command in your terminal.

        ros2 run turtlesim turtlesim_node

    The simulator window should appear, with a random turtle in the center.

    ![Figure 1.3 TurtleSim](lab1-turtlesim.png)

    ***Figure 1.3** TurtleSim*

    In the terminal, under the command, you will see messages from the node:

        [INFO] [1725638823.233052860] [turtlesim]: Starting turtlesim with node name /turtlesim
        [INFO] [1725638823.245832389] [turtlesim]: Spawning turtle [turtle1] at x=[5.544445], y=[5.544445], theta=[0.000000]

1. Open a new terminal to run a new node to control the turtle in the first node. If you didn't add the `source` code in `.bashrc`, you'll need to source ROS 2 again.

        ros2 run turtlesim turtle_teleop_key

    At this point you should have three windows open: a terminal running `turtlesim_node`, a terminal running `turtle_teleop_key` and the "turtlesim window". Arrange these windows so that you can see the turtlesim window, but also have the terminal running `turtle_teleop_key` active so that you can control the turtle in turtlesim.

1. Use the arrow keys on your keyboard to control the turtle. It will move around the screen, using its attached “pen” to draw the path it followed so far.

### Use rqt

rqt is a graphical user interface (GUI) tool for ROS 2. Everything done in rqt can be done on the command line, but rqt provides a more user-friendly way to manipulate ROS 2 elements.

1. Open a new terminal and run rqt.

        rqt

1. When running rqt for the first time, the window will be blank. No worries; just select **Plugins > Services > Service Caller** from the menu bar at the top.

    ![Figure 1.4 rqt](lab1-rqt.png)

    ***Figure 1.4** rqt*

1. Use the refresh button to the left of the Service dropdown list to ensure all the services of your turtlesim node are available.

1. Click on the Service dropdown list to see turtlesim’s services, and select the `/spawn` service to spawn another turtle.

    Give the new turtle a unique name, like `turtle2`, by double-clicking between the empty single quotes in the **Expression** column. You can see that this expression corresponds to the value of **name** and is of type **string**.

    Next enter some valid coordinates at which to spawn the new turtle, like x = `1.0` and y = `1.0`.

    ![Figure 1.5 rqt spawn](lab1-rqt-spawn.png)

    ***Figure 1.5** rqt spawn*

    If you try to spawn a new turtle with the same name as an existing turtle, you will get an error message in the terminal running `turtlesim_node`.
    
1. Call the `spawn` service by clicking the Call button on the upper right side of the rqt window. You should see a new turtle (with a random design) spawn at the coordinates you input for x and y.

1. Refresh the service list in rqt and you will also see that now there are services related to the new turtle, `/turtle2/...`, in addition to `/turtle1/...`.

1. Next, we'll give `turtle1` an unique pen using the `/set_pen` service and have turtle1 draw with a distinct red line by changing the value of **r** to `255`, and the value of **width** to `5`. Don’t forget to call the service after updating the values.

    ![Figure 1.6 rqt set_pen](lab1-rqt-set-pen.png)

    ***Figure 1.6** rqt set_pen*

1. Return to the terminal where `turtle_teleop_key` is running and press the arrow keys, you will see `turtle1`’s pen has changed.

    ![Figure 1.7 TurtleSim Turtles](lab1-turtlesim-2.png)

    ***Figure 1.7** TurtleSim Turtles*

### Remapping `turtle_teleop_key`

1. To control `turtle2`, you need a second teleop node. However, if you try to run the same command as before, you will notice that this one also controls turtle1. The way to change this behavior is by remapping the `cmd_vel` topic.

    In a new terminal, source ROS 2, and run:

        ros2 run turtlesim turtle_teleop_key --ros-args --remap turtle1/cmd_vel:=turtle2/cmd_vel

    Now, you can move `turtle2` when this terminal is active, and `turtle1` when the other terminal running `turtle_teleop_key` is active.

    ![Figure 1.8 TurtleSim Turtles Moving](lab1-turtlesim-3.png)

    ***Figure 1.8** TurtleSim Turtles*

## Lab Question

1. Create a third turtle that you can control in turtlesim with green (g = 255) as the pen line colour.

Once you've completed all the above steps, ask the lab professor or instructor over and demostrate that you've completed the lab and written down all your observations. You might be asked to explain some of the concepts you've learned in this lab.

## Reference

- [ROS 2 Documentation: Humble](https://docs.ros.org/en/humble/index.html)
- EECS 106A Labs