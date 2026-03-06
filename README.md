# Alex Camaj Worklog

[[_TOC_]]

# 2026-02-17 - Worklog Entry

Joined Any-Surface-Stylus for Computer (team 86) late due to my original project getting unapproved last week. For previous project had data architecture designed as well as specific requirements for each sub-part determined. Talked with machine shop about Polycase for getting enclosurement for the main frame of the design. I read and understood the current proposal for the current team as well as added requirements for the project components. 

# 2026-02-23 - Determining Parts

Over the past few days, I have been looking into determining which sensor would be best for our main way of tracking movement with the stylus pen. Through research, I determined there was 2 main pathways. One was to use an optical flow sensor to see x and y movement. The other was to use a 3axis IMU to determine the relative change in position of the pen and then update the cursor based on wrist rotation over movement. The problem I found with the optical flow sensor was that there was a lot of issues with the range. The minimum range flow sensor I could find was 80mm which means the sensor has to be at least 80mm off the ground in order to work. This poses an issue with the pen since it will make us need to have a bulky design and have the pen have a sensor pop out the side without being obscured. The other option which I tested with was an IMU. I used a microcontroller board with an LSM6DSL IMU built in and I was able to use that to write an algorithm to move the cursor on my computer through the pitch and roll movement of the microcontroller. This shows us that an IMU is very practical for our application. The remaining problem was what sensor to use when the board is on the table since the IMU will be less effective. I believe going with a standard mouse optical sensor would do the job, and based on whether the pen is down or up we can decide which sensor controls the cursor movement. The LSM6DSL has 6 DOF, which is the perfect amount for this stylus. A standard PMW3360 would be fit for the on-surface tracking. <img width="750" height="500" alt="image" src="https://github.com/user-attachments/assets/3adf30f3-8ba0-4dfc-a8b9-dd5721e67255" />

# 2026-03-06 - Final Design Updates

Over the past week, we have finalized parts and are moving into the implementation phase. We will be implementing an optical sensor with a fiber optic cable in order to get the lighting to reach the lens. This will be used for the ground surface tracking, and the IMU will control air movement. Right now, we need to update our requirements and verification since a lot of them are very strict and time-consuming. The design document was the main focus of last week, and we were able to put all of our research into that. We begin testing this weekend with our optical sensor and HID interface to setup basic mouse functionality for our design. 
