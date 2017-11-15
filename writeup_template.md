## Project: Perception Pick & Place
### Writeup Template: You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---


# Required Steps for a Passing Submission:
1. Extract features and train an SVM model on new objects (see `pick_list_*.yaml` in `/pr2_robot/config/` for the list of models you'll be trying to identify). 
2. Write a ROS node and subscribe to `/pr2/world/points` topic. This topic contains noisy point cloud data that you must work with.
3. Use filtering and RANSAC plane fitting to isolate the objects of interest from the rest of the scene.
4. Apply Euclidean clustering to create separate clusters for individual items.
5. Perform object recognition on these objects and assign them labels (markers in RViz).
6. Calculate the centroid (average in x, y and z) of the set of points belonging to that each object.
7. Create ROS messages containing the details of each object (name, pick_pose, etc.) and write these messages out to `.yaml` files, one for each of the 3 scenarios (`test1-3.world` in `/pr2_robot/worlds/`).  [See the example `output.yaml` for details on what the output should look like.](https://github.com/udacity/RoboND-Perception-Project/blob/master/pr2_robot/config/output.yaml)  
8. Submit a link to your GitHub repo for the project or the Python code for your perception pipeline and your output `.yaml` files (3 `.yaml` files, one for each test world).  You must have correctly identified 100% of objects from `pick_list_1.yaml` for `test1.world`, 80% of items from `pick_list_2.yaml` for `test2.world` and 75% of items from `pick_list_3.yaml` in `test3.world`.
9. Congratulations!  Your Done!

# Extra Challenges: Complete the Pick & Place
7. To create a collision map, publish a point cloud to the `/pr2/3d_map/points` topic and make sure you change the `point_cloud_topic` to `/pr2/3d_map/points` in `sensors.yaml` in the `/pr2_robot/config/` directory. This topic is read by Moveit!, which uses this point cloud input to generate a collision map, allowing the robot to plan its trajectory.  Keep in mind that later when you go to pick up an object, you must first remove it from this point cloud so it is removed from the collision map!
8. Rotate the robot to generate collision map of table sides. This can be accomplished by publishing joint angle value(in radians) to `/pr2/world_joint_controller/command`
9. Rotate the robot back to its original state.
10. Create a ROS Client for the “pick_place_routine” rosservice.  In the required steps above, you already created the messages you need to use this service. Checkout the [PickPlace.srv](https://github.com/udacity/RoboND-Perception-Project/tree/master/pr2_robot/srv) file to find out what arguments you must pass to this service.
11. If everything was done correctly, when you pass the appropriate messages to the `pick_place_routine` service, the selected arm will perform pick and place operation and display trajectory in the RViz window
12. Place all the objects from your pick list in their respective dropoff box and you have completed the challenge!
13. Looking for a bigger challenge?  Load up the `challenge.world` scenario and see if you can get your perception pipeline working there!

## [Rubric](https://review.udacity.com/#!/rubrics/1067/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  


![demo-2](https://user-images.githubusercontent.com/20687560/28748286-9f65680e-7468-11e7-83dc-f1a32380b89c.png)

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  

You're reading it!

### Exercise 1, 2 and 3 pipeline implemented
#### 1. Complete Exercise 1 steps. Pipeline for filtering and RANSAC plane fitting implemented.
Before analyzing the pointcloud to identify objects I filtered out statistical outliers using the make_statistical_outlier_filter(). I set the mean k to 2 and the scale factor to .0001 to aggressively filter out points and minimize the noise.

Once the outliers were removed I downsampled the point cloud using a voxel_grid_filter with a leaf size of .005 to make the image processing faster. Using this smaller number of points I first filtered them by only keeping the points inside the table's bounding box of z .6m to 1.1m, x .25m to 1m, and y -.5m to .5m. This prevented other tables and side bins from interfering with further filtering.

Once the bounding box selected all points that were interesting to us a RANSAC model of a plane was used to separate the table from the objects resting on the table. The max distance for this plane was set to .01 so the bottoms of objects were left mostly in tact and were not clustered as part of the table.


#### 2. Complete Exercise 2 steps: Pipeline including clustering for segmentation implemented.  

Once the objects were separated from the table, the next task was to separate them from each other. This was accomplished by using the EuclideanClusterExtraction algorithm with a cluster tolerance set to .05, min cluster size of 25, and a max cluster size of 10000. The objects were fairly spaced out on the table so the tolerances for this step did not need to be very tight.

![cluster-1](https://raw.githubusercontent.com/violas15/RoboND-Perception-Project/master/pr2_robot/scripts/clusteredObjects.png)
Here is the image of the separate clustered objects. The objects here are color coded for identification but the real colors were used moving forwards.

#### 2. Complete Exercise 3 Steps.  Features extracted and SVM trained.  Object recognition implemented.

With the objects separated from each other it was time to identify them. Color and shape's of objects were extracted by creating historgrams of colors and surface normals for each object in many different poses. These formed the trained model that the image classifier would compare against later. The success of this model can be seen by looking at the normalized confusion matrix.

![matrix](https://raw.githubusercontent.com/violas15/RoboND-Perception-Project/master/pr2_robot/scripts/normalizedConfusingMatrix.png) 
From this matrix it is clear that the model was able to correctly classify over 95% of objects in the testing set. This performance helped the performance of the overall object recognition.

With the model trained object recognition was implemented by gathering the same histograms, color and surface normals, from each of the segmented objects. These histograms were compared to the model and a prediction was made about what each object was.

### Pick and Place Setup

#### 1. For all three tabletop setups (`test*.world`), perform object recognition, then read in respective pick list (`pick_list_*.yaml`). Next construct the messages that would comprise a valid `PickPlace` request output them to `.yaml` format.

After the object recognition completed every object had a label and a corresponding set of points. To represent this in a format that could be used for a pick and place operation the information was compared to a pick list to create the yaml dictionary values. The objects centroid, calculated from all the corresponding points, the name, which arm, and which bin was the correct location for the item were combined into individual yaml dictionary entries and then combined into a list containg all of the results. The output of this operation can be seen in the .yaml files in the pr2_robot/scripts folder.

This completed the pipeline and results could be visualized using RVIZ, where the corresponding labels were published for each world.

World 1
![world-1](https://raw.githubusercontent.com/violas15/RoboND-Perception-Project/master/pr2_robot/scripts/World1.png)
3/3 objects were correctly identified.

World 2
![world-2](https://raw.githubusercontent.com/violas15/RoboND-Perception-Project/master/pr2_robot/scripts/World2.png)
4/5 objects were correctly identified. The book was labelled incorrectly as soap.

World 3
![world-3](https://raw.githubusercontent.com/violas15/RoboND-Perception-Project/master/pr2_robot/scripts/World3.png)
6/8 objects were correctly identified. The book and the glue were labelled incorrectly as soap and biscuits respectively.



The above technique worked fairly well in this situation because it was a very controlled enviornment. The matching of histograms for color and surface normals would lose accuracy in a real world situation due to changing light levels and potentially misformed packaging. By having additional images of objects to train the model this could be counteracted but the training process would continue to take an increasing amount of time. 



