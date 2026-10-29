# Project Thesis Extension

## Porting and Unification of the Three Pipelines on a Single Intel macOS Machine

*Additional chapters supplementing the main project report.*  
*Instructional guide for installing the models on an Intel macOS machine.*

---

# 10. Why a Single Machine: From Theoretical Comparison to Simultaneous Testing

In the main thesis, the three scenarios—**A** (SAM3DBody-cpp), **B** (AlphaPose + HybrIK), and **C** (MMDetection + HybrIK)—are presented and compared from architectural (Chapter 3) and implementation (Chapter 4) perspectives as three equivalent alternatives for human pose estimation. In practice, however, they originated under conditions that were anything but equivalent: the Python scripts for AlphaPose and MMDetection had been written by separate team members assuming **Linux with mandatory NVIDIA/CUDA GPU acceleration**; the C++ engine of SAM3DBody-cpp was likewise designed around CUDA acceleration, using `--cuda DEVICE` as its default parameter. In short, each scenario could **only be demonstrated separately**, on the specific machine or environment in which it was originally developed by the person responsible for it.

This fragmentation prevented a direct, "apples-to-apples" comparison among the three pipelines and, above all, ruled out a unified live demonstration of the project: switching from one scenario to another would have required switching machines, operating systems, and, effectively, the person operating them.

Hence the decision to attempt porting **all three scenarios onto the same machine**, pursuing a two-fold objective:

1. concretely verify whether it was possible to build a **single launcher/program** capable of starting any of the three detection/pose/mesh-recovery models on demand and connecting it to the **same target Unity project**, without requiring manual reconfiguration whenever the pipeline is changed;
2. be able to **test, compare, and demonstrate** all three pipelines within the same working session, using the same webcam and the same avatar, rather than relying on different machines or separate video recordings made by different people.

The machine selected for this unification is a **16" Intel MacBook Pro** (thus completely lacking CUDA). This choice imposed a cross-cutting constraint across all three scenarios—the central thread of the entire work described in this extension: none of the three could rely any longer on the NVIDIA hardware acceleration for which they were originally developed, and each had to be adapted to run **stably and exclusively on CPU**.

The unification effort was **functionally successful**: each of the three pipelines runs on the same machine, within the same shared Python environment where applicable, and exposes to Unity the identical UDP protocol with JSON payload already detailed in the main thesis (Chapter 5). However, it also brought to light a significant practical limitation, particularly for **Scenario A**: the computational cost of the SAM3DBody-cpp pipeline—a heavy Vision Transformer derived from DINOv2 and designed for high-end GPUs—proved, without any hardware acceleration, to make real-time execution virtually impossible on this machine (see §13.8). This in itself represents a key comparative finding among the three scenarios, complementing the qualitative conclusions drawn in the main thesis (Chapter 9).

---

# 11. Project Organization on the Development Machine

To enable the three pipelines to coexist, the project was organized around a single root directory (hereafter referred to as **`Tesina/`**), which brings together both the shared Python environment and the cloned repositories of the three models, keeping them distinct from the group's actual application repository.

```text
Tesina/
│
├── venv_tesina_310/              # Python 3.10 virtual environment, shared by AlphaPose and MMDetection
│                                  # (and also used for SAM3DBody-cpp auxiliary Python scripts:
│                                  #  model downloads, quantization)
│
├── AlphaPose/                    # Cloned repository — Scenario B (detection + 2D/3D pose)
│   └── detector/yolo/data/
│       └── yolov3-spp.weights     # YOLO weights (~240 MB, downloaded separately)
│
├── HybrIK/                       # Cloned repository — shared by Scenarios B and C
│   ├── pretrained_models/
│   │   └── hybrik_hrnet48_wo3dpw.pth
│   └── model_files/
│       └── smplx/SMPLX_NEUTRAL.npz
│
├── mmdetection/                  # Cloned repository — Scenario C (detector)
│   ├── checkpoints/
│   │   └── rtmdet_tiny.pth
│   └── configs/rtmdet/
│       └── rtmdet_tiny_8xb32-300e_coco.py
│
├── SAM3DBody-cpp/                # Cloned repository — Scenario A (standalone C++ engine)
│   ├── CMakeLists.txt
│   ├── build/                    # CMake build directory, contains the executable
│   │   └── fast_sam_3dbody_run
│   ├── src/                      # Source files directory
│   │   └── main.cpp              # Scenario A script, adapted for macOS/CPU
│   └── onnx/                     # Model weights (backbone, decoder, YOLO, pipeline.gguf, body_model.lbs)
│
└── ImageProcessing/               # Git repository of the group's application project
    └── test_vale/                 # Author's personal working/test directory for this extension
        ├── angoliAlphaPose_HybrIK.py       # Scenario B script, adapted for macOS/CPU
        ├── angoliMMdetection_HybrIK.py     # Scenario C script, adapted for macOS/CPU
        └── main_launcher.py                # Pipeline selection menu (see Chap. 16)
```

Key points regarding this structure:

- The three model repositories (`AlphaPose/`, `HybrIK/`, `mmdetection/`, `SAM3DBody-cpp/`) were **clonally hosted locally**, with their paths dynamically added to `sys.path` at runtime via a dedicated context manager (`in_directory`, see §15.2) for the Python scenarios, while `SAM3DBody-cpp` is built directly using CMake, generating a standalone native binary independent of the venv.
- The `venv_tesina_310/` folder serves as the single Python environment for the entire project: the decision to share it between AlphaPose and MMDetection (instead of isolating each scenario in its own venv) is the starting point of Chapter 12 and the source of the primary dependency conflicts encountered.
- `ImageProcessing/project_files` is the actual application repository of the group (containing the "final" executable scripts rather than third-party libraries), with `test_vale/` acting as a personal workspace dedicated to this author's tests.
- The Unity project (housing `AvatarController.cs`, `IPoseProvider.cs`, the various Providers, `ConnectionManager.cs`, and `UDPReceiver.cs`; see Chapter 16) is managed as a standalone project, developed and versioned independently by the teammates working on the C# side; interaction between both components occurs exclusively at runtime via UDP sockets on port `5065`, eliminating any requirement for a shared file directory.

---

# 12. Building the Shared Environment

The fundamental challenge underlying this entire chapter is that hosting **three heterogeneous pipelines** (two PyTorch-based packages installed in development mode, and one written in C++ and natively compiled) on the same machine implies **sharing system resources** which, if modified for one scenario, can silently break the other two.

