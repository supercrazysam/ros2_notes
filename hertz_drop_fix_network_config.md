Multiple Network Interfaces  

Fast-DDS supports multiple network interfaces out of the box. A ROS 2 process will automatically use all the interfaces that were available when it started (it will not use network interfaces activated while the process was already running).  
Disable Multicast  

Some networks (e.g., academic or corporate Wi-Fi) may block the multicast packets used by ROS 2 by default. The following XML profile can be used on your laptop (or compute board) to force using unicast and directly connect to the IP address of your robot.  
https://iroboteducation.github.io/create3_docs/setup/xml-config/  

===============================  


I applied this solution (https://robotics.stackexchange.com/questions/98466/how-to-specify-the-network-interface-ros2-uses-for-communication) for selecting specific networks interfaces for ros2 to use, but when specified, it still seems to end up using other interfaces.  
https://robotics.stackexchange.com/questions/107919/how-to-get-select-ros2-network-interface-with-xml-working  
