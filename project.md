# Human Motion Recognition from Video

## and Its Reproduction on a 3D Avatar in Unity

_A markerless motion capture system based on computer vision, UDP communication, and real-time skeletal retargeting_

**Project thesis — Image Processing and Computer Vision 2026**

---

# Abstract

This work describes the design and implementation of a system that analyzes a video containing a person, recognizes their movements using computer vision techniques, and reproduces them in real time on a humanoid 3D avatar within the Unity game engine. Three different human pose estimation pipelines were analyzed and compared: **Scenario A, based on SAM3DBody-cpp; Scenario B, based on AlphaPose + HybrIK; and Scenario C, based on MMDetection + HybrIK**. The three solutions were evaluated in terms of their output, computational complexity, skeletal detail, and suitability for real-time animation.

A shared communication architecture based on the UDP protocol was designed. Pose data is serialized in JSON format and transmitted from the computer vision module to the Unity engine. A multithreaded receiver acquires packets without blocking rendering, while the skeletal retargeting component maps the received rotations to the bones of the Humanoid avatar and applies them through interpolation. The shared JSON format keeps the Unity side independent of the specific computer vision pipeline being used.

The system was validated through local simulation tests and by connecting the pipelines to the same receiving module, verifying the correct operation of the end-to-end chain. Possible extensions are also discussed, including the simultaneous handling of multiple people in a scene and interaction between the avatar and surrounding objects and environment.

---

# 1. Introduction

The goal of this project is to build a complete **motion capture** pipeline: starting from a video stream (webcam or file) containing a person, the system must recognize the human body's pose and faithfully reproduce its movement on a 3D avatar in Unity, in real time and without wearable sensors or physical markers.

The overall architecture is divided into three main blocks that communicate over a local network:

- **Pose acquisition and estimation (Python / C++):** a computer vision model detects the person in the frame and extracts the rotations or 3D positions of their joints.
- **Data transmission (UDP):** pose data is serialized as JSON and sent through a UDP socket to a local port monitored by Unity.
- **Reception and animation (Unity / C#):** a script receives and decodes the packet, then applies the rotations to the Humanoid avatar's bones through skeletal retargeting and spherical interpolation (Slerp).

### Overall data flow

```text
[ Video / Webcam ]
        |
        v
[ CV Model: SAM3DBody-cpp / AlphaPose+HybrIK / MMDetection+HybrIK ]
        |
        |   JSON via UDP (port 5065)
        v
[ UDPReceiver.cs ]  --background thread-->  latestJSON
        |
        v
[ AvatarController.cs ]  (LateUpdate, Slerp/Lerp)
        |
        v
[ Humanoid 3D Avatar (X-Bot) in the Unity scene ]
```

The following chapters first examine the main pose-estimation frameworks available in the literature (Chapter 2), then describe the three architectural scenarios in detail (Chapter 3), their computer vision pipelines (Chapter 4), the communication protocol (Chapter 5), Unity integration (Chapter 6), the tests performed (Chapter 7), and possible system extensions (Chapter 8).

---

# 2. State of the Art: Human Pose Estimation and Human Mesh Recovery

Recognizing human pose from images or video is a central computer vision problem, addressed through different approaches depending on the required output. Three main categories can be distinguished:

1. **2D pose estimation**, which returns keypoints on the image plane.
2. **3D pose estimation**, which returns the spatial coordinates of joints.
3. **Human Mesh Recovery**, which reconstructs the entire body surface using parametric models such as SMPL and its successors.

During the preliminary phase of the project, several existing frameworks were analyzed to identify the solution best suited to the real-time constraint and compatibility with Unity.

| Algorithm | Category | Output | Main strength |
|---|---|---|---|
| OpenPose | 2D Pose | 2D keypoints | Established, multi-person, but slow |
| AlphaPose | 2D Pose | 2D/3D keypoints | Accurate, real-time, multi-person tracking |
| MMPose | Framework | 2D/3D pose and mesh | Modular, with many available models |
| VoxelPose | Multi-view 3D | Multi-person 3D pose | Robust to occlusions, requires multiple cameras |
| HybrIK | Human Mesh Recovery | SMPL mesh | Inverse kinematics, highly accurate |
| 4D-Humans (HMR2) | Human Mesh Recovery | SMPL mesh | State of the art, requires a powerful GPU |
| SAM3DBody | Human Mesh Recovery | MHR mesh + rig | Detailed rig (~70 joints), computationally demanding |
| SAM3DBody-cpp | Real-time Mesh Recovery | MHR mesh + BVH | C++/ONNX runtime, no Python dependency |

This comparison identified three feasible strategies for the project, described in the next chapter as **Scenario A, Scenario B, and Scenario C**.

---

# 3. System Architecture and Choice of Approach

The choice of computer vision pipeline directly determines the type of data that must be transferred to Unity and how that data is used to animate the avatar.

Three distinct architectural scenarios were analyzed and tested during the project. They share the same final UDP communication stage towards Unity.

The three approaches mainly differ in how the 3D pose is obtained and in the model used to represent the human skeleton:

- **Scenario A:** directly uses SAM3DBody-cpp, which combines detection, pose estimation, and MHR rig reconstruction in a single C++ pipeline.
- **Scenario B:** uses AlphaPose together with HybrIK to obtain the 3D pose and joint rotations.
- **Scenario C:** uses MMDetection as the detector and HybrIK for 3D pose reconstruction.

## 3.1 Scenario A — SAM3DBody-cpp

In the first scenario, the entire pose-estimation pipeline is handled by **SAM3DBody-cpp**, a C++ implementation of the SAM-3D-Body model.

The system receives the video stream directly, detects the person, and reconstructs the human body using the **MHR (Momentum Human Rig)** model.

The output includes an articulated rig with significantly more joints than the SMPL skeleton used by HybrIK, allowing hands, fingers, and feet to be represented in greater detail.

The pipeline's output is converted into the format used by the project and sent to Unity over UDP. Since the model directly provides joint-orientation information, Unity can apply the received rotations to the corresponding bones without requiring a generic inverse-kinematics solver.

---

## 3.2 Scenario B — AlphaPose + HybrIK

In the second scenario, **AlphaPose is used together with HybrIK**.

The pipeline combines two components:

- **AlphaPose**, used for video processing and person detection;
- **HybrIK**, used for 3D body reconstruction and the calculation of joint rotations.

The advantage of this configuration is that person detection is separated from the subsequent 3D pose reconstruction.

HybrIK provides 3D information and rotation matrices for the **24 SMPL joints**. The rotations can then be converted into the format used by Unity and transmitted directly over UDP.

This scenario is an intermediate solution between the integrated SAM3DBody-cpp pipeline and the pipeline based on a general-purpose detector.

---

## 3.3 Scenario C — MMDetection + HybrIK

In the third scenario, person detection is handled by **MMDetection**, while 3D pose estimation is performed by **HybrIK**.

Unlike Scenario B, MMDetection is used specifically as a detector: it identifies the person in the frame and returns a **bounding box**, which is then used to crop the region of interest supplied to HybrIK.

The bounding box is also stabilized through temporal smoothing, and detection does not necessarily run on every frame, reducing computational load.

HybrIK receives the cropped person and returns:

- rotation matrices for the 24 SMPL joints;
- 3D joint coordinates;
- the skeleton's root translation.

The rotations are converted into the format used by Unity and inserted into the same JSON schema shared by the other scenarios.

---

## 3.4 Comparison of the Three Scenarios

| Scenario | Pipeline | Detection | Pose / Mesh Recovery | Output used by Unity |
|---|---|---|---|---|
| **A** | SAM3DBody-cpp | Integrated into the pipeline | SAM-3D-Body / MHR | MHR rig rotations |
| **B** | AlphaPose + HybrIK | AlphaPose | HybrIK / SMPL | Rotations + 3D joints |
| **C** | MMDetection + HybrIK | MMDetection | HybrIK / SMPL | Rotations + 3D joints |

All three scenarios converge on the same communication and animation module: pose data is serialized as JSON, sent via UDP, and then processed by Unity.

This design makes it possible to compare the different computer vision pipelines directly while keeping the final part of the system unchanged.

---

# 4. Computer Vision Pipelines Analyzed

Three pipelines were analyzed for pose extraction, corresponding to the three architectural scenarios described in the previous chapter.

They share the same overall goal—obtaining sufficiently rich information about human pose to animate a 3D avatar in Unity—but use different models and skeletal representations.

---

## 4.1 SAM3DBody-cpp — Scenario A

SAM3DBody-cpp is **Scenario A** and is the most integrated pipeline among those analyzed.

It is a fully C++ reimplementation of Meta's SAM-3D-Body model, designed to run without dependencies on a Python runtime.

The pipeline includes:

1. person detection;
2. visual feature extraction;
3. pose-parameter estimation;
4. application of the parameters to the MHR model.

The system uses a more detailed body representation than the SMPL skeleton, also allowing the joints of the hands and fingers to be represented.

In this project, the C++ program directly integrates a UDP socket into the pipeline's main loop. For each detected person, a JSON packet is built containing joint rotations, 3D keypoint coordinates, and the root position.

The packet is then sent to Unity through `sendto()`.

```cpp
int udp_sock = udp_open(c.udp_ip, c.udp_port, udp_addr);

for (int i = 0; i < (int)results.size(); ++i)
    send_pose_udp(
        udp_sock,
        udp_addr,
        i,
        results[i],
        bvh_writer,
        bvh_joint_names
    );

// inside send_pose_udp():

json << "\"unity_rotations_deg\":{ ... for each BVH joint ...";
json << "\"root_position\":{...}";

sendto(
    sock,
    payload.c_str(),
    payload.size(),
    0,
    (sockaddr*)&addr,
    sizeof(addr)
);
```

The main advantage of this approach is therefore the availability of a detailed rig and the ability to run the entire pipeline in C++, reducing the overhead associated with the Python interpreter.

The main limitation is its greater computational and integration complexity compared with pipelines based on SMPL models.

---

## 4.2 AlphaPose + HybrIK — Scenario B

**Scenario B** combines AlphaPose and HybrIK.

AlphaPose is used to process the video and detect the person, while HybrIK transforms the resulting information into a 3D representation of the human body.

AlphaPose uses a **top-down** pipeline: a detector identifies the people in a frame, and the pose-estimation network analyzes each detected region.

The output includes the person's keypoints and their confidence scores. This information is used as the basis for subsequent 3D processing.

HybrIK completes the pipeline by reconstructing the parametric **SMPL 3D skeleton**.

The model uses inverse kinematics to obtain joint rotations consistent with the body's 3D configuration.

In this project, the rotation matrices for the **24 SMPL joints** are converted directly into unit quaternions `(x, y, z, w)` and organized by joint name. Direct conversion avoids an intermediate representation using Euler angles and explicitly preserves the fourth component, `w`, which is necessary to represent orientation correctly in Unity.

The main advantage of Scenario B is that it combines an established detection and pose-estimation system such as AlphaPose with a Human Mesh Recovery model such as HybrIK.

The resulting output is richer than 2D keypoints alone, as it includes both 3D information and joint orientations.

The drawback is the need to run and coordinate two processing components, increasing pipeline complexity compared with a fully integrated system.

---

## 4.3 MMDetection + HybrIK — Scenario C

**Scenario C** uses MMDetection as the detector and HybrIK as the Human Mesh Recovery model.

In this case, the two modules have clearly separated responsibilities:

- **MMDetection** detects the person in the image using a bounding box;
- **HybrIK** receives the cropped region and reconstructs the 3D pose.

In the developed program, MMDetection is initialized with a **Faster R-CNN configuration using a ResNet-50 backbone and FPN**, together with a pretrained checkpoint.

During frame processing, detections belonging to the person class are considered, and a confidence threshold is applied to remove less reliable detections.

To improve the stability of the region of interest, the bounding box undergoes **temporal smoothing**.

Detection can also run at regular intervals rather than on every frame. In intermediate frames, the previously filtered bounding box is retained, reducing the number of detector inferences and improving overall performance.

The detected region is then preprocessed according to HybrIK's input requirements and passed to the model.

HybrIK returns:

- rotation matrices for the 24 SMPL joints;
- 3D joint coordinates;
- root translation.

The rotation matrices are converted into quaternions and then into the values used by the Unity module.

```python
JOINT_NAMES = [
    "Pelvis", "L_Hip", "R_Hip", "Spine1",
    "L_Knee", "R_Knee", "Spine2", ...
    "L_Wrist", "R_Wrist"
]


def rotmats_to_quat_dict(theta_mats):
    r = R.from_matrix(theta_mats)
    quats = r.as_quat()

    return {
        name: q.tolist()
        for name, q in zip(JOINT_NAMES, quats)
    }
```

The result is finally inserted into the same JSON schema used by the other scenarios, containing:

- `person_id`;
- `timestamp`;
- `unity_rotations_deg`;
- `joint_xyz_3d`;
- `root_position`.

The packet is then sent to Unity over UDP on port 5065.

The main advantage of Scenario C is **modularity**: MMDetection can be used exclusively for detection, leaving 3D pose reconstruction to HybrIK.

Separating the two tasks also makes it possible to adjust the detector and Human Mesh Recovery model parameters independently.

Compared with Scenario B, the main difference is therefore the module used to detect the person. AlphaPose provides a pipeline specifically designed for pose estimation, whereas MMDetection acts as a general-purpose detector to obtain the bounding box subsequently processed by HybrIK.

---

# 5. UDP Communication Protocol

The connection between the computation environment (Python/C++) and the rendering environment (Unity) uses the **UDP (User Datagram Protocol)**.

This choice is motivated by the system's latency requirements. Unlike TCP, which guarantees ordered packet delivery through retransmission, UDP sends data continuously without waiting for acknowledgements.

In real-time motion capture, an "old" pose frame has no value once a newer one is available.

Occasional packet loss is therefore acceptable and preferable to blocking the pipeline while waiting for retransmission, helping keep latency low.

## 5.1 JSON Packet Structure

All three pipelines share the same packet schema, keeping the Unity-side receiver independent of the data source:

```json
{
  "person_id": 0,
  "timestamp": 1784979327.127,
  "unity_rotations_deg": {
    "Pelvis": { "x": 0.12, "y": -0.08, "z": 0.31, "w": 0.94 },
    "L_Elbow": { "x": 0.05, "y": 0.21, "z": -0.12, "w": 0.96 }
  },
  "joint_xyz_3d": [0.0, 0.0, 0.0, 0.035, -0.007, 0.015],
  "root_position": {
    "x": -0.088,
    "y": -0.132,
    "z": -4.544
  }
}
```

### Main fields

- **`person_id`**: identifier assigned by tracking, allowing each packet to be associated with the correct avatar in multi-person scenarios.
- **`timestamp`**: the time at which the frame was processed by the model.
- **`unity_rotations_deg`**: a field retained for compatibility with the existing protocol; for the HybrIK pipelines, it contains each joint's local quaternion in `x,y,z,w` format, used to move the avatar.
- **`joint_xyz_3d`**: relative joint positions in 3D space, useful for debugging, visualization, or alternative approaches.
- **`root_position`**: the skeleton root position relative to the camera, used to translate the entire avatar in the scene.

The three pipelines mainly differ in the set of joint names they use.

HybrIK uses the 24-joint SMPL schema, whereas SAM3DBody-cpp uses a more extensive schema based on the MHR/BVH rig.

---

# 6. Unity Integration

## 6.1 Humanoid Avatar Setup

The selected 3D model is **X-Bot**, available for free on Mixamo and downloaded as a binary FBX in a T-pose.

The T-pose is the reference pose needed to calculate local rotations at rest correctly.

After importing the model into the Unity project, the `Animation Type` in the Inspector's **Rig** tab was set to **Humanoid**.

This allows Unity to automatically map the model's specific bone hierarchy to its internal `HumanBodyBones` standard, making the skeleton independent of the mesh's proportions and compatible with animation scripts based on that standard.

---

## 6.2 UDPReceiver.cs — Network Reception

The `UDPReceiver.cs` component listens on the local UDP port (`5065`) using a dedicated background thread, separate from Unity's main rendering thread.

This prevents frame-rate drops caused by the blocking wait in the following function:

```csharp
UdpClient.Receive()
```

The latest received packet is stored in a shared string variable (`latestJSON`), protected by a `lock` to prevent race conditions between the network thread and the main loop.

```csharp
private void ReceiveData()
{
    IPEndPoint localEndPoint =
        new IPEndPoint(IPAddress.Parse("127.0.0.1"), port);

    client = new UdpClient(localEndPoint);

    IPEndPoint remoteEndPoint =
        new IPEndPoint(IPAddress.Any, 0);

    while (running)
    {
        byte[] data = client.Receive(ref remoteEndPoint);

        string text =
            Encoding.UTF8.GetString(data);

        lock (lockObject)
        {
            latestJSON = text;
        }
    }
}
```

---

## 6.3 AvatarController.cs — Retargeting and Animation

The `AvatarController.cs` component runs in `LateUpdate()` and deserializes the JSON using the Newtonsoft Json library.

The data is organized into helper classes:

- `JointData`;
- `MotionPayload`.

The received rotations are then applied to the corresponding bones of the Humanoid avatar.

Spherical interpolation using `Quaternion.Slerp` is used to achieve smooth movement.

### SMPL → Unity Humanoid Mapping

| SMPL Joint | Unity Humanoid Bone | Biomechanical description |
|---|---|---|
| Pelvis | Hips | Pelvis / center of mass |
| Spine1 / Spine2 / Spine3 | Spine / Chest / UpperChest | Spine and chest |
| Neck / Head | Neck / Head | Neck and head |
| L_Shoulder / R_Shoulder | LeftUpperArm / RightUpperArm | Upper arm |
| L_Elbow / R_Elbow | LeftLowerArm / RightLowerArm | Forearm |
| L_Wrist / R_Wrist | LeftHand / RightHand | Wrist / hand |
| L_Hip / R_Hip | LeftUpperLeg / RightUpperLeg | Thigh |
| L_Knee / R_Knee | LeftLowerLeg / RightLowerLeg | Lower leg |
| L_Ankle / R_Ankle | LeftFoot / RightFoot | Foot |

The retargeting loop can be represented as follows:

```csharp
foreach (var item in motionData.unity_rotations_deg)
{
    if (boneMap.TryGetValue(
        item.Key,
        out Transform boneTransform))
    {
        Quaternion targetRotation =
            Quaternion.Normalize(
                new Quaternion(
                    item.Value.x,
                    item.Value.y,
                    item.Value.z,
                    item.Value.w
                )
            );

        boneTransform.localRotation =
            Quaternion.Slerp(
                boneTransform.localRotation,
                targetRotation,
                Time.deltaTime * smoothing
            );
    }
}
```

For Scenarios B and C, the `unity_rotations_deg` field retains this name for compatibility with the historical JSON schema, but its values are no longer angles in degrees: they are unit quaternions `(x, y, z, w)`. The fourth component, `w`, cannot be omitted. Treating the three quaternion values `x`, `y`, and `z` as Euler angles, or as a quaternion without `w`, produces incorrect joint orientations.

The `root_position` field is instead applied using `Vector3.Lerp` to the local position of the `Hips` bone, translating the entire avatar in space according to the distance estimated from the camera.

The `smoothing` parameter, exposed in the Inspector, controls the responsiveness of interpolation:

- low values → smoother movement but greater perceived delay;
- high values → lower perceived latency but less smooth movement.

---

# 7. Testing and Validation

The pipeline was validated in two stages.

## 7.1 Local Simulation Test

During an initial internal stage, a temporary `UDPSimulator` component was used to send test JSON packets locally to port `5065`.

This made it possible to test the following components in isolation:

- network reception;
- JSON deserialization;
- joint mapping;
- skeletal retargeting;
- avatar animation.

The X-Bot avatar responded correctly by moving its head and arms in real time, confirming that the pipeline worked as intended:

```text
UDP → JSON parsing → retargeting → Humanoid animation
```

## 7.2 End-to-End Testing

In a second stage, the three computer vision pipelines:

- SAM3DBody-cpp;
- AlphaPose + HybrIK;
- MMDetection + HybrIK;

were connected to the same Unity receiver.

The test verified that, given the same packet format, the avatar responded to data received from the different sources.

This confirms the value of using a **shared JSON schema** across the different implementations.

### 7.2.1 Rotation Retargeting Correction

During testing of the **AlphaPose + HybrIK** pipeline, an inconsistency was found in the application of rotations to Unity's Humanoid model. The overall retargeting structure was not rewritten: the project already used a calibration stage, calculation of the delta relative to the initial pose, application of the bind pose, and rotation smoothing in the other pipelines as well.

Instead, the correction concerned how the rotations produced by HybrIK are interpreted by the rig used in Unity. HybrIK returns the rotations of the 24 SMPL joints as \(3\times3\) matrices, which are converted directly into quaternions and transmitted in `x,y,z,w` format. In the AlphaPose Unity controller, the received quaternion is therefore used directly, avoiding an intermediate conversion to Euler angles.

The issue found during testing was not simply the matrix-to-quaternion conversion, but the difference between the reference coordinate system of the skeleton rotations produced by HybrIK and the local axes of Unity's Humanoid rig. For this reason, specific retargeting corrections were introduced for certain joints, transforming quaternion components differently from the general conversion. A correction to the root/Pelvis orientation was also introduced to compensate for the global axis misalignment.

Calibration makes it possible to use the initial pose as a reference. For each joint, the delta between the current rotation and the rotation recorded during calibration is calculated, and this delta is applied to the Unity bone's bind rotation. In this way, the user's initial pose is not transferred as an absolute avatar rotation; instead, it defines the reference against which subsequent movements are applied.

It is important to distinguish this correction from the general features already present in the project. The controller used for **MMDetection + HybrIK** also receives `x,y,z,w` quaternions directly and applies calibration, the delta relative to calibration, bind pose, and smoothing, but it does not use the axis-specific corrections introduced in the AlphaPose controller. The **SAM3DBody-cpp** controller, on the other hand, receives rotations in the format expected by its own pipeline during testing and converts them on the Unity side using `Quaternion.Euler(...)`. The changes described in this section are therefore specific to the retargeting stage of the AlphaPose + HybrIK pipeline, rather than a general rewrite of Unity's animation system.

The following videos demonstrate the tests carried out with the three methods:

**SAM3DBody-cpp test running on Linux in CPU mode:**

![SAM test](project_files/video/Sam_test.gif)

**AlphaPose test running on macOS in CPU mode:**

![AlphaPose test](project_files/video/Alphapose_test.gif)

**MMDetection test running on Windows + WSL in GPU mode:**

![MMDetection test](project_files/video/MMDetection_test.gif)

### 7.2.2 Hip and Leg Corrections

#### Model Selection: MMDetection + HybrIK

After further tests on all three models, it emerged that the hips are unlocked by default in the **MMDetection + HybrIK** pipeline. However, the initial movement was unsatisfactory: displacements were difficult to control, the legs appeared stiff, and the movements were not sufficiently accurate.

![Initial MMDetection test](project_files/video/MMDetection_hip_pre.gif)

![Final MMDetection test](project_files/video/MMDetection_hip_fix.gif)

To improve the behavior, changes were introduced in `AvatarController.cs` while leaving the computer vision pipeline unchanged.

#### Leg Mirroring Mode

A configuration section dedicated to leg corrections was added:

```csharp
[Header("Leg Mirroring Mode")]
public bool mirrorLegs = false;
public bool invertLegRotationX = false;
public bool invertLegRotationY = false;
public bool invertLegRotationZ = false;
public Dictionary<string, string> legMirrorMap;
```

The mapping associates the left leg bones with their right-side counterparts and vice versa, including hips, knees, ankles, and feet. Inversions can be configured separately for each quaternion axis to correct orientation differences between the source skeleton and the avatar.

#### Rotation Dead Zone and Smoothing

To reduce small oscillations and unwanted movements, a dedicated dead zone was introduced for leg rotations. Changes below the threshold are ignored, while larger changes continue to be applied.

```csharp
public float legRotationDeadZone = 0.05f;
public float legSmoothing = 14f;
public float legFastSmoothing = 35f;
```

Adaptive smoothing attenuates smaller movements without making the response to larger changes excessively slow.

#### Leg Movement Amplification

Since the estimated rotations may not be very noticeable on the 3D model, an amplification factor was added:

```csharp
public float legSwingBoost = 1.6f;
```

Amplification is applied to the change relative to the calibration rotation, making leg movements more visible while retaining the initial pose as the reference.

#### Procedural Pelvis Sway

To make the overall movement less stiff, a procedural pelvis sway was added:

```csharp
public bool enableHipSway = true;
public float hipSwayAmount = 1.2f;
public float hipSwaySpeed = 0.25f;
```

The `GetHipSwayOffset` function calculates the offset and applies it when the bone being processed is `Pelvis`. This adds a slight sway to the pelvis, helping the animation look more natural.

#### Result

The changes to `AvatarController.cs` improve control over root movement through scaling, limits, and smoothing. Leg movement becomes easier to control through mirroring, a dedicated dead zone, and rotation amplification, while procedural pelvis sway contributes to a more natural animation.

---

## 7.3 Limitations

As shown in the videos, the results obtained on the Unity avatar are often only approximations of the actual poses captured by the webcam. There are several reasons for this, and they affect different stages of the pipeline.

The first reason is the inherent difficulty of estimating a person's unique 3D pose from a monocular webcam. SAM3DBody-cpp and HybrIK attempt to resolve these ambiguities by regressing the complexity of human 3D movement towards more plausible poses. Other difficulties arise from image sampling and possible occlusions, caused either by objects in the scene or by the person's own body hiding or crossing limbs, making neural-network pose estimation more difficult.

From the perspective of Unity Humanoid avatar animation, it is important to consider that the SMPL and SAM3DBody skeletons have a different bone hierarchy from the Humanoid skeleton. Copying rotations between different rigs can therefore introduce animation distortions. Further distortions can occur if the estimator's reference pose does not perfectly match the calibration pose of the Unity Humanoid; in that case, local rotations would be reproduced with a systematic offset.

Finally, there are real-time system constraints. The computer vision pipeline's frame rate is much lower than Unity's refresh rate, forcing the avatar to follow the estimated poses with a noticeable delay.

The estimator's low frame rate can also cause motion peaks to be missed. Unity must interpret poses that are far apart in time; if the person performs a rapid movement in between, it may not be sampled, forcing Unity to approximate it much more slowly than it actually occurred.

Smoothing performed by Unity to compensate for estimation noise also contributes to this effect. This approach sacrifices some fidelity in favor of animation stability.

# 8. System Extensions

As mentioned in the project abstract, two extensions to the base architecture were designed, focusing respectively on handling multi-person scenes and enabling physical interaction between the avatar and its environment.

---

## 8.1 Handling Multiple People in the Scene

When the detection module identifies several people at the same time, the JSON stream sends packets with distinct `person_id` values:

```text
0, 1, 2, ...
```

The `MultiAvatarManager.cs` component maintains a:

```csharp
Dictionary<int, GameObject>
```

which associates each identifier with its dynamically instantiated avatar.

When a new `person_id` first appears, a new avatar is created:

```csharp
if (!avatarsInScene.ContainsKey(id))
{
    GameObject newAvatar =
        Instantiate(
            avatarPrefab,
            Vector3.zero,
            Quaternion.identity
        );

    avatarsInScene.Add(id, newAvatar);
}

avatarsInScene[id]
    .GetComponent<AvatarPoseController>()
    .UpdatePose(packet);
```

A second dictionary can store the timestamp of the latest packet received for each person.

If a person is not updated for longer than a configurable threshold, the corresponding avatar is removed from the scene.

---

## 8.2 Physical Interaction with Objects and the Environment

Since the avatar is moved by directly forcing the rotation and position of its bones, by default it does not collide with the environment like a conventional physical body.

To enable interaction, Unity's built-in **Ragdoll** tool can be used to add `Capsule Collider` and `Rigidbody` components to the main limbs.

The architecture includes:

- bone `Rigidbody` components set to **Kinematic**;
- colliders on the main limbs;
- a `Sphere Collider` on the hand in `Trigger` mode;
- identification of interactable objects using the `\"Grabbable\"` tag.

To grab objects, when the hand comes into contact with a `Grabbable` object and the pose-estimation data indicates a configuration consistent with a closed hand, the object can be made a child of the hand.

The object's physics is temporarily disabled:

```text
Closed hand
     |
     v
Trigger Collider
     |
     v
Grabbable object
     |
     v
Parenting to the hand
     |
     v
Physics disabled
```

When the hand opens again, the object is released and physics is re-enabled.

Combining kinematic bones with a parenting-based grabbing script makes it possible, with relatively little implementation effort, to create a convincing illusion of physical interaction between the person captured by the webcam and objects in the Unity scene.

---

# 9. Conclusions

The project made it possible to implement and compare **three computer vision strategies** for real-time animation of a 3D avatar:

1. **Scenario A — SAM3DBody-cpp**
2. **Scenario B — AlphaPose + HybrIK**
3. **Scenario C — MMDetection + HybrIK**

**Scenario A** is the most integrated solution, since SAM3DBody-cpp combines the different processing stages and produces a detailed MHR rig.

**Scenario B** combines AlphaPose with HybrIK, using the former for person analysis and the latter for 3D reconstruction and rotation estimation.

**Scenario C** clearly separates detection and pose recovery, using MMDetection to obtain the bounding box and HybrIK to reconstruct the SMPL skeleton.

Among the analyzed pipelines, SAM3DBody-cpp stands out for its detailed rig, including information about hands and fingers, and for running in C++ without a dependency on the Python runtime.

The HybrIK-based pipelines, on the other hand, benefit from using the 24-joint SMPL model and directly providing joint-orientation information, making them particularly suitable for direct retargeting to a Humanoid avatar.

The UDP protocol with a shared JSON payload across the different sources proved to be an effective architectural choice, providing low latency and decoupling the vision module from the rendering module in Unity.

Possible future developments include:

- introducing a temporal filter to further reduce jitter;
- more robust handling of occlusions;
- multi-person support;
- physical interaction with objects;
- multi-camera calibration for more robust 3D reconstruction.

---

# Appendix A — Repositories and References

| Project | Repository |
|---|---|
| AlphaPose | `github.com/MVIG-SJTU/AlphaPose` |
| MMDetection | `github.com/open-mmlab/mmdetection` |
| HybrIK | `github.com/jeffffffli/HybrIK` |
| SAM3DBody-cpp | `github.com/AmmarkoV/SAM3DBody-cpp` |

---

# Appendix B — Extended List of Evaluated Pose/Mesh Estimation Frameworks

| Algorithm | Category | Output | Main advantage | Main limitation |
|---|---|---|---|---|
| OpenPose | 2D Pose | 2D keypoints | Established, multi-person | Slow, outdated |
| AlphaPose | 2D Pose | 2D/3D keypoints | Accurate, real-time | More complex pipeline when combined with HybrIK |
| MMPose | Framework | 2D/3D pose and mesh | Modular | Not a single algorithm |
| VoxelPose | Multi-view 3D | 3D pose | Robust to occlusions | Requires multiple calibrated cameras |
| PoseFormer | 3D Pose | 3D pose | Uses temporal information | Requires accurate 2D poses |
| MotionBERT | 3D Motion | Temporal 3D pose | Well suited to motion capture | Computationally demanding |
| ROMP | Mesh Recovery | SMPL mesh | Real-time, simple | Lower accuracy |
| PARE | Mesh Recovery | SMPL mesh | Robust to occlusions | Slower than ROMP |
| HybrIK | Mesh Recovery | SMPL mesh | Inverse kinematics | Requires the SMPL model |
| CLIFF | Mesh Recovery | SMPL mesh | Handles perspective | Monocular |
| VIBE | Video Mesh Recovery | Mesh + motion | Uses temporal information | Not real-time |
| 4D-Humans (HMR2) | Human Mesh Recovery | SMPL mesh | State of the art | Requires a powerful GPU |
| SAM3DBody | Mesh Recovery | MHR mesh + rig | Detailed rig | Computationally demanding, research-oriented |
| SAM3DBody-cpp | Real-time Mesh Recovery | MHR mesh + BVH | Real-time, no Python | Unofficial |
| MMHuman3D | Framework | Mesh, 3D pose | Includes many models | Complex framework |