## 12.1 Shared Python 3.10 venv

```bash
python3.10 -m venv venv_tesina_310
source venv_tesina_310/bin/activate
```

From this point forward, this venv accommodates **both** AlphaPose and MMDetection (Scenario C), in addition to the helper Python scripts used to prepare models for Scenario A (see §12.3).

## 12.2 The Risk of Reuse: A Dependency for One Scenario Can Break Another

Rather than setting up a second isolated venv for MMDetection, the choice was made to **reuse `venv_tesina_310`**, which had already been configured for AlphaPose. The clear advantage was having only one environment to maintain and a single Python interpreter to call from the launcher or Unity; the risk, evident from the outset, was that **any dependency installed for one scenario could break the other** if a common library (such as PyTorch, NumPy, or OpenCV) was altered. This risk had been explicitly flagged as a constraint during development and **actually materialized** several times:

- **NumPy/OpenCV Conflict Between MMDetection and AlphaPose.** An attempt to address an MMDetection issue by reinstalling OpenCV (`pip install opencv-python`) silently upgraded **NumPy from 1.26.4 to 2.2.6** (an automatic requirement of OpenCV 5.x) and installed **OpenCV 5.0.0.93**; both versions proved incompatible with the Cython/PyTorch extensions compiled for AlphaPose, where NumPy ≥2 had already been identified as a critical breaking change (see §14.3). The fix was an explicit and targeted downgrade:

```bash
pip uninstall opencv-python numpy
pip install "numpy<2.0"
pip install "opencv-python==4.9.0.80"
```

- **MMCV Compilation Failure.** Attempting to install the full MMCV release (rather than `mmcv-lite`) resulted in `Failed building wheel for mmcv` / `clang failed with exit code 1`: no prebuilt binary wheel (`.whl`) existed for macOS compatible with PyTorch 2.2.2, prompting `pip` to **compile MMCV from source** using `clang`. This build failed because the underlying C++ source (`iou3d.cpp`, `nms.cpp`, etc.) contained constructs unsupported by the compiler/headers available on macOS.

- **Architectural Decision: `mmcv-lite` + `torchvision.ops`.** To entirely avoid building native C++ extensions on a Mac without CUDA, it was decided to install **`mmcv-lite`** (which omits native `mmcv._ext` binaries while keeping the entire algorithmic logic in pure Python) and redirect critical geometric computations to **`torchvision.ops`**, already packaged with PyTorch inside the venv: `torchvision.ops.roi_align` replaced MMCV's custom RoI Align, and `torchvision.ops.nms` replaced MMCV's C++ NMS. The initial deployment of `mmcv-lite` still triggered a predictable error—`ModuleNotFoundError: No module named 'mmcv._ext'`—at points where MMDetection components still attempted to load the absent native binaries; redirection to `torchvision.ops` was therefore applied systematically across all affected operations.

- **Bypassing the MMCV Version Check in MMDetection.** `mmdet/__init__.py` enforces a rigid version check on the installed MMCV package (`assert mmcv_version >= min and mmcv_version < max`); the installed `mmcv-lite` version (2.2.0) was slightly higher than the expected upper limit, causing this `assert` to abort execution before reaching any actual pipeline logic. It was therefore necessary to directly edit the installed library source file of MMDetection, relaxing the upper bound (`<` → `<=`)—another instance, along with AlphaPose's `setup.py` (see §14.2), of directly modifying library internals rather than simply application code.

- **Missing Dependency: `pytorch3d`.** While absent in Scenario B, HybrIK in the variant used for Scenario C (which additionally loads the **SMPL-X** model) requires `pytorch3d` for rotation representations and conversions. `pytorch3d` does not typically supply precompiled wheels for macOS; building from source initially failed with the identical pattern observed in AlphaPose (`ModuleNotFoundError: No module named 'torch'` during pip's isolated build environment), which was resolved in the exact same manner:

```bash
pip install --no-build-isolation "git+https://github.com/facebookresearch/pytorch3d.git"
```

## 12.3 SAM3DBody-cpp and Its Separate Toolchain

Unlike AlphaPose and MMDetection, **SAM3DBody-cpp is not a Python package**: it is a standalone C++ project compiled natively using **CMake** and linked to system libraries managed via **Homebrew** (`cmake`, `opencv`), residing completely outside the virtual environment. This fundamentally shielded the project from a fourth potential Python dependency conflict—yet still required careful handling at two specific intersection points with the shared environment:

1. **Two Distinct, Non-Conflicting OpenCV Distributions.** SAM3DBody-cpp's C++ build links against the **system OpenCV installed via Homebrew** (discovered by CMake through `OpenCV_DIR`), whereas the Python scripts for Scenarios B and C utilize the **`opencv-python` package within the venv**. These are two physically separated installations (one as a system dynamic library, the other in the venv's `site-packages`), and they never conflicted—demonstrating how physical isolation of environments cleanly avoided the NumPy/OpenCV conflicts previously encountered between AlphaPose and MMDetection (see §12.2).
2. **Auxiliary Python Scripts Run Within `venv_tesina_310`.** Retrieving and preparing the ONNX models for SAM3DBody-cpp (downloading from HuggingFace, quantization attempts) were performed using Python scripts executed **inside** the shared venv, installing additional helper packages (`huggingface_hub`, `onnxruntime`, `onnx`). This pragmatic choice (keeping a single active Python interpreter on the system) introduced no friction with AlphaPose or MMDetection, as those packages were not shared by either of the two Python scenarios.

In summary, the coexistence of the three models on the same machine rested on **two distinct, intentionally decoupled environments**—the shared Python venv (AlphaPose + MMDetection + SAM3D helper scripts) and the system-level Homebrew/CMake toolchain (SAM3DBody-cpp)—with only one genuine internal friction point: sharing the virtual environment between AlphaPose and MMDetection, as discussed in §12.2.

---

# 13. Scenario A — SAM3DBody-cpp: Installation and Adaptation for Intel Mac CPU

Unlike Scenarios B and C, Scenario A did not involve modifying an existing Python script, but rather **compiling a C++ project from scratch** (the ONNX/GGML optimized engine, `Fast-SAM-3D-Body`) and configuring it to run on a machine without CUDA. The work detailed in this chapter was the last of the three to be completed, building upon the insights gained from Scenarios B and C.

## 13.1 Linker Failure: `librt` Does Not Exist on macOS

