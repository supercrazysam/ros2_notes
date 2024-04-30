# ros2_notes  
=================  
Issue:  
I’m currently developing a ROS2 driver for an event based camera.  
The sensor publishes data at very high rates (1khz) and fairly large messages (80kb).  
Wrote the driver in ROS1: works like a charm.  
Wrote it for ROS2 (Galactic on Ubuntu): terrible performance.  
.  
This is not my first bout with ROS2. I’ve gotten a bloody nose when trying to write ROS2 drivers for other sensors as well. RMW performance was terrible (back then under Eloquent) and I couldn’t figure out what was going on.  
Before completely throwing in the towel with respect to ROS2, I decided to file a proper bug report....  (https://discourse.ros.org/t/ros2-speed/20162/18)  
.  
https://github.com/ros2/rmw_cyclonedds/issues/346  
.  
sudo sysctl -w net.core.rmem_max=67108864 net.core.rmem_default=67108864  
sudo sysctl -w net.core.wmem_max=67108864 net.core.wmem_default=67108864  

https://github.com/ros2/rmw_cyclonedds/issues/346#issuecomment-1550349624  

