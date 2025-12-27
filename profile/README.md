# PG Research Project 2025 - SWW
## Symulator Wad Wzroku (Visual Impairments Simulator)

<div align="center">

![VR Simulation](https://img.shields.io/badge/VR-CAVE%20System-blue?style=for-the-badge&logo=oculus)
![Research](https://img.shields.io/badge/Research-2025-green?style=for-the-badge&logo=academia)
![University](https://img.shields.io/badge/Politechnika-Gdańska-red?style=for-the-badge&logo=university)

*Advancing accessibility research through immersive virtual reality simulation*

</div>

---

## 🎯 Project Overview

The **Symulator Wad Wzroku (SWW)** is a research initiative developed at **Politechnika Gdańska** (Gdansk University of Technology). The project aims to create a simulation environment for visualizing various eye defects and impairments.

The system is composed of two primary applications:
1.  **SWW-UnitySimulator**: The main simulation engine built in Unity, responsible for rendering the virtual environment and applying visual impairment effects.
2.  **SWW-ExternalController**: A WPF (Windows Presentation Foundation) application that acts as a remote control for the simulator, allowing researchers to toggle and adjust impairments in real-time.

## 🏛️ Institution

**Politechnika Gdańska (Gdansk University of Technology)**
- Faculty of Electronics, Telecommunications and Informatics
- Department of Intelligent Interactive Systems
- [Immersive 3D Visualization Lab (LZWP)](https://eti.pg.edu.pl/en/lzwp-en)

## 🔬 Research Objectives

### Primary Goals
The primary goal of the research is to evaluate the usability and realism of the VR application in simulating various visual impairments. The study aims to:
1.  **Assess Realism**: Verify if the simulation faithfully reproduces the perception of individuals with specific visual defects.
2.  **Evaluate User Experience**: Determine the degree of perceptual limitation felt by users without visual impairments.
3.  **Analyze Task Difficulty**: Measure how different simulated conditions affect the subjective difficulty of performing daily tasks.
4.  **Validate Design Utility**: Confirm if the simulator can serve as an effective tool for architectural design oriented towards accessibility.

### Research Questions
- Does the VR application faithfully reproduce the perception of the world by people with selected visual impairments?
- To what extent do users (without visual impairments) feel perceptual limitations when using the simulator?
- Does the simulation of different eye conditions affect the subjective assessment of the difficulty of performing daily tasks in a virtual environment?
- Can the immersive spatial visualization simulator be effectively used as a tool to support the design of architectural space adapted to people with visual impairments?

### Implemented Visual Impairments

Based on the source code analysis and project documentation, the simulator currently implements the following visual impairments:

#### Camera-Based Effects (Post-Processing)
These effects are applied to the camera's view to simulate global vision changes.
*   **Refractive & Focus Errors**: `Farsighted`, `Shortsighted`, `DepthBlur`, `BlurVision`
*   **Color Vision Deficiencies**: `ColorBlind_Deuteranopia`, `ColorBlind_Protanopia`, `ColorBlind_Tritanopia`
*   **Environmental & Light Sensitivity**: `FoggyVision`, `NightVision`, `Desaturation`, `Bloom`, `Halo`
*   **Distortions & Disorientations**: `Distortion`, `LineDistortion`, `DoubleVision`, `Dizziness`, `CameraShake`

#### Sphere-Based Effects (Localized)
These effects are rendered using a sphere around the user to simulate localized field loss or obstructions.
*   **Field Loss**: `RadialVignette` (Tunnel Vision), `DarkSpot`
*   **Obstructions**: `Floaters`
*   **Legacy Effects**: `(Legacy)BlindSpots`, `(Legacy)Floaters`

## 🏗️ System Architecture

The system operates on a **Client-Server** architecture designed for a CAVE (Cave Automatic Virtual Environment) setup.

*   **Server (Controller)**: The WPF application acts as the command center. It hosts a TCP server and broadcasts its presence via UDP.
*   **Client (Simulator)**: The Unity application runs on the CAVE cluster. It auto-discovers the server, connects via TCP, and executes commands.

### Communication Protocol
*   **Discovery**: UDP Broadcast on port **7777**. Message: `SWW_ExternalController:<IP>:<PORT>`.
*   **Command Channel**: TCP connection on port **41002**.
*   **Data Format**: Custom string-based protocol.
    *   *Impairments*: `VisualImpairments:Name,Strength;Name2,Strength2\n`
    *   *Transform*: `SphereRendererInitialTransform:X;Y;Z;...`

## 🧩 Component Analysis

### 1. SWW-UnitySimulator (The Engine)
Built on **Unity 2018.1.9f2**, this component handles the immersive visualization.

#### 👁️ Visual Impairments Engine
The core of the simulation uses a **Dual-Rendering Pipeline** to support different types of vision defects:

*   **ScriptableObject Architecture**: Each impairment is defined as a `VisualImpairmentSO` asset. This allows for modular configuration of shaders, default strengths, and renderer types without recompiling code.
*   **Camera Renderer (`VisualImpairmentsRenderer_Camera`)**:
    *   Handles **Post-Processing Effects** (e.g., Blur, Color Blindness, Distortions).
    *   Uses `OnRenderImage` to intercept the render pipeline.
    *   Dynamically stacks multiple shader passes using `Graphics.Blit`. If multiple effects are active, it manages temporary `RenderTexture` buffers to chain them efficiently.
*   **Sphere Renderer (`VisualImpairmentsRenderer_Sphere`)**:
    *   Handles **Localized Effects** (e.g., Tunnel Vision, Floaters, Scotomas).
    *   Renders a physical mesh sphere surrounding the user.
    *   Manipulates the `MeshRenderer` material array at runtime, layering impairment materials over the base material.

#### 🎮 Task Management System
A structured system for conducting ADL (Activities of Daily Living) experiments:
*   **TaskManagerBehaviour**: A Singleton manager that orchestrates the experiment flow. It synchronizes the state of the "Clipboard" (instructions) across the cluster using RPCs.
*   **Task Logic**:
    *   Base `Task` class defines the contract (`isDone`, `getDescription`).
    *   Complex tasks like `WashHandsTask` utilize **Trigger Zones** and **Flystick Tracking** (checking for specific controller handles within a collider) to validate user actions over time.
*   **Cluster Synchronization**:
    *   **LZWPlib**: A custom library used for synchronizing the CAVE cluster (Master/Slave nodes).
    *   **NetworkView**: Uses legacy RPCs (`RPCMode.All`, `RPCMode.OthersBuffered`) to ensure all walls of the CAVE display the same task state and UI text.

### 2. SWW-ExternalController (The Remote)
A **WPF (.NET)** application built with the **MVVM (Model-View-ViewModel)** pattern for robust state management.

#### 🔌 Network Manager
*   **TCP Server**: Uses `TcpListener` to accept connections from the Unity Simulator.
*   **UDP Broadcaster**: Runs a background thread to announce the controller's IP address to the local network, enabling "Zero-Config" connections.
*   **Thread Safety**: Marshals network events back to the UI thread for safe updates.

#### 🎛️ Visual Impairments Manager
*   **State Management**: Maintains an `ObservableCollection<VisualImpairment>` representing the current simulation state.
*   **Real-time Updates**: Subscribes to `PropertyChanged` events on every impairment model. When a slider is moved, it immediately serializes the new state and transmits it to the simulator.

#### 💾 Presets System
*   **JSON Persistence**: Configurations are saved as JSON files in a `./presets/` directory.
*   **Manager**: `PresetsManager` handles CRUD operations, ensuring researchers can switch between complex impairment profiles (e.g., "Glaucoma Stage 3") instantly.

#### 💻 Developer Console
*   **Debug Tool**: A built-in console (`DevConsoleManager`) allows for direct command injection and debugging.
*   **Features**: Supports command suggestions and history, useful for testing the network protocol directly.

## 📊 Features

### Real-time Control
The **External Controller** allows for real-time manipulation of the simulation. It uses a TCP connection to send commands to the Unity Simulator.
*   **Discovery**: The controller broadcasts its presence on UDP port 7777, allowing the simulator to automatically find and connect to it.
*   **Parameter Adjustment**: Each impairment has adjustable parameters (e.g., `EffectStrength`) that can be modified on the fly.

### Preset System
The system supports saving and loading of **Visual Impairment Presets**. This allows researchers to define specific combinations of impairments (e.g., "Severe Myopia with Color Blindness") and quickly apply them during experiments.


### Task Management System
The Unity Simulator includes a `TaskManager` to handle experimental tasks based on Activities of Daily Living (ADL).
*   **Task Workflow**: Tasks are activated sequentially.
*   **Implemented Tasks**:
    1.  **Meal Preparation**: Sandwich assembly in the Kitchen.
    2.  **Personal Hygiene**: Hand washing in the Bathroom.
    3.  **Consumer Electronics**: Finding a remote and turning on the TV in the Bedroom.
*   **Extensibility**: New tasks can be added by extending `TaskSetup` and `Task` classes.

### Sceneries
The simulation takes place in a virtual single-family home environment, consisting of 4 main rooms arranged in an inverted "T" layout:
*   **Hall**: Central hub connecting all rooms.
*   **Kitchen**: Fully equipped with appliances and interactive objects.
*   **Bathroom**: Standard fixtures for hygiene tasks.
*   **Bedroom**: Cozy environment with a bed and TV.
*   **Intro**: A lobby scene for tutorials and initial setup.
## 🖥️ Deployment Requirements

The system is engineered for a specific high-end visualization environment.

### Hardware Environment
*   **CAVE System**: A multi-wall projection system driven by a cluster of rendering nodes.
*   **Tracking**: 6-DOF tracking system for user head position and interaction devices ("Flystick").
*   **Network**: Low-latency LAN connecting the rendering cluster and the control station.

### Software Dependencies
*   **Runtime**: Unity 2018.1.9f2 (Specific version required for `LZWPlib` compatibility).
*   **OS**: Windows 10/11 (Required for WPF External Controller).
*   **Dependencies**:
    *   `LZWPlib`: Proprietary library for cluster synchronization and projection mapping.
    *   `.NET Framework`: Required for the External Controller.
## 📄 Documentation

Project documentation is maintained internally by the research team.
*   **Systematic Literature Review**: A comprehensive review of existing VR visual impairment simulations has been conducted to inform the design.
*   **Research Article**: A paper detailing the system architecture, implementation challenges, and validation results is currently in preparation.

## 👥 Team

**Supervisor**: dr inż. Jacek Lebiedź

**Research Team**:
- Adam Cherek
- Mikołaj Bisewski
- Karolina Zaborowska
- Barbara Badziąg
- Marcin Chętnik
- Bogumiła Merc
- Tomasz Pietrowski

## 📄 License

**Property of Politechnika Gdańska (Gdansk University of Technology) and the reseach team.**

This is a **closed-source** research project. The code and assets are proprietary.
Access to the source code, binaries, or use of the simulator requires explicit permission and contact with the university representatives.

## 🙏 Acknowledgments

- **Politechnika Gdańska** for providing research facilities
- **LZWP Team** for guidance and great atmosphere 

---

<div align="center">

**Made with ❤️ by the PG Research Team 2025**

*Advancing accessibility through innovation*

![Politechnika Gdańska](https://img.shields.io/badge/Powered%20by-Politechnika%20Gdańska-blue?style=for-the-badge)

</div>