Once OpenCV was properly linked, compilation progressed up to the **linking** stage of the shared library `libfast_sam_3dbody.dylib`, where it failed. The root cause lay in the project's top-level `CMakeLists.txt`, which automatically assigned `RT_LIB` to `rt` for any non-Windows platform:

```cmake
if(WIN32)
    set(MATH_LIB "")
    set(RT_LIB    "")
else()
    set(MATH_LIB m)
    set(RT_LIB    rt)
endif()
```

While this correctly links `librt` (POSIX real-time extensions) on Linux, on **macOS** a separate `librt` library does not exist because those functions are integrated directly into the standard system C library—attempting to link it explicitly causes a fatal linker error. The fix, made directly to the project source file (analogous to the modifications made to AlphaPose's `setup.py` and MMDetection's `mmdet/__init__.py`, see Chapters 12 and 15), was to treat macOS like Windows for this specific setting:

```cmake
if(WIN32 OR APPLE)
    set(MATH_LIB "")   # libm is built into system libraries on macOS/Windows
    set(RT_LIB    "")  # librt is not needed on macOS/Windows
else()
    set(MATH_LIB m)
    set(RT_LIB    rt)
endif()
```

## 13.2 ONNX Runtime: Automatic Download for the Wrong Platform

The project's build system attempts to automatically download ONNX Runtime during CMake configuration; however, the build logs showed `Downloading ONNX Runtime 1.20.1 (linux-x64)`—a **Linux** binary entirely unusable on macOS. The solution was to manually download the correct build for **Intel macOS (x86_64)** from the official ONNX Runtime GitHub releases page and pass its path explicitly to CMake:

```bash
curl -L -o onnxruntime-osx.tgz \
  https://github.com/microsoft/onnxruntime/releases/download/v1.20.1/onnxruntime-osx-x86_64-1.20.1.tgz
tar xf onnxruntime-osx.tgz
cmake .. -DONNX_RUNTIME_DIR=$(pwd)/onnxruntime-osx-x86_64-1.20.1
```

With these two fixes (patched `CMakeLists.txt` and native macOS ONNX Runtime), the project successfully built the `fast_sam_3dbody_run` executable.

## 13.3 Model Weights: Git LFS Placeholder Files Instead of Real Binaries

During configuration, `CMakeLists.txt` checks for the presence of ONNX model files in the repository directory and points to where they can be downloaded (a repository on HuggingFace: `AmmarkoV/SAM3DBody-cpp-onnx-models`).  
Initial download attempts using standard web requests only retrieved small Git LFS text pointer files rather than the actual model binaries. The issue was resolved by switching to the **official HuggingFace Hub CLI**, which properly resolves LFS pointers and fetches the actual binary weights:

```bash
pip install -U huggingface_hub
huggingface-cli download AmmarkoV/SAM3DBody-cpp-onnx-models --local-dir ../onnx
```

## 13.4 External `.data` Files and Mismatched YOLO Filenames

Two practical details, not readily apparent from standard error messages, required additional troubleshooting:

- Each large `.onnx` file (backbone, decoder) contains **only the network graph structure**; the actual weight tensors reside in an accompanying `.data` file that must be placed alongside the corresponding `.onnx` file. Omitting it triggers out-of-bounds file read errors during runtime (see §13.5).
- The YOLO detector file published on HuggingFace is named `libreyolo9.onnx`, rather than `yolo.onnx` as assumed by the default startup command. This was resolved by either renaming the file or explicitly supplying `--yolo ../onnx/libreyolo9.onnx` and `--detector libreyolo` to select the correct detector type in the code.

Intermediate artifacts from failed attempts (such as generic INT8 variants and cleaned models, each several gigabytes in size) were subsequently purged from the `onnx/` directory to prevent the executable from picking up stale or corrupted files.

## 13.5 Webcam on macOS: Permissions, Device Index, and the False "Freeze"

Once model loading was working, running the program with webcam input initially appeared to have no effect (no display window, no console output, and a seemingly frozen terminal). Systematic troubleshooting isolated **three distinct, concurrent factors**, mirroring the diagnostic approach used for Scenarios B and C (a minimal `test_cam.py`/`check_cam.py` script containing only `cv2.VideoCapture` to determine whether the issue stemmed from the camera or the AI model):

1. **macOS Privacy & Security Camera Permissions**: Terminal required explicit camera access, requiring a full application restart after granting permission for the change to take effect.
2. **Device Index**: On this specific MacBook, index `1` (rather than `0`) mapped to the built-in webcam—a detail uncovered only by comparing logs across different sessions ("previously it worked with `--from 0`" vs. "now it freezes with `--from 0`"—the real discrepancy lay elsewhere, as explained next).
3. **The "Freeze" Was Not a Lockup**: In `main.cpp`'s main loop, the `cv::imshow()` window is opened **only after** the frame has been processed by the AI pipeline (`pipeline.process_bgr(...)`). On a CPU without hardware acceleration, running a single frame through the heavy Vision Transformer required **15–30 seconds or more**. The application was not hanging: it was simply computing at 400–500% CPU utilization (as verified via Activity Monitor) before the first frame could ever be displayed.

## 13.6 CPU Performance Optimization

Recognizing that the true bottleneck was the computational cost per frame, two lines of intervention were pursued.

**Targeted Dynamic Quantization.** An initial attempt at generic dynamic quantization (`quantize_dynamic` applied across all layers) resulted in a new error: `Could not find an implementation for ConvInteger`. This occurred because quantization had been applied to early convolutional layers (`patch_embed`), generating integer operations unsupported by ONNX Runtime's CPU execution provider. The solution was restricting quantization strictly to `MatMul` layers (matrix multiplications, which dominate Vision Transformers), preserving convolutions in full FP32 precision:

```python
from onnxruntime.quantization import quantize_dynamic, QuantType

quantize_dynamic(
    model_input="onnx/backbone_fp32.onnx",
    model_output="onnx/backbone_int8_dynamic.onnx",
    weight_type=QuantType.QInt8,
    op_types_to_quantize=['MatMul'],   # essential to avoid unsupported ConvInteger nodes
)
```

Even after quantization, inference remained sluggish: the underlying model (a Vision Transformer derived from DINOv2) is fundamentally designed for high-end NVIDIA GPUs—the authors benchmarked it at approximately 65 ms/frame on an RTX 5090—and no software-level CPU optimization can approach that regime.

**Direct Source Modifications to `main.cpp`.** Two targeted interventions were added directly to the main loop immediately following frame capture:

