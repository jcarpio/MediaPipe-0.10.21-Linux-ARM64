# MediaPipe 0.10.21 for Linux ARM64 — Installation and Verification

> **Unofficial experimental release (`v0.10.21-arm64-alpha.1`).** This guide describes the configuration validated on **Ubuntu 22.04 aarch64, CPython 3.10**. The build is **not standalone**, is **not an official Google MediaPipe distribution**, and has **not been certified for general use**.

## Why this project exists

MediaPipe 0.10.21's classic `mp.solutions` API is used in educational robotics and computer-vision projects, including applications in the Yahboom ROSmaster / ROS 2 Humble ecosystem. The original project used an x86 Ubuntu VM. Moving that environment to ARM64 (Ubuntu 22.04 in UTM on Apple Silicon) required rebuilding the native MediaPipe extensions: a compatible upstream 0.10.21 Linux aarch64 wheel was not available for that environment.

The community build preserves the existing Python APIs rather than requiring applications to migrate to newer MediaPipe Tasks APIs.

## Compatibility and validated results

| Component | Result |
| --- | --- |
| Ubuntu 22.04 Linux aarch64 + CPython 3.10 | **Built and tested** |
| MediaPipe 0.10.21 wheel native extensions | **Present** |
| Hands, Pose, Face Mesh, Holistic | **Successfully imported and ran CPU inference on a blank image** |
| TensorFlow Lite XNNPACK CPU delegate | **Initialized** |
| Objectron | **Module import succeeded; inference not tested** |
| Real photos, webcam/video, benchmark/accuracy | **Not tested** |
| Clean VM installation using release files only | **Not yet validated** |
| Raspberry Pi OS, other distros/Python versions | **Not validated** |
| YOLO, ROS 2/Yahboom real hardware integration | **Not validated** |

**Known technical compromises:** the native extension links against **eight OpenCV 3.4 shared libraries** shipped separately; Python `cv2` in the successful test was Ubuntu's **OpenCV 4.5.4**, which may cause ABI/symbol conflicts in other applications. Generated Python Protobuf modules required `PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python` with `protobuf==4.25.8`. Do not install or rename OpenCV 3.4 libraries globally.

## Download

Release page: <https://github.com/jcarpio/MediaPipe-0.10.21-Linux-ARM64/releases/tag/v0.10.21-arm64-alpha.1>

