# TurtleBot3 Human-Following Robot

> **YOLO · MediaPipe · Finite State Machine · ROS 2**


https://github.com/user-attachments/assets/6a32c3fb-03b1-4017-95bf-57bd3f3b3c3a


<img width="1651" height="646" alt="2" src="https://github.com/user-attachments/assets/a8432db0-f733-4fd1-ba4c-9c5459d67267" />

  



> **TurtleBot3 + ROS 2 Humble + MediaPipe Pose + Keypoint-Based Action Classification + FSM**
BuddyBot is a service-oriented robot system that uses a camera to understand a user's position and body action, then converts that understanding into robot behaviour.

The project focuses on two core capabilities:

- **YOLO-based person tracking** for follow and distance control
- **MediaPipe-based action understanding** for recognizing user intent

Those two perception results are combined by an **FSM (Finite State Machine)** to control:

- robot following and approach behaviour
- retreat behaviour
- platform height adjustment

---

## 1. What This Project Does

BuddyBot is not only a person-following robot. It is designed for service interaction.

That means the robot should behave differently depending on what the user is doing:

- if the user waves, the robot starts or stops service
- if the user walks, the robot follows
- if the user sits, the robot moves closer
- if the user reaches out, the robot moves even closer
- if the user stands up again, the robot restores a more comfortable distance

So the central idea of the project is:

> **Track the person with YOLO, understand the person's action with MediaPipe and a trained classifier, and use those results to drive service behaviour.**

---

## 2. Full System Flow

```mermaid
flowchart LR
    A[Camera Stream] --> B[YOLO Person Detection]
    A --> C[MediaPipe Pose]
    B --> D[Target Center and Box Size]
    C --> E[33 Body Keypoints]
    E --> F[Keypoint Feature Vector]
    F --> G[Trained Action Classifier]
    G --> H[Predicted Human Action]
    D --> I[FSM Decision Layer]
    H --> I
    I --> J[/cmd_vel: Base Motion]
    I --> K[/gix_controller/joint_trajectory: Platform Height]
```

This one diagram summarizes the whole project:

1. the camera provides the input
2. YOLO finds and tracks the target person
3. MediaPipe extracts the user's pose
4. the trained classifier predicts the user's action
5. the FSM combines target position and action result
6. the robot moves and adjusts its service height

---

## 3. How YOLO Is Used for Following

YOLO is used to answer one basic question:

> **Where is the target person, and how far are they from the robot?**

### 3.1 Detection Logic

For each camera frame:

1. run YOLO person detection
2. keep only detections of class `person`
3. choose the largest person bounding box as the service target

The reason for choosing the largest box is simple:

- in a service scenario, the closest visible person is usually the one interacting with the robot
- the largest bounding box is therefore used as the current target

### 3.2 What Is Extracted from YOLO

Once the target bounding box is selected, the robot uses:

- **bounding-box center** -> to know whether the person is left or right in the image
- **bounding-box width / size** -> as a rough distance cue

This gives two pieces of information needed for movement:

- **direction control**: how the robot should turn to face the user
- **distance control**: whether the robot should move forward, stop, or retreat

### 3.3 How Following Works

The robot uses the target center to keep the user near the image center.

- if the target is left of center, the robot turns left
- if the target is right of center, the robot turns right
- if the target is near the image center, the robot keeps heading

At the same time, the robot uses bounding-box size to estimate whether the target is too far or too close.

- if the box is too small, the target is far away -> move forward
- if the box reaches the desired range -> stop advancing
- if the box is too large, the target is too close -> stop or retreat

This is the core of follow behaviour:

```text
camera -> YOLO -> target box
       -> box center -> heading control
       -> box size   -> distance control
```

### 3.4 Why YOLO Works Well Here

YOLO is a good fit for this project because:

- it detects people in real time
- it is simple to deploy
- it gives both target location and target size
- it does not require extra depth hardware for basic service interaction

So in BuddyBot, YOLO provides the **tracking and follow backbone**.

---

## 4. How Human Actions Are Recognized

Tracking alone is not enough. The robot also needs to know:

> **What is the user doing right now?**

To answer that, BuddyBot uses **MediaPipe Pose + a trained action classifier**.

The main actions in the project are:

- `Wave`
- `Reach Out`
- `Sitting`
- `Standing`
- `Walk`

These actions are not recognized from raw images directly. Instead, the system first extracts pose keypoints and then classifies the action from those keypoints.

---

## 5. How MediaPipe Is Used

MediaPipe Pose is used as the pose-estimation front end.

For each frame, MediaPipe extracts **33 human body keypoints**, including:

- shoulders
- elbows
- wrists
- hips
- knees
- ankles

If only 2-D coordinates are used, then each frame becomes:

- 33 keypoints
- each keypoint has `(x, y)`
- total feature length = **66**

So instead of feeding the classifier a full image, BuddyBot feeds it a **66-dimensional pose feature vector**.

That is important because it removes a lot of irrelevant information such as:

- background clutter
- clothing appearance
- lighting variation
- camera noise

In other words:

MediaPipe turns an image frame into a clean body-structure representation, and the classifier learns actions from that structure.