```cpp
// Forced software resizing: macOS webcam drivers often ignore
// cv::VideoCapture::set() for resolution, so resizing is performed
// explicitly right after frame capture
if (c.cap_w > 0 && c.cap_h > 0)
{
    cv::resize(frame, frame, cv::Size(c.cap_w, c.cap_h));
}
```

This enabled downscaled resolutions (such as `--size 320 240`) to genuinely alleviate the transformer's workload without relying on the webcam hardware to natively support that resolution. The `cv::imshow()` display window with overlaid 2D skeleton was **deliberately kept active** (contrary to an earlier suggestion of disabling it to conserve CPU cycles): it was indispensable for live demonstrations, and software resizing achieved the needed computational relief without eliminating real-time visual feedback. In addition, execution was configured with `OMP_NUM_THREADS=4` to constrain and parallelize thread allocation in ONNX Runtime.

## 13.7 Summary of Differences: Assumed Linux/CUDA → Intel macOS CPU (Scenario A)

| Aspect | Original Version (Assumed CUDA) | Intel macOS Version |
|---|---|---|
| Build Toolchain | `cmake`/`make`, `--cuda DEVICE` default | Same toolchain; `cuda_device = -1` by default in `Config`; `RT_LIB`/`MATH_LIB` cleared for `APPLE` |
| OpenCV | Unspecified | Homebrew `opencv@4` (avoiding default 5.x for compatibility with `calib3d` required by `multiview`) |
| ONNX Runtime | Auto-downloaded by build system (`linux-x64`) | Manually downloaded (`osx-x86_64`) and supplied via `-DONNX_RUNTIME_DIR` |
| Model Weights | Assumed pre-existing/downloaded | Git LFS pointers initially fetched by `urllib`, resolved via `huggingface-cli` |
| Backbone Precision | Unspecified (likely FP16/INT8 on GPU) | FP32 for CPU compatibility, followed by `MatMul`-only dynamic quantization |
| Webcam | Assumed direct capture | macOS permissions required, non-trivial camera index, apparent "freeze" caused purely by inference latency |
| Performance | GPU (up to top-tier RTX), real-time claimed by authors | CPU, several seconds per frame even with quantization and software downsampling |

---

# 14. Scenario B — AlphaPose + HybrIK: Adaptation for Intel Mac CPU

The baseline script (`angoliAlphaPose_HybrIK.py`, Linux release) implements a real-time pipeline that captures webcam input, detects people with a YOLO detector (AlphaPose), estimates 3D SMPL poses via HybrIK (24 joints, rotations + 3D coordinates), directly converts rotation matrices to **unit quaternions `(x, y, z, w)`**, and sends data over UDP, optionally displaying an annotated video window. It was originally written strictly for a **Linux environment with NVIDIA/CUDA GPU acceleration**: it queried device names via `torch.cuda.get_device_name(...)` without fallback logic, utilized `torch.autocast(device_type="cuda", ...)` for FP16 inference, and relied on AlphaPose components (threaded webcam loaders, NMS) compiled natively for that architecture.

The porting effort unfolded in two distinct phases: first, **environment setup and installation** (§14.1–14.4), followed by **runtime debugging** once the script was executable but exhibited erratic behavior (§14.5–14.8). In both stages, progress was strictly incremental: each run exposed a new issue that was isolated with minimal test scripts, resolved, and followed by the next hurdle—a necessary layered approach given that errors spanned operating system quirks, library signature changes, missing native extensions, and data logic bugs.

## 14.1 Removing the Hard CUDA Dependency

The original script halted immediately if CUDA was absent (`raise RuntimeError("CUDA non disponibile...")`). Because the Intel Mac possesses neither CUDA nor MPS support for these pipelines (the latter being primarily targeted at Apple Silicon, though see §15.6 for nuances), an alternative fallback branch `device = torch.device("cpu")` was introduced, along with automatic deactivation of FP16/AMP (GPU-specific optimizations inapplicable to CPU).

## 14.2 Native AlphaPose Build: Dependency Conflicts and `setup.py` Patches

Installing AlphaPose in editable mode (`pip install -e .` in the cloned repo) required several consecutive fixes:

- **Missing `requirements.txt`**: Packages were installed individually (`opencv-python`, `numpy`, `pyyaml`, `scipy`, `matplotlib`, `cython`, `easydict`, `json_tricks`, `tensorboard`, `shapely`, `tqdm`, etc.).
- **`ModuleNotFoundError: No module named 'torch'` during build**: `setup.py` imports `torch` to configure C++/CUDA extensions, but `pip`'s build isolation hides already-installed packages in the venv. This was circumvented using `pip install --no-build-isolation -e .`.
- **NumPy 2.x Incompatibility**: Fixed by pinning `pip install "numpy<2.0"` and aligning OpenCV accordingly (`pip install "opencv-python<5.0"`, since OpenCV 5.x mandates NumPy ≥2).
- **`OSError: CUDA_HOME environment variable is not set`**: The primary roadblock. `setup.py` unconditionally attempted to compile CUDA extensions (including the Cython module `soft_nms_cpu.pyx`, which despite its name was bound to CUDA checks in the original build script), failing on a system without an NVIDIA GPU. It was necessary to **rewrite `setup.py`**, removing all CUDA compilation logic and retaining solely the CPU-compatible Cython extension (`soft_nms_cpu`) inside a simplified, exception-safe `get_ext_modules()` function.
- **Residual Syntax Bug in the Patch**: The initial patch passed `extra_compile_args` as a `dict` rather than a `list` on macOS (`TypeError: can only concatenate list (not "dict") to list`), which was fixed by normalizing compiler arguments into a list of strings.

## 14.3 Installing HybrIK and the `chumpy` Dependency

HybrIK was installed in development mode (`pip install -e .`), presenting fewer setup hurdles than AlphaPose. The legacy package **`chumpy`** was required to properly deserialize `.pkl` files (the legacy data format for SMPL statistical models), installed alongside `pycocotools`, `tqdm`, and `easydict`.

## 14.4 Initial NMS Fallback (`nms_wrapper.py`)

With `soft_nms_cpu` as the only compiled extension (omitting `nms_cpu`), the unmodified code threw `ImportError: cannot import name 'nms_cpu' from partially initialized module`. An initial version of `detector/nms/nms_wrapper.py` was introduced with cascaded fallback imports (`nms_cuda` → `nms_cpu` → `None`) and an `nms()` routing function. While this allowed the module to be imported without crashing, it did not provide a functional fallback when **neither** native extension was compiled—a limitation resolved definitively in §14.6 using `torchvision.ops.nms`.

