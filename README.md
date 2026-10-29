# Markerless Motion Capture: Analysis and Comparison of Computer Vision Models for 3D Animation in Unity

_Alessio Giorgetti s345637_  
_Martina Pasquinelli s363700_  
_Valentina Giordano s364229_

---

This repository documents the analysis and development of an Image Processing project for virtual animation through *motion capture* (*"A Markerless Motion Capture System Based on Computer Vision"*).

The work was carried out as follows:

- Research and analysis of *human detection* models, including those suggested via email and additional models found through online research.
- Selection of the most efficient and feasible models to download and test.
- Definition of a workflow pipeline for sharing output data and animating an avatar in Unity in real time.
- Implementation of the code for the three selected models.
- Creation of the Unity project and implementation of the C# code required to receive data over the network.
- Writing of the report and testing.
- Extension of the report to support macOS.
- Final correction to enable the avatar to move in three-dimensional space.

The report describes all the steps carried out and the analyses performed, along with several proposed extensions for potential future development of the project.

Additionally, an attempt was made to consolidate the project into a single application capable of analysing and comparing the three downloaded models on the same operating system. The aim was to evaluate their performance and characteristics using the same system architecture. This document also provides guidance on installing and running the project on macOS.

The additional document `correzione_tesina.md` contains a proposed correction to enable the virtual avatar to move in three-dimensional space.

The repository is organised as follows:

```text
Markerless Motion Capture/projects_files
│
├── 1_alphapose+Hybrik/            # Code for the first model
│
├── 2_mmDec+Hybrik+Unity/          # Code for the second model
│
├── 3_sam+Unity/                   # Code for the third model
│
├── materiale/                     # All materials useful for the research,
│                                  # including those not directly used
│                                  # in the final implementation
│
├── test_vale/                     # Folder containing report extensions
│
├── Unity/                         # Unity project files
│
└── video/                         # Test videos
```

```text
ImageProcessing
│
├── tesina.md                      # Main project report
│
├── estensione_tesina.md           # Report extension
│
├── correzione_tesina.md           # Report correction
│
└── README.md
```
