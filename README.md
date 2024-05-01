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

===

![alt text](https://github.com/supercrazysam/ros2_notes/blob/main/Screenshot%20from%202024-04-30%2013-14-28.png)

![alt text](https://github.com/supercrazysam/ros2_notes/blob/main/Screenshot%20from%202024-04-30%2015-59-18.png)  


===================  
I don’t have an easy fix, and I fully believe I understand why things are the way they are in ROS 2  
and I also understand that making high-performance networking applications is still complex and needs lots of moving parts to do just the right things in the right way,  
but seeing what @codebot had to do to get to the performance of @Bernd_Pfrommer’s ‘naive’ ROS 1 implementation makes me sad..... (https://discourse.ros.org/t/ros2-speed/20162/23)  

https://discourse.ros.org/t/ros2-speed/20162/19    (involve setting rmem wmem  and not using certain types,  use some other type instead)  
https://discourse.ros.org/t/ros2-speed/20162/21  

===================  

Some closing remarks on the ROS2 performance issues I encountered when writing a ROS2 driver for the event based Prophesee cameras 38.  

The serialization/deserialization of structures in ROS2 is currently much slower than in ROS1 in particular if such structures contain other non-primitive data types like e.g. Time. To get high performance in ROS2 currently requires avoiding such situations. I ended up squeezing all event data into a uint64 field and then essentially packing/unpacking manually into that 64bit space. As a fringe benefit, the compressed data format also cut the storage and bandwidth requirements by almost a factor of two.  

AFAIK Galactic currently does not support running rosbag record as a composable node. However with a small temporary tweak to the Recorder code 11 I was able to write my own composable node that can store without inter-process communication.  

With all the necessary optimizations in place (simple data types, composable nodes) ROS2 (cyclonedds) actually runs with slightly less CPU consumption than ROS1 as shown in this table (scroll all the way down) 38. So it’s not like ROS2 can’t perform, it’s just that one has to work harder to get there.  

I’m still curios though as to what the path forward is for fixing the serialization/deserialization performance issues. Can/will these be fixed by a more efficient rmw implementation?  ....... (https://discourse.ros.org/t/ros2-speed/20162/28)  

==================  

Potential Problem:  
-   Network throughput not enough:  https://docs.ros.org/en/foxy/How-To-Guides/DDS-tuning.html#cyclone-dds-tuning
-   Serialization slow by default in ROS2 problem:  https://docs.ros.org/en/foxy/How-To-Guides/DDS-tuning.html#cross-vendor-tuning (under Issue: Sending custom messages with large variable-sized arrays of non-primitive types causes high serialization/deserialization overhead a.....)

Comment about non-primative datatype serialization speed: 
  " high (de)serialization cost of custom messages with large variable-sized arrays of non-primitive types"  
  It's also not specific to custom messages, it's true of any message with any non-primitive type over any DDS.  
  .  
  There is something seriously, seriously wrong with either the implementation or the design of the serialization for it to perform as badly as it does.  
  https://github.com/ros2/rmw_cyclonedds/issues/346#issuecomment-1162596314  

More hints on DDS tuning (looks good info):  
http://59.110.158.57/35448  

===================  

Since ros2 by default use fastdds,  here are the things to note when using fastdds,  for optimization:  
https://docs.ros.org/en/humble/Tutorials/Advanced/FastDDS-Configuration.html  

====================  
Environmental variable setup:  
export ROS_DOMAIN_ID=0         #all ros2 nodes use 0 by default,   this exist so that there can be separate clusters of nodes, sharing the same network interface, while not interacting with stuff from other cluster.  
export ROS_LOCALHOST_ONLY=1    #limits everything within localhost  
https://docs.ros.org/en/humble/Tutorials/Beginner-CLI-Tools/Configuring-ROS2-Environment.html  


export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
export RMW_FASTRTPS_USE_QOS_FROM_XML=1
export FASTRTPS_DEFAULT_PROFILES_FILE=/home/shum/SyncAsync.xml

```
<?xml version="1.0" encoding="UTF-8" ?>
<profiles xmlns="http://www.eprosima.com/XMLSchemas/fastRTPS_Profiles">

    <!-- default publisher profile -->
    <publisher profile_name="default_publisher" is_default_profile="true">
        <historyMemoryPolicy>DYNAMIC</historyMemoryPolicy>
    </publisher>

    <!-- default subscriber profile -->
    <subscriber profile_name="default_subscriber" is_default_profile="true">
        <historyMemoryPolicy>DYNAMIC</historyMemoryPolicy>
    </subscriber>

    <!-- publisher profile for topic sync_topic -->
    <publisher profile_name="/sync_topic">
        <historyMemoryPolicy>DYNAMIC</historyMemoryPolicy>
        <qos>
            <publishMode>
                <kind>SYNCHRONOUS</kind>
            </publishMode>
        </qos>
    </publisher>

    <!-- publisher profile for topic async_topic -->
    <publisher profile_name="/async_topic">
        <historyMemoryPolicy>DYNAMIC</historyMemoryPolicy>
        <qos>
            <publishMode>
                <kind>ASYNCHRONOUS</kind>
            </publishMode>
        </qos>
    </publisher>

 </profiles>

```

======================  
ROS2 QoS settings:  
https://docs.ros.org/en/humble/Concepts/Intermediate/About-Quality-of-Service-Settings.html

======================  
ROS1 bag to ROS2 bag  :
https://discourse.ros.org/t/ros1-bag-file-to-ros2-bag-file-converter/19995/3  
or  
https://docs.openvins.com/dev-ros1-to-ros2.html  
*seems like both method are the same, both pointing to the use of "rosbags" python library that supports ROS1 <=> ROS2 bags mainpulation. There are options to exclude, include topics as well.  

======================  

[Optional] Good to have for realtime operation, in the future:  
Implementing a custom memory allocator  
why? Suppose you want to write real-time safe code, and you’ve heard about the many dangers of calling “new” during the real-time critical section, because the default heap allocator on most platforms is nondeterministic....  
https://docs.ros.org/en/humble/Tutorials/Advanced/Allocator-Template-Tutorial.html  

=======================  
Important info for intraprocess comm (ROS2)  
https://docs.ros.org/en/humble/Tutorials/Demos/Intra-Process-Communication.html  

========================  

ROS2 design philosophy?  
http://design.ros2.org/  