## 14.5 Locating Pretrained Weights and Resolving Checkpoint/Config Mismatches

Deploying the pretrained weights involved multiple hurdles: the expected HybrIK checkpoint (`pretrained_w_cam.pth`) was not immediately located (an alternative file was initially used and later replaced); the SMPL model file (`basicModel_neutral_lbs_10_207_0_v1.0.0.pkl`) was recovered from prior archives and placed in the proper path; and the YOLO detector weights (`yolov3-spp.weights`, ~240 MB) were manually downloaded into `detector/yolo/data/`.

Once file paths were verified, loading model weights raised `RuntimeError: Error(s) in loading state_dict` for two distinct reasons:
1. An early test paired an HRNet checkpoint with a configuration structured for a ResNet-34 backbone, resulting in incompatible layer names and dimensions—resolved by restoring the matching checkpoint.
2. Even with the proper checkpoint, a *size mismatch* persisted on the `smpl.shapedirs` tensor (300 shape components in the code's model versus 10 in the checkpoint). This was resolved by dynamically slicing the tensor at runtime prior to loading the state dictionary:

```python
if hasattr(pose_model, 'smpl') and hasattr(pose_model.smpl, 'shapedirs'):
    if pose_model.smpl.shapedirs.shape[-1] == 300:
        pose_model.smpl.shapedirs = pose_model.smpl.shapedirs[:, :, :10]
```

## 14.6 Webcam Capture and Decoupling Acquisition from Inference

**Initial Symptom:** Implementations relying directly on `WebCamDetectionLoader` often failed to display the camera feed window on macOS because AlphaPose's loader performed video capture on a secondary thread. On macOS, AVFoundation can block `VideoCapture.read()` when detached from the main thread's event loop.

**Resolution:** Capture was cleanly decoupled from inference. The final script discards `WebCamDetectionLoader`: `cv2.VideoCapture` is managed strictly on the **main thread**, dedicated to acquiring frames and servicing `cv2.imshow()`. The inference thread receives a snapshot of the latest available frame and executes detection, preprocessing, and HybrIK inference.

This decoupling provides significant performance advantages: no backlog queue of pending frames is accumulated. The inference thread continuously operates on the **most recent frame available**, silently dropping frames that arrived while the CPU was busy with inference. Consequently, slower processing does not lead to unbounded latency growth.

Thread synchronization is guarded by a `Lock`, while an additional lock protects the latest frame rendered for visualization. If inference lags behind webcam FPS, `cv2.imshow()` continues updating with the freshest camera data rather than freezing until HybrIK finishes.

Capture settings can be tuned via `--webcam`, `--cam-width`, and `--cam-height`. On the Intel Mac, default dimensions are set to **640×480**, substantially easing the detector's processing overhead compared to HD resolutions.

## 14.7 Additional Runtime Bug Fixes

- **Signature Incompatibility in `SimpleTransform3DSMPL`**: `TypeError: got an unexpected keyword argument 'gpu_device'`, caused by version variations that rejected previously documented parameters. Fixed by dynamically inspecting constructor signatures via `inspect.signature` and filtering keyword arguments accordingly.
- **String Type in `args.gpus`**: `TypeError: '<' not supported between instances of 'str' and 'int'` inside YOLO, which expects a list of integers. Resolved by explicitly normalizing `args.gpus` (`[gpu_ids]` on CUDA, `[-1]` on CPU).
- **Uncompiled NMS Extension (Definitive Fallback)**: `ImportError: Neither nms_cuda nor nms_cpu is available`, solved by introducing a pure PyTorch fallback in `nms_wrapper.py` using `torchvision.ops.nms` (requiring zero native compilation) while matching the return signature expected by callers.
- **Array Indexing Error in Skeleton Visualization**: `IndexError: invalid index to scalar variable`, caused by HybrIK's `pred_xyz_jts_29` returning a flattened 1D array (29 joints × 3 coordinates = 87 values) instead of shape `(29, 3)`. Fixed via `.reshape(-1, 3)`.
- **Inverted Color Channels (RGB vs. BGR)**: `SyncWebcamLoader` converts frames to RGB for AlphaPose, but `cv2.imshow` expects BGR, causing blue-tinted output. Resolved by calling `cv2.cvtColor(vis_img, cv2.COLOR_RGB2BGR)` prior to display.
- **Incorrect Skeleton Visualization Scaling**: `pred_xyz_jts_29` outputs metric, root-relative 3D coordinates without a fixed relationship to image pixels. Resolved by dynamically computing a scaling factor per frame (mapping the 90th percentile of joint-to-pelvis distances to ~45% of the detected bounding box height).

## 14.8 Rotation Output Correction and CPU Performance Tuning

Testing revealed an inconsistency in rotation representations. An intermediate iteration converted HybrIK's \(3\times3\) matrices to Euler angles `(x, y, z)` before inserting them into UDP packets. This format conflicted with the Unity provider used by Scenari B and C, which expects **quaternions `(x, y, z, w)`**. Treating three Euler angles as quaternion components while omitting `w` produces distorted, nonsensical orientations.

The solution retained HybrIK's rotation matrices until the final stage, converting them via `scipy.spatial.transform.Rotation.from_matrix(...).as_quat()`. Each joint orientation is thus transmitted as a 4-element unit quaternion `(x, y, z, w)` without loss of information. While the JSON key name `unity_rotations_deg` was preserved for backward compatibility with the established protocol, its payload in Scenarios B and C consists strictly of quaternion values.

On the **Intel Mac**, the entire pipeline executes on CPU when CUDA is unavailable and FP16/AMP is disabled. Because CPU compute is limited, several targeted optimizations were introduced:

1. **Decoupled Inference Threading:** The main thread handles `cv2.read()` and `cv2.imshow()`, while inference processes only the latest available frame without queue buildup.
2. **Subsampled Detection:** YOLO is executed every `--det-interval` frames (defaulting to **8**). During intermediate frames, the previous bounding box is reused.
3. **Bounding Box Temporal Smoothing:** Between detection steps, exponential smoothing is applied with factor `--bbox-smoothing` (default **0.25**) to dampen cropping jitter without introducing latency.
4. **Single-Person Focus:** Only the detection with the highest confidence score is tracked, matching the single-avatar Unity setup and avoiding redundant compute.
5. **Explicit CPU Parallelism:** `torch.set_num_threads()` is tuned using `--num-threads`; setting it to `0` leverages all available CPU cores.
6. **Controlled Resolution:** The webcam is initialized at **640×480** to reduce detection overhead.
7. **Optional Profiling:** Passing `--profile` logs execution times across detection, transform, HybrIK, UDP broadcast, display, and total latency, making it straightforward to pinpoint bottlenecks.

The combined outcome does not fundamentally accelerate the neural network's intrinsic compute speed, but it ensures that inference delays do not translate into an ever-growing queue of stale frames. UDP packets are emitted at the highest sustainable rate while capture and rendering windows remain smooth and responsive.

## 14.9 Summary of Differences: Linux/CUDA → Intel macOS CPU (Scenario B)

| Aspect | Original Linux/CUDA Version | macOS CPU Version |
|---|---|---|
| Python Environment | Unmanaged system environment | Python 3.10 virtual environment (avoiding Conda and Python 3.13) |
| Native Extension Builds | `setup.py` compiles CUDA extensions (including NMS) | Rewritten `setup.py`: CUDA disabled, only Cython `soft_nms_cpu` compiled |
| Webcam Capture | Background thread (`WebCamDetectionLoader`) | `cv2.VideoCapture` on main thread; inference on background thread with latest-frame exchange |
| Device Target | Mandatory CUDA, no fallback | Automatic CUDA/CPU detection, `args.gpus` normalized across backends |
| Precision | FP16/AMP on GPU | FP16/AMP disabled on CPU |
| Runtime NMS | Native compiled CUDA/CPU extension | Pure PyTorch `torchvision.ops.nms` fallback |
| Joint Rotations | Euler angle intermediate representation | SMPL matrices → unit quaternions `(x,y,z,w)` directly via SciPy |
| SMPL Model | Aligned by default | Manual tensor realignment (HRNet/ResNet backbone, `shapedirs` truncated 300→10) |
| Detection Frequency | Every frame | Every 8 frames by default + bbox smoothing + single person filter |
| Webcam Resolution | Unconstrained | 640×480 default, fully configurable |
| Performance | Hardware GPU acceleration | Decoupled inference thread, latest-frame-only processing, CPU multithreading, and stage profiling |

---

# 15. Scenario C — MMDetection + HybrIK: Adaptation for Intel Mac CPU

As with Scenario B, the starting script (`angoliMMdetection_HybrIK.py`) had been created for **Linux with mandatory NVIDIA/CUDA acceleration** (hardcoded `/home/alessio/...` paths, `device = f'cuda:{opt.gpu}'`, runtime errors when CUDA was missing, Faster R-CNN detector, and `hybrik_hrnet.pth` weights). Adapting it to Intel macOS required not only establishing the shared Python environment with AlphaPose (Chapter 12), but also resolving a distinct chain of runtime failures.

## 15.1 Repository Architecture and Core Frameworks

**MMDetection (v3.3.0)** — The primary PyTorch-based vision framework, cloned directly from source. It provides the infrastructure to parse configuration files, instantiate models (backbones, bounding box heads), and perform inference on webcam streams.

**MMEngine (v1.x)** — OpenMMLab's foundational library overseeing model training and inference execution (Runners), checkpoint loading, and registry tree management.

## 15.2 Adjusting Models and Paths

Initial startup attempts revealed that the script still referenced legacy Linux paths and file configurations:

- `DET_CONFIG`/`DET_CKPT` pointed to **Faster R-CNN** (`faster_rcnn_r50_fpn_1x_coco.py`) instead of **RTMDet-Tiny**, which had been selected for lightweight CPU inference. These were updated to target `configs/rtmdet/` and `checkpoints/rtmdet_tiny.pth`.
- `HYBRIK_CKPT` pointed to `hybrik_hrnet.pth`, which did not exist locally; the downloaded checkpoint was actually **`hybrik_hrnet48_wo3dpw.pth`** (HRNet-W48). The script path was updated directly to reflect the downloaded file and preserve traceability.

## 15.3 Webcam Capture Stabilization: A Different Root Cause than AlphaPose

Once the pipeline was able to initialize, it exhibited the same symptom seen in Scenario B ("the webcam window opens and closes immediately"), but stemmed from an **entirely different root cause**—highlighting how identical symptoms on macOS can trace back to unrelated mechanisms. A diagnostic test (`check_cam.py`) revealed:

```text
opened: True
0 ret = False
```

Unlike Scenario B—where failure was caused by secondary-thread capture—this script already executed capture on the main thread. The issue was that **the initial `cap.read()` immediately following `VideoCapture` initialization consistently returns `ret = False`**: AVFoundation needs a brief window to negotiate camera hardware buffers even after `isOpened()` reports `True`. The solution was adding an explicit **camera warm-up**: a short delay accompanied by discarding the initial frames:

```python
cap = cv2.VideoCapture(opt.webcam_id)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)
assert cap.isOpened(), 'Unable to open webcam'

time.sleep(1.0)               # allow AVFoundation to initialize
for _ in range(5):            # discard initial frames, which are often invalid
    cap.read()
```

This workaround is **complementary to, rather than an alternative for**, the changes made to AlphaPose: had capture remained on a secondary thread, a warm-up delay alone would not have resolved the lockup (see §14.6); conversely, on the main thread, the lockup was purely due to hardware buffer initialization delay.

## 15.4 Device Selection: The CUDA → MPS → CPU Fallback and MPS Instability

A notable difference from Scenario B was the inclusion of three-tier automatic device selection:

```python
if torch.cuda.is_available():
    device = torch.device('cuda:0')
elif torch.backends.mps.is_available():
    device = torch.device('mps')
else:
    device = torch.device('cpu')
```

**An Unexpected Detail:** Despite running on an **Intel Mac** (rather than Apple Silicon), `torch.backends.mps.is_available()` evaluated to `True`. This is because PyTorch's **MPS backend utilizes the Metal API**, which is present on Intel Macs equipped with compatible discrete GPUs (in this instance, an AMD Radeon Pro common in 16" MacBook Pros). Therefore, MPS is not strictly synonymous with Apple Silicon (as initially assumed in §14.1). Consequently, the script selected `device = 'mps'`, which immediately surfaced multiple stability issues:

1. **`NotImplementedError: torchvision::nms not implemented for MPS`** — Temporarily bypassed by setting `PYTORCH_ENABLE_MPS_FALLBACK=1`, which silently delegates unsupported operations to CPU within the MPS context.
2. **`AssertionError` in MMEngine's `InstanceData.__getitem__`** — Following the CPU NMS fallback, filtered indices (`keep_idxs`) returned with mismatched formatting when transitioning back to MPS. This was resolved by forcing the **entire detector** (not just NMS) onto CPU, while keeping HybrIK on MPS.
3. **`TypeError: Cannot convert a MPS Tensor to float64`** — MPS does not support double-precision floating-point numbers; NumPy arrays passed into HybrIK (`bbox`, `img_center`) defaulted to `float64`. This required explicitly casting tensors to `dtype=np.float32`.
4. **Unrecoverable Metal Driver Crash** (`MPSNDArrayDescriptor ... failed assertion`, terminating with `zsh: abort`) — A matrix slicing and determinant operation within HybrIK's SMPL layers (`torch.det(...)` in `lbs.py`) executed on MPS triggered an unhandled driver-level exception that could not be caught or recovered in Python.

Point 4 demonstrated unequivocally that **MPS is not a viable or stable platform for this pipeline**: HybrIK was engineered and tested exclusively for CUDA, containing operations whose MPS translations remain incomplete or buggy. The final solution was **abandoning MPS altogether** and pinning the entire pipeline (detector and HybrIK) to CPU:

```python
# --- FORCING CPU TO AVOID MPS BUGS IN HYBRIK ---
device = torch.device('cpu')
```

While the automatic CUDA → MPS → CPU hierarchy remains in the code as defensive fallback logic for other platforms, on the Intel Mac, CPU execution is the only stable path.

## 15.5 Local Execution Pipeline Diagram

```text
[ Webcam frame ]
       │
       ▼
┌───────────────────────────┐
│     MMDetection Core      │ ──► RTMDet-Tiny on CPU (person bbox, every 10 frames)
└───────────────────────────┘
       │
       ▼
┌───────────────────────────┐
│      Torchvision CPU      │ ──► roi_align & NMS (pure PyTorch fallback)
└───────────────────────────┘
       │
       ▼
┌───────────────────────────┐
│   HybrIK (HRNet-W48 cam)  │ ──► CPU (MPS abandoned due to instability)
└───────────────────────────┘
       │
       ▼
[ JSON packet via UDP → Unity ConnectionManager ]
```

## 15.6 Summary of Differences: Linux/CUDA → Intel macOS CPU (Scenario C)

| Aspect | Original Linux/CUDA Version | macOS Version |
|---|---|---|
| Python Environment | Unmanaged system environment | `venv_tesina_310` **shared** with AlphaPose, with mitigated risk of NumPy/OpenCV conflicts (see Chap. 12) |
| Detector | Faster R-CNN (`faster_rcnn_r50_fpn_1x_coco`) | RTMDet-Tiny (lightweight, optimized for CPU) |
| MMCV | Full version with CUDA extensions | `mmcv-lite` + `torchvision.ops` fallbacks (roi_align, nms) |
| MMCV Version Check in MMDetection | Passed natively | `mmdet/__init__.py` patched to accommodate installed version |
| `pytorch3d` | Assumed available/prebuilt | Compiled from source with `--no-build-isolation` |
| Compute Device | Mandatory `cuda:{gpu}`, `RuntimeError` if absent | CUDA→MPS→CPU hierarchy; MPS abandoned due to Metal driver crashes, pinned to CPU |
| Webcam Capture | Direct capture, no known issues | `ret=False` on initial frame: solved via warm-up delay and frame purging (**not** thread restructuring) |
| Tensor Precision | Standard behavior on CUDA | Explicit `float64`→`float32` casts for MPS (eventually superseded by CPU pinning) |
| Checkpoint | Assumed `hybrik_hrnet.pth` | Explicitly configured to `hybrik_hrnet48_wo3dpw.pth` |

---

# 16. The Main Launcher and Unity Integration

## 16.1 The Initial Challenge

With all three scenarios operational on the same machine (Chapters 13–15), there remained the challenge raised at the beginning of this work (Chapter 10): having **three separate scripts/binaries requiring manual execution**, each accompanied by its own independently developed version of Unity's `AvatarController.cs` script:

- **Scenario A – SAM3DBody-cpp** (C++): Transmitted joint rotations as **Euler angles** (in degrees) using custom joint nomenclature (e.g., `hip`, `lShldr`, numbered fingers) via a `BVHWriter`.
- **Scenario B – AlphaPose + HybrIK** (Python): Transmitted rotations as **unit quaternions** `(x, y, z, w)` mapped across the 24 standard SMPL joints.
- **Scenario C – MMDetection + HybrIK** (Python): Utilized the same quaternion/SMPL schema as Scenario B, but paired with a distinct upstream detector.

This divergence resulted in **three mutually incompatible `AvatarController.cs` implementations**, each embedding its own calibration logic, anti-jitter smoothing/dead-zones, and root translation routines—valuable features that were duplicated and could not be interchanged across pipelines.

## 16.2 Provider Architecture (Strategy Pattern)

The adopted architectural solution is the **Strategy pattern**: rather than maintaining three distinct avatar controllers, the system uses **a single unified controller** that delegates raw data parsing and transformation to interchangeable provider components.

- **`IPoseProvider`** — A shared interface implemented by all pipelines (`void ApplyPose(string jsonPayload, Animator animator)`). The controller remains agnostic to *how* data is parsed or translated, expecting only processed rotations ready for application.
- **Dedicated Pipeline Providers** (`SAM3DProvider`, `HybrIKProvider`—the latter shared by Scenarios B and C due to their common quaternion/SMPL format). Each provider handles: parsing its respective JSON schema, mapping joint identifiers to avatar bones (e.g., `ConvertSamBoneName` for SAM3DBody's custom naming), initial rest-pose calibration, and smoothing.
- **Unified `AvatarController`**, which forwards incoming network payloads to whichever Provider is currently active.

## 16.3 Dynamic Pipeline Switching via Network Handshake

Providing multiple Providers does not on its own remove the inconvenience of manually configuring the Unity scene every time a different vision pipeline is tested. A **dynamic network-driven selection mechanism** was therefore designed across both ends of the pipeline:

**Python Side — `main_launcher.py`** (located in `ImageProcessing/test_vale/`, see Chapter 11): An interactive CLI control panel that prompts the user to select which pipeline to launch. Upon selection:

1. It immediately dispatches an initial UDP **handshake** datagram (e.g., `{"pipeline_type": "sam3dbody"}`, `"alphapose-hybrik"`, or `"mmdetection-hybrik"`) *before* initializing the vision model, ensuring Unity configures itself without waiting for heavy model weights to load.
2. It spawns the selected pipeline script or executable as a subprocess—`angoliAlphaPose_HybrIK.py`, `angoliMMdetection_HybrIK.py`, or the native C++ binary `SAM3DBody-cpp/build/fast_sam_3dbody_run` configured with the parameters established in Chapter 13 (`--cuda -1 --backbone .../backbone_fp32.onnx --detector libreyolo ...`)—which then continuously streams pose datagrams to the shared UDP port.

**Unity Side — `ConnectionManager.cs`:** A persistent listener initialized in `Play` mode, starting with **all Providers disabled** by default. When the handshake datagram arrives, it inspects `pipeline_type`, enables the corresponding Provider, and disables all others. Subsequent packets (pose data rather than handshakes) are routed directly to the active Provider. Switching pipelines on the fly (terminating the active Python or C++ process and selecting another from the menu) issues a new handshake that cleanly swaps active Providers without touching the Unity editor.

## 16.4 Separation of Concerns in Unity: `UDPReceiver` as a Pure Sensor

The project already included `UDPReceiver.cs`, which ran a background thread listening on the UDP socket. Integrating it into the new architecture required **narrowing its scope**: transitioning from an all-in-one component to a pure "sensor" that receives raw bytes and immediately forwards them to `ConnectionManager`, which alone interprets message semantics (handshake vs. pose data). Because `UDPReceiver` operates on a background thread while Unity mandates that scene scene graph objects (`Transform`, `Animator`) only be modified on the main thread, a helper component, **`UnityMainThreadDispatcher`**, was introduced to queue incoming payloads and execute them inside the main thread's `Update()` loop.

## 16.5 Runtime Execution Flow

1. **Play in Unity** → `ConnectionManager` listens on the UDP port, all Providers are disabled, and the avatar remains stationary in its bind pose.
2. The user executes `main_launcher.py` in Terminal and selects a pipeline from the interactive menu (SAM3DBody-cpp / AlphaPose+HybrIK / MMDetection+HybrIK).
3. Python immediately broadcasts the UDP handshake datagram specifying the selected pipeline type.
4. `UDPReceiver` captures the datagram and forwards it to `ConnectionManager`, which recognizes the handshake, enables the appropriate Provider, and deactivates all others.
5. The vision process launches (Python script or C++ binary with webcam input); from this point forward, each processed frame emits a JSON pose packet over UDP.
6. `ConnectionManager` classifies these incoming packets as pose data and routes them exclusively to the active Provider, which applies calibration offsets, smoothing filters, and skeletal rotations to the avatar in real time.

To switch pipelines, the user simply stops the active process, returns to `main_launcher.py`, and picks another option: no adjustments to the Unity scene are needed. With Scenario A integrated into this workflow (its Provider and handshake conventions having been planned into the architecture from the start, see §16.2), the objective formulated in Chapter 10—a unified entry point capable of running, evaluating, and demonstrating all three pipelines on a single machine with a single Unity scene—was fully accomplished, subject to the CPU throughput constraints of Scenario A detailed in Chapter 13.

---

# 17. Extension Conclusions

The engineering work documented across these chapters addressed three diverse categories of technical challenges, each typical of projects integrating heterogeneous software stacks (Python/CV, C++, Unity/C#) developed by different contributors, across differing platforms, brought together to operate on a single machine.

**Building the Shared Environment** (Chapter 12) introduced challenges absent in isolated per-project venvs: changes made for one pipeline risked silently breaking another. This occurred in practice with the NumPy/OpenCV mismatch between AlphaPose and MMDetection, which was resolved by strictly enforcing pinned versions; it was successfully avoided upfront between Python and the C++ toolchain for SAM3DBody-cpp by maintaining strict physical separation.

The **Installation and Configuration** phases (Chapters 13–15) required, across all three pipelines, modifying library source files directly—rather than restricting changes to application-level scripts—in order to bypass the common assumption in research repositories that an NVIDIA GPU with CUDA is universally available: from rewriting AlphaPose's `setup.py` and patching `mmdet/__init__.py`, to modifying `CMakeLists.txt` in SAM3DBody-cpp due to the absence of `librt` on macOS.

The **Runtime Debugging** phases resolved recurring challenges spanning: operating system platform differences (AVFoundation video capture threading and macOS privacy permissions), missing or incorrectly fetched binary assets (YOLO weights, uncompiled NMS routines, and Git LFS text pointers masquerading as ONNX models), and subtle algorithmic defects discovered during live execution (HybrIK output flattening, visual coordinate scale discrepancies, `args.gpus` type mismatches, and unsupported CPU operators).

Regarding **Performance**, the three scenarios exhibit a marked hierarchy when restricted to CPU compute: the HybrIK-based pipelines (Scenarios B and C), while throttled, remain usable in near real-time when paired with targeted optimizations (multithreading, reduced resolution, subsampled detection); Scenario A, centered on a massive Vision Transformer engineered for high-end GPUs, remains significantly below real-time frame rates even after dynamic quantization and aggressive software downscaling. This empirical finding enriches the qualitative comparison presented in the main thesis (Chapter 9) with concrete quantitative data regarding computational feasibility in hardware-constrained environments.

On the **Unity Integration** front (Chapter 16), the core challenge was software design rather than platform compatibility: three pipelines outputting divergent data formats (quaternions vs. Euler angles, distinct joint schemas) had produced three incompatible `AvatarController` scripts. The adopted solution—a Strategy pattern utilizing `IPoseProvider`, dynamic runtime activation via network handshakes, and strict separation between network reception (`UDPReceiver`) and message routing (`ConnectionManager`)—delivered a unified architecture (`main_launcher.py` on the Python side, a single Unity scene on the client side) that runs, compares, and switches among all three pipelines dynamically without requiring manual scene modifications: fulfilling the core goal set forth in Chapter 10.

Throughout every stage, the most effective methodology remained consistent: isolate one issue at a time, validate hypotheses with minimal diagnostic scripts or targeted log output before applying changes, and only then proceed to the next issue—avoiding batched, multi-variable modifications that obscure which specific intervention resolved a given failure.