Download these **three assets** (not GitHub's auto-generated “Source code” archives):

1. `mediapipe-0.10.21-cp310-cp310-linux_aarch64.whl`
2. `mediapipe-0.10.21-opencv34-runtime-linux-aarch64.zip`
3. `mediapipe-0.10.21-opencv34-runtime-linux-aarch64.zip.sha256`

Place all three in the same working directory. The ZIP contains eight native libraries, a per-virtualenv installer, and internal `SHA256SUMS` (depending on the released ZIP content). **Check actual ZIP contents before running its scripts.**

### Verify the download

```bash
cd ~/Downloads  # or wherever the release assets were saved
sha256sum -c mediapipe-0.10.21-opencv34-runtime-linux-aarch64.zip.sha256
unzip -t mediapipe-0.10.21-opencv34-runtime-linux-aarch64.zip
unzip -l mediapipe-0.10.21-opencv34-runtime-linux-aarch64.zip
sha256sum mediapipe-0.10.21-cp310-cp310-linux_aarch64.whl
```

Compare the wheel's SHA-256 with the digest displayed by GitHub for the release asset. **A checksum obtained from the same repository as the artifact verifies transfer integrity, not independently established authenticity.** Review unfamiliar scripts before executing them.

## Ubuntu 22.04 prerequisites

```bash
uname -m                # must be aarch64
python3 --version       # must be Python 3.10.x
sudo apt update
sudo apt install -y python3-venv python3-pip python3-opencv \
  libopencv-dev unzip ca-certificates
/usr/bin/python3 -c 'import cv2; print(cv2.__version__, cv2.__file__)'
```

The tested `cv2` module is Ubuntu's `/usr/lib/python3/dist-packages/cv2.cpython-310-aarch64-linux-gnu.so`, version 4.5.4. The package is **not** guaranteed to work with arbitrary PyPI OpenCV wheels.

## Create an isolated Python environment

```bash
python3 -m venv "$HOME/mediapipe-arm64-test"
VENV="$HOME/mediapipe-arm64-test"
"$VENV/bin/python" -m pip install --upgrade pip
"$VENV/bin/python" -m pip install --no-deps \
  ./mediapipe-0.10.21-cp310-cp310-linux_aarch64.whl
"$VENV/bin/python" -m pip install \
  'numpy==1.26.4' 'protobuf==4.25.8' \
  absl-py attrs flatbuffers matplotlib sentencepiece
```

`--no-deps` is intentional: this build still declares `jax`, `jaxlib`, `opencv-contrib-python`, and `sounddevice` as upstream requirements, but the four-model CPU smoke test did not need those packages installed. **Other MediaPipe APIs can require them.** `pip check` may therefore report missing requirements; this is an experimental partial installation, not a completely resolved PyPI package.

The test setup exposed Ubuntu's `cv2` inside the virtual environment with a `.pth` file:

```bash
printf '%s\n' '/usr/lib/python3/dist-packages' \
  > "$VENV/lib/python3.10/site-packages/ubuntu-system-packages.pth"
```

**Caution:** this exposes *all* system Python packages in that directory, not just `cv2`, and may lead to library collisions (a Matplotlib `Axes3D` warning occurred on the original builder). Restrict this method to a disposable test virtual environment.

## Install the matching private OpenCV runtime

```bash
unzip mediapipe-0.10.21-opencv34-runtime-linux-aarch64.zip
cd mediapipe-0.10.21-opencv34-runtime-linux-aarch64
# Inspect the script before running it:
less install_opencv_runtime.sh
bash install_opencv_runtime.sh "$VENV"
```

The installation script checks architecture and bundled library hashes and copies the eight `.so.3.4` files to:

```text
$VENV/lib/mediapipe-opencv/
```

It also creates `$VENV/run-with-mediapipe-opencv`, a wrapper that sets `LD_LIBRARY_PATH` **for the requested process only**, instead of modifying `/usr/lib` or your shell profile.

**Never** create symlinks from `.so.3.4` to `.so.4.5d`: their ABIs are not interchangeable.

## Verify native loader dependencies

```bash
SO="$VENV/lib/python3.10/site-packages/mediapipe/python/_framework_bindings.cpython-310-aarch64-linux-gnu.so"
LD_LIBRARY_PATH="$VENV/lib/mediapipe-opencv" ldd "$SO" \
  | grep -E 'opencv|not found'
```

Expected: eight OpenCV libraries resolve inside `$VENV/lib/mediapipe-opencv/`, and **no `not found` entries**. `ldd` checks library resolution, not inference correctness. Avoid running `ldd` on unknown, untrusted binaries.

## Import smoke test

Run the test outside any cloned `mediapipe/` source directory:

```bash
cd /tmp
PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python \
  "$VENV/run-with-mediapipe-opencv" "$VENV/bin/python" - <<'PY'
import cv2
import mediapipe as mp
print('OpenCV Python:', cv2.__version__)
print('MediaPipe:', mp.__version__)
for name in ('hands', 'pose', 'face_mesh', 'holistic', 'objectron'):
    print(name, getattr(mp.solutions, name))
PY
```

`PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python` is required because the wheel's `_pb2.py` modules were generated by `protoc 3.12.4`; their descriptors were rejected by the installed `protobuf 4.25.8` native backend. The pure-Python parser is a compatibility workaround and may be slower.

## CPU inference smoke test

```bash
cd /tmp
PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python \
  "$VENV/run-with-mediapipe-opencv" "$VENV/bin/python" - <<'PY'
import mediapipe as mp
import numpy as np
image = np.zeros((480, 640, 3), dtype=np.uint8)
tests = [
    ('Hands', lambda: mp.solutions.hands.Hands(static_image_mode=True)),
    ('Pose', lambda: mp.solutions.pose.Pose(static_image_mode=True)),
    ('Face Mesh', lambda: mp.solutions.face_mesh.FaceMesh(static_image_mode=True)),
    ('Holistic', lambda: mp.solutions.holistic.Holistic(static_image_mode=True)),
]
for name, factory in tests:
    with factory() as detector:
        detector.process(image)
    print(name + ': OK')
PY
```

These four operations completed on the original BUILDER VM. TensorFlow Lite's XNNPACK CPU delegate initialized, and non-fatal feedback-manager warnings were emitted. **A blank-image test proves that models load and run, not that detections are accurate.** Objectron inference remains untested.

## Troubleshooting

| Symptom | Likely explanation / next check |
| --- | --- |
| `not a supported wheel on this platform` | Confirm Linux `aarch64`, CPython **3.10** and matching wheel filename. |
| `libopencv_*.so.3.4: cannot open shared object file` | Install the runtime ZIP into this virtualenv and use the wrapper; check `ldd`. |
| `No module named cv2` | Check Ubuntu `python3-opencv` and the test-only `.pth` exposure. |
| `Descriptors cannot be created directly` | Set `PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python` for the launched process. |
| `No module named jax` / other missing Python requirements | Feature-specific dependency still missing. Do not install incompatible versions blindly. |
| Segfault, symbol errors or unusual image results | Potential OpenCV 3.4 / 4.5 coexistence issue; stop and reproduce in an isolated environment. |
| Matplotlib `Axes3D` warning | Possible conflict after exposing system `dist-packages`; not the cause of the successful four-model smoke test. |

## Known limits and next release

This is **not a universal ARM64 wheel**, nor a manylinux-certified package. The wheel has a `linux_aarch64` platform tag and depends on system libraries. A future build should use newer `protoc` for generated Python bindings; test whether native Protobuf works; vendor and audit the OpenCV dependency with `auditwheel` as appropriate; eliminate the mixed OpenCV runtime; resolve declared Python dependencies; run clean-VM, camera, Objectron and ROS 2 integration tests. Do not copy BUILDER-specific absolute paths into user environments.

## Attribution and redistribution

Read [`LICENSES_AND_ATTRIBUTION.md`](LICENSES_AND_ATTRIBUTION.md) before redistribution. **This documentation does not constitute a completed license audit.** The ZIP/Wheel contain third-party native code; applicable licenses and required notices must be preserved and reviewed. This is an unofficial community build, not endorsed by Google, OpenCV, Yahboom or Ubuntu.
