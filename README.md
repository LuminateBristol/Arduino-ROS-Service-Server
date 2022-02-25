# A guide to creating a ROS service server on an Arduino
Note - example files coming soon.

## Generating the .srv and corresponding header file
For a service to work, an .srv file must first be created. This states the data type for the request and response of the ROS service. For an Arduino server, there must also be a corresponding header file which describes the service in a format that Arduino can understand. Luckily for us, the rosserial library is able to autogenerate this for us.

Firstly, navigate to your ROS packaage's directory. For this example let's call the package `arduino_ros`. For more info on creating packages, see the ROS Wiki.

Once we have navigated to the correct directory, create a srv folder and within that create a .srv file.

```
$ ~/cd catkin_ws/src/arduino_ros
$ mkdir srv
$ gedit test_service.srv
```

The srv file is formatted by describing the request data type and the response data type with three dashes `---` between them. For more info on the format of srv files, see the ROS Wiki. Populate the test_service.srv file with a string request and response named input and output respectively.

```
string input
---
string output
```

Now we need to make some changes to our package files to ensure that ROS recognises the new service and rosserial generates a header file for it. 

Open the package.xml file and ensure that these two lines are included and are not commented out:
```
<build_depend>message_generation</build_depend>
<exec_depend>message_runtime</exec_depend>
```
For the same in the CMakeLists.txt file for these three blocks of lines:
```
find_package(catkin REQUIRED COMPONENTS
   roscpp
   rospy
   std_msgs
   message_generation
)
```
```
add_service_files(
  FILES
  test_service.srv
)
```
Note - any other .srv files should be defined here too.
```
generate_messages(
  DEPENDENCIES
  std_msgs
)
```

Save those changes. At this point we should be able to do a quick check.
```
$ rossrv show test_service.srv
```

Now we have update the package so we need to rebuild the catkin workspace. Navigate back to `catkin_ws` and rebuild.
```
$ cd ../..
$ catkin_make
```

The .srv file should now have been baked into our catkin_ws ready to use. However, for the Arduino server to run, it still needs a header file so let's generate that using rosserial. Curiously the documented method of doing this is to run the following:
```
rosrun rosserial_client make_library.py <path_to_arduino_libraries> <package_name>
```
However this has not worked well for me in the past. Instead I have simply navigated to the folder where my Arduino libraries are stored, removed the ros_lib and then rebuilt it again from scratch. If your arduino libraries are saves in `sketchbook` for example:
```
$ cd
$ cd ~/sketchbook/libraries
$ rm -r ros_lib
$ rosrun rosserial_arduino make_libraries.py .
```
Hopefully one of these should work. If not there is one other alternative here: http://wiki.ros.org/rosserial/Tutorials/Adding%20Other%20Messages

At this point we have created our .srv file, added it to our workspace and created a header file ready for the Arduino server sketch. 

## Writing an Arduino server sketch
The server sketch will be uploaded onto the Arduino board and then connect to ros via the rosserial library. Take a look at the following sketch which takes an input to periodically blink the Arduino LED on and off.
```
#include <ros.h>
#include <std_msgs/String.h>
#include <arduino_ros/test_service.h>

ros:NodeHandle nh;
using arduino_ros::test_service;

void callback(const test_service::Request & req, test_service::Response & res){
  if((i++)%2){
    digitalWrite(LED_BUILTIN, HIGH);
    res.output = "LED switched on"; }
  else {
    digitalWrite(LED_BUILTIN, LOW);
    res.output = "LED switched off"; }
}

ros::ServiceServer<test_service::Request, test_service::Response> server("test_service",&callback);

void setup() {
  nh.initNode();
  nh.advertiseService(server);
}

void loop() {
  nh.spinOnce();
  delay(1000);
}
```

Looking at individual compononents of this sketch:

`#include <arduino_ros/test_service.h>` includes the previously created header file. We can check the ros_lib library to make sure this was generated in the correct place.

`using arduino_ros::test_service;` starts the service that comes from our `arduino_ros` ROS package and uses our `test_service.srv` service definition.

`void callback........` is a function that defines what the Arduino should do when a service request is recieved. The first line is a standard formate for the test_service request and response parameters. The remainder simply tells the LED to switch on/off for each service request recieved and print a confirmation of its current state. Note that res.output is the output response, the word output is used as this was defined in our .srv file.

`ros::ServiceServer<test_service::Request, test_service::Response>server("test_service",&callback);` defines the action when a service request is recieved, in this case run the callback function.

To test if this is working we can upload it to our arduino and then iniate a rosserial node on our ROS machine. The rosserial node is needed to allow communication between the Arduino and the ROS network. Ensure that the `<port_name>` matches the port that the Arduino is connected to.

```
$ rosrun rosserial_python serial_node.py <port_name>
```

In a new terminal, we can test that the service is working:
```
$ rosservice list
$ rosservice call /test_service "input "true""
```
The response of each call should be a change in LED state (on or off) and a printed message noting the new state.

## Troubleshooting: 
In some versions of the rosserial there is an error which results in the following error code when trying to call a service:
```
"ERROR: service [/topic] responded with an error: service cannot process request: service handler returned None"
```

The workaround I have been successful with (using the Ubuntu Melodic version of rosserial) is detailed in this forum: https://github.com/ros-drivers/rosserial/pull/414

An older version of `SerialClient.py` seems to work better than the updated one.








References: 
1) http://wiki.ros.org/ROS/Tutorials/CreatingMsgAndSrv
2) https://wiki.ros.org/rosserial_client/Tutorials/Generating%20Message%20Header%20Files
3) http://wiki.ros.org/rosserial/Tutorials/Adding%20Other%20Messages
4) http://wiki.ros.org/rosserial_arduino/Tutorials/Arduino%20IDE%20Setup
5) https://roboticsclub.org/redmine/projects/quadrotor/repository/revisions/c1426757626c59452c6bd8a0dad1d23430b56fae/entry/quad2/arduino/src/ros_lib/examples/ServiceServer/ServiceServer.pde
6) http://wiki.ros.org/rosserial_arduino/Tutorials/Hello%20World