---

## 6. How the Action Model Is Trained

This is the training story of the project.

### 6.1 Step 1: Record Action Videos

First, videos are recorded for each target action:

- wave
- reach out
- sitting
- standing
- walking

Each action is recorded multiple times so the model can see natural variation in:

- body shape
- motion speed
- small pose differences
- viewpoint changes

### 6.2 Step 2: Convert Videos into Pose Samples

The recorded videos are processed frame by frame.

For each valid frame:

1. MediaPipe extracts 33 body keypoints
2. the `(x, y)` coordinates are read out
3. all keypoints are flattened into one feature vector
4. the vector is paired with the action label

So the training sample is not an image.  
It is:

```text
pose feature vector + action label
```

### 6.3 Step 3: Build the Dataset

All labeled pose samples are collected into dataset files such as:

- `pose_data.csv`
- `motion_dataset.npz`

These files are the processed training data used by the classifier.

### 6.4 Step 4: Train the Classifier

The classifier is trained to map:

- **input**: 66-D keypoint vector
- **output**: action category

Its job is to learn which pose patterns correspond to which actions.

For example:

- certain upper-body patterns correspond to `Wave`
- forward arm extension corresponds to `Reach Out`
- lower-body bending patterns correspond to `Sitting`
- upright posture patterns correspond to `Standing`

After training, the learned weights are saved as:

- `pose_model.pth`

So the full training chain is:

```text
action videos
  -> MediaPipe pose extraction
  -> keypoint coordinate vectors
  -> labeled dataset
  -> classifier training
  -> trained action model
```

---

## 7. How Action Recognition Runs Online

During live robot operation, the same idea is used online.

For each frame:

1. capture the current image
2. run MediaPipe Pose
3. extract the same 66-D feature vector used during training
4. send that vector into the trained classifier
5. get the predicted action label
6. publish the result to the robot control logic

So the online inference chain is:

```text
camera frame
  -> MediaPipe keypoints
  -> feature vector
  -> trained classifier
  -> predicted action
```

This is why the training pipeline and runtime pipeline are consistent:

- offline: videos are converted into pose features for training
- online: live frames are converted into pose features for prediction

---

## 8. How the FSM Uses Tracking and Action Results

The FSM is the decision layer of BuddyBot.

It combines:

- **YOLO result** -> where the person is and how far away they are
- **action classification result** -> what the person is doing

Then it decides what the robot should do next.

Main states:

- `IDLE`
- `FOLLOW`
- `APPROACH_60`
- `APPROACH_30`
- `APPROACH_20`
- `RETREAT_60`

Typical logic:

- first `Wave` -> start service
- second `Wave` -> stop service
- `Walk` -> follow at a normal distance
- `Standing` -> keep standard service distance
- `Sitting` -> move closer and lower platform
- `Reach Out` while sitting -> move to the closest interaction distance
- stand after sitting -> retreat and restore height

This is why BuddyBot is not just detecting or classifying.  
It is using those perception results to produce meaningful service behaviour.

---



## 9. System Architecture

The system is composed of four ROS 2 nodes that communicate over topics:


| Node | File | Subscribes | Publishes |
|---|---|---|---|
| `BiggestPersonTracker` | `tracker.py` | `/image_raw/compressed` | `/target_person_pos` |
| `HumanActionPublisher` | `action_publisher.py` | `/image_raw/compressed` | `/human_action` |
| `FollowerFSMV2` | `follower_fsm_v2.py` | `/image_raw/compressed`, `/human_action` | `/cmd_vel` |
| `PlatformHeightController` | `platform_ctrl.py` | `/human_action` | `/gix_controller/joint_trajectory` |

---


## 10. Main Artifacts

- `pose_model.pth` -> trained action classifier
- `pose_data.csv`, `motion_dataset.npz` -> processed pose dataset
- `assets/design.png`, `assets/FSM.png` -> design illustrations

These artifacts show both sides of the project:

- the **training pipeline** for learned action recognition
- the **runtime pipeline** for deployed robot service behaviour

---

## 11. Installation

Requirements:

- Ubuntu 22.04
- ROS 2 Humble
- Python 3.10+
- TurtleBot3 or similar mobile base
- camera input

Install dependencies:

```bash
pip install ultralytics mediapipe opencv-python numpy torch torchvision
```

Build the ROS 2 package:

```bash
colcon build --packages-select buddybot
source install/setup.bash
```

---

## 12. Running

Launch the ROS 2 system:

```bash
ros2 launch buddybot launch.launch.py
```

Run the learned action-recognition pipeline:

```bash
python3 buddybot/model_gesture.py
```

For manual behaviour testing:

```bash
ros2 run buddybot manual
```

---

## 13. Summary

BuddyBot is a service-oriented human-robot interaction system with two perception backbones:

- **YOLO** for target tracking and follow control
- **MediaPipe + trained classifier** for human action recognition

Those outputs are fused by an **FSM** to generate:

- follow behaviour
- approach and retreat behaviour
- height-adjustable service behaviour

So the full logic of the project is:

**camera -> YOLO tracking + MediaPipe pose -> trained action classification -> FSM -> robot service action**
