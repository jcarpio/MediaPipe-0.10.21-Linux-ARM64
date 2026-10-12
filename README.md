# MediaPipe 0.10.21 on Linux ARM64 (aarch64)

> **Unofficial experimental community build. Not affiliated with Google or Yahboom.**
>
> **Status:** A 31 MB `mediapipe-0.10.21-cp310-cp310-linux_aarch64.whl` was built on Ubuntu 22.04 ARM64 and installed in a test virtual environment. `mp.solutions.hands`, `pose`, `face_mesh`, and `holistic` successfully initialized their TensorFlow Lite CPU graphs and processed a blank RGB image. `objectron` imports, but its inference has **not** been tested. The wheel currently **requires external OpenCV 3.4.11 shared libraries and a Protobuf workaround**; it is **not yet a standalone distributable**.

## Why this exists

Prebuilt MediaPipe **0.10.21** Linux aarch64 wheels were unavailable for the required setup. This work arose while migrating a Yahboom ROSmaster / ROS 2 Humble development VM from x86 VMware to Ubuntu 22.04 ARM64 in UTM on an Apple M4 Mac. The original Yahboom applications use the classic `mp.solutions` APIs extensively, so upgrading to newer MediaPipe Tasks interfaces would involve significant rewriting.

The aim is to preserve this MediaPipe version on ARM64, document a reproducible source build, and eventually supply a convenient, legally redistributable release.

## Verified scope

| Feature | State |
| --- | --- |
| Linux aarch64 / CPython 3.10 wheel build | **Passed** |
| Three native extensions packaged (`_framework_bindings`, `_pywrap_metadata_version`, `_pywrap_flatbuffers`) | **Passed** |
| Hands, Pose, Face Mesh, Holistic imports and blank-image CPU inference | **Passed** |
| Objectron module import | **Passed** |
| Objectron inference / detections on real images | **Not tested** |
| Real-time camera processing, accuracy, performance | **Not tested** |
| Raspberry Pi 5, other ARM64 operating systems, GPU | **Not tested** |
| Clean installation with no external shared libraries | **Not yet supported** |
| Complete Yahboom / physical ROS robot integration | **Not tested** |

The wheel was built with **MediaPipe v0.10.21**, **Bazel 6.5.0**, **Python 3.10**, and **CPU inference**. The Python `cv2` module used in the successful tests came from Ubuntu **OpenCV 4.5.4**. The compiled MediaPipe native extension, however, links to **eight Bazel-built OpenCV 3.4.11 libraries**. Mixing these OpenCV versions in one Python process needs further testing.

## Experimental GitHub release

The alpha release is available at **[v0.10.21-arm64-alpha.1](https://github.com/jcarpio/MediaPipe-0.10.21-Linux-ARM64/releases/tag/v0.10.21-arm64-alpha.1)**. It provides the CPython 3.10 ARM64 wheel and a **separate OpenCV 3.4.11 runtime archive**. These are experimental artifacts, **not** a fully self-contained or manylinux-certified installation.

The release also has a second-generation license/audit bundle prepared for publication:

- `mediapipe-0.10.21-arm64-licenses-audit-v2.zip`
- `mediapipe-0.10.21-arm64-licenses-audit-v2.zip.sha256`

**Note:** The audit-v2 files have been generated and copied to the Mac shared folder; their presence among GitHub release assets should be verified before relying on the release download links. Check each downloaded artifact against its published checksum. On Ubuntu, use `sha256sum -c filename.zip.sha256` in the directory containing both files.

### Licensing and redistribution status

A preliminary inventory has located license/notice files for **24 external Bazel repository names** observed in the native-linker input listings, plus the MediaPipe and OpenCV license texts. The expanded archive contains **42 collected license/notice documents**, including TensorFlow's extensive `THIRD_PARTY_NOTICES.txt`, and supporting audit reports. An inventory is **not** a legal certification and does not prove that every notice is applicable or that every obligation has been met.

The OpenCV libraries were verified as **version 3.4.11**, and their dynamic dependencies were inspected. `libopencv_imgcodecs.so.3.4` contains a `grfmt_webp.cpp` source-file symbol but does **not** declare a dynamic `libwebp` dependency; this does not establish whether any WebP-related third-party code is incorporated statically. The exact set of transitive components and applicable license obligations is still under review, including relevant codec/FFmpeg considerations. No claim is made that the release is fully audited for redistribution compliance.

## Repository and release layout

```text
mediapipe-0.10.21-linux-aarch64/
├── README.md
├── patches/                 # Planned: reviewable source patches
├── scripts/                 # Planned: reproducible build/tests
├── LICENSES/                # Planned: license inventory in Git
└── release-notes/           # Planned: verification notes
```

The layout above is a **planned repository structure**, not a statement that these directories already exist. Put documentation and source patches in Git; put the binary wheel and SHA256 checksums in a **GitHub Release**, rather than committing binary outputs or build caches to Git history. Do not publish a release advertised as standalone until the packaging work below is completed.

## Building on Ubuntu 22.04 aarch64 (experimental instructions)

### 1. Install prerequisites

```bash
sudo apt update
sudo apt install -y build-essential git curl wget cmake ninja-build \
  pkg-config python3-dev python3-venv python3-pip python3-setuptools \
  protobuf-compiler libopencv-dev libssl-dev zip unzip
python3 --version         # Python 3.10.x
pkg-config --modversion opencv4  # Tested: 4.5.4
protoc --version          # Ubuntu may supply 3.12.x
```

Install **Bazelisk for Linux ARM64** using its official release instructions; MediaPipe's `.bazelversion` selects Bazel **6.5.0**. Confirm `bazel --version` before building.

```bash
mkdir -p ~/build_projects && cd ~/build_projects
git clone --branch v0.10.21 --depth 1 \
  https://github.com/google-ai-edge/mediapipe.git
cd mediapipe
```

### 2. Configure Bazel

These settings were used for the successful build. They apply to the builder's user account.

```bash
mkdir -p "$HOME/bazel-distdir"
cat > "$HOME/.bazelrc" <<EOF_BAZEL
build --jobs=4
build --distdir=$HOME/bazel-distdir
build --repo_env=HERMETIC_PYTHON_VERSION=3.10
EOF_BAZEL
```

**KleidiAI caveat:** The original build encountered TLS/certificate download problems for a GitLab-hosted KleidiAI archive, then a SHA mismatch because GitLab had repackaged the ZIP. The pinned source revision was `40a926833857fb64786e02f97703e42b1537cb57`. The builder verified archive files against this Git revision, placed the exact archive in `~/bazel-distdir` and modified the `WORKSPACE` checksum to match the **verified archive bytes**. Do not copy an unchecked SHA value or disable integrity checking. A minimal patch and verifiable download procedure need to be published with the repository.

### 3. Apply source changes

The working MediaPipe source had these edits:

- `setup.py`: change `__version__ = 'dev'` to `__version__ = '0.10.21'`.
- `setup.py`, `GeneratePyProtos._generate_proto()`: insert `'--experimental_allow_proto3_optional'` between `self._protoc` and `'-I.'` to support Ubuntu's `protoc 3.12` when generating `proto3 optional` fields.
- `third_party/opencv_linux.BUILD`: for the optional system-OpenCV route, enable header glob `include/opencv4/opencv2/**/*.h*`, include path `include/opencv4/`, and ARM64 header `include/aarch64-linux-gnu/opencv4/opencv2/cvconfig.h`. Check that the corresponding header exists in `/usr/include`.
- `WORKSPACE`: only if necessary, use the provenance-verified KleidiAI archive checksum described above.

**Important:** Even after testing system OpenCV settings, the successful wheel remained linked to Bazel's own OpenCV **3.4.11** libraries (`opencv_cmake`). Do not assume `--link-opencv` changed the resulting linkage; check `readelf` and `ldd`.

### 4. Build the wheel

```bash
cd ~/build_projects/mediapipe
python3 -m venv ~/build_projects/mediapipe-build-env
source ~/build_projects/mediapipe-build-env/bin/activate
python -m pip install 'pip<25' setuptools wheel packaging \
  'numpy==1.26.4' 'PyYAML==6.0.2'
export MEDIAPIPE_DISABLE_GPU=1
set -o pipefail
python setup.py bdist_wheel 2>&1 | tee ~/mediapipe_wheel_build.log
ls -lh dist/mediapipe-0.10.21-cp310-cp310-linux_aarch64.whl
```

In the project, `setup.py build_py --link-opencv bdist_wheel` was also tested. Always inspect the resulting shared libraries: the wheel actually tested still required OpenCV **3.4.11**, not the Ubuntu system's 4.5 libraries.

If the build stops, `setup.py` may have temporarily modified `mediapipe/__init__.py`, root `__init__.py` and `third_party/BUILD`. Inspect `git status --short` and their corresponding `.backup` files before rerunning. **Do not routinely delete `bazel-bin`, `build/` or the Bazel cache.**

### 5. Inspect wheel native modules and dependencies

```bash
python - <<'PY'
from pathlib import Path
from zipfile import ZipFile
wheel = next(Path('dist').glob('mediapipe-0.10.21*.whl'))
with ZipFile(wheel) as z:
    for n in z.namelist():
        if n.endswith('.so'):
            print(n)
PY
readelf -d bazel-bin/mediapipe/python/_framework_bindings.so | grep NEEDED
ldd bazel-bin/mediapipe/python/_framework_bindings.so
```

The tested wheel contains:

```text
mediapipe/python/_framework_bindings.cpython-310-aarch64-linux-gnu.so
mediapipe/tasks/cc/metadata/python/_pywrap_metadata_version.cpython-310-aarch64-linux-gnu.so
mediapipe/tasks/python/metadata/flatbuffers_lib/_pywrap_flatbuffers.cpython-310-aarch64-linux-gnu.so
```

## How the current experimental build was tested

> This recipe runs only on a builder that has the matching Bazel-built OpenCV libraries. It is **not** the final end-user installation procedure.

### 1. Install the wheel and Python basics into a test virtual environment

```bash
python3 -m venv ~/build_projects/mediapipe-test-env
source ~/build_projects/mediapipe-test-env/bin/activate
cd ~/build_projects/mediapipe
python -m pip install --no-deps \
  dist/mediapipe-0.10.21-cp310-cp310-linux_aarch64.whl
python -m pip install 'numpy==1.26.4' 'protobuf==4.25.8' \
  absl-py attrs flatbuffers matplotlib sentencepiece
```

Because `--no-deps` was used, the package metadata still reports `jax`, `jaxlib`, `opencv-contrib-python`, and `sounddevice` as missing. The successful four-model test did **not** install these. Other MediaPipe features may require them; this is not a complete dependency-resolution claim.

### 2. Stage eight OpenCV 3.4.11 shared libraries privately

```bash
cd ~/build_projects/mediapipe
VENV="$HOME/build_projects/mediapipe-test-env"
mkdir -p "$VENV/lib/mediapipe-opencv"
find -L bazel-bin/_solib_aarch64 -name 'libopencv_*.so.3.4' -type f \
  -exec cp -L {} "$VENV/lib/mediapipe-opencv/" \;
LD_LIBRARY_PATH="$VENV/lib/mediapipe-opencv" \
  ldd "$VENV/lib/python3.10/site-packages/mediapipe/python/_framework_bindings.cpython-310-aarch64-linux-gnu.so" \
  | grep -E 'opencv|not found'
```

The eight libraries are `core`, `calib3d`, `features2d`, `highgui`, `video`, `videoio`, `imgcodecs`, and `imgproc`; they totaled approximately **11 MB**. Confirm **no `not found`** entries. Never replace ABI-incompatible libraries with fake SONAME symlinks.

### 3. Expose Ubuntu Python `cv2` temporarily

The tested builder provided `/usr/lib/python3/dist-packages/cv2.cpython-310-aarch64-linux-gnu.so`, version **4.5.4**. In the test venv:

```bash
printf '%s\n' '/usr/lib/python3/dist-packages' \
  > "$VENV/lib/python3.10/site-packages/ubuntu-system-packages.pth"
```

**Warning:** This exposes other Ubuntu packages as well and can cause conflicts, including a Matplotlib `Axes3D` warning observed in the test. Using `cv2` OpenCV 4.5.4 alongside native OpenCV 3.4 is an **unverified mixed-version configuration**, suitable only for controlled testing.

### 4. Work around old generated Protobuf files

Protobuf 4.25.8 rejected Python descriptors generated by Ubuntu `protoc 3.12.x`. The working test selected pure-Python Protobuf instead:

```bash
export PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python
```

This may be slower. The release-quality solution is to regenerate the bundled `_pb2.py` files with a compatible modern `protoc`, build a new wheel and verify that the native Protobuf runtime loads them.

## CPU smoke test: four working models

Run from `/tmp` (not the MediaPipe source tree). Save the following as `test_smoke.py`:

```python
import mediapipe as mp
import numpy as np

print('MediaPipe:', mp.__version__)
image = np.zeros((480, 640, 3), dtype=np.uint8)
checks = [
    ('Hands', lambda: mp.solutions.hands.Hands(static_image_mode=True)),
    ('Pose', lambda: mp.solutions.pose.Pose(static_image_mode=True)),
    ('Face Mesh', lambda: mp.solutions.face_mesh.FaceMesh(static_image_mode=True)),
    ('Holistic', lambda: mp.solutions.holistic.Holistic(static_image_mode=True)),
]
for name, factory in checks:
    with factory() as detector:
        detector.process(image)
    print(name, 'OK')
print('Objectron import:', mp.solutions.objectron)
```

```bash
cd /tmp
VENV="$HOME/build_projects/mediapipe-test-env"
PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python \
LD_LIBRARY_PATH="$VENV/lib/mediapipe-opencv${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}" \
python /path/to/test_smoke.py
```

**Observed:** `Hands OK`, `Pose OK`, `Face Mesh OK`, `Holistic OK`, with TensorFlow Lite's **XNNPACK CPU delegate** initialized. A blank image is only an initialization/inference smoke test; it does not establish detection accuracy. Objectron imports but **inference remains untested**.

Optional Objectron verification (not yet performed):

```python
import mediapipe as mp
import numpy as np
with mp.solutions.objectron.Objectron(
        static_image_mode=True, model_name='Shoe', max_num_objects=1) as detector:
    result = detector.process(np.zeros((480, 640, 3), dtype=np.uint8))
print('Objects:', len(result.detected_objects or []))
```

## Next milestone: a self-contained OpenCV wheel

**Yes, this is worth doing.** But a wheel bundled with OpenCV is **self-contained only for a defined platform/ABI**, not literally *universal*. Python `cp310`, architecture `aarch64`, glibc version and other native dependencies still impose compatibility constraints. A `linux_aarch64` wheel is not automatically a standards-compliant `manylinux` wheel.

Recommended engineering plan:

1. **Resolve the OpenCV strategy:** ideally make MediaPipe native code and Python `cv2` use a compatible single OpenCV build. Alternatively, bundle the eight OpenCV 3.4.11 shared objects in an isolated directory and carefully validate coexistence with system OpenCV 4.5.4. Both approaches require real testing.
2. **Audit licensing:** include Apache-2.0 attribution for MediaPipe plus OpenCV and any redistributed native dependencies. Evaluate FFmpeg and codec licensing before including transitive libraries.
3. **Investigate `auditwheel`:** install it in a separate packaging environment and run `auditwheel show dist/*.whl`. Consider `auditwheel repair` with the required `.so` files visible to the repair environment. Verify what it actually vendors, the ELF `RPATH/RUNPATH`, generated tags and glibc requirements. Repair may fail or require extra build work.
4. **Fix Protobuf:** regenerate Python protobuf sources using a supported `protoc`, and verify imports/inference without `PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python`.
5. **Clean-room testing:** install only the release artifacts in a *fresh* Ubuntu 22.04 aarch64 VM, without Bazel paths or custom library environment variables. Test real images/video, all five classic solutions, `cv2`, and at least one ROS 2 Yahboom example.
6. **Publish after validation:** attach the wheel, SHA256 checksum, source patches, reproducibility instructions, test outputs, and complete license notices to a GitHub Release. State clearly which devices and distributions were **actually tested**.

An **interim GitHub Release** may instead ship the existing wheel **plus a separate archive of its OpenCV 3.4 `.so` files and a launcher** that uses a local `LD_LIBRARY_PATH`, with clear experimental labeling. This is less convenient but avoids claiming the existing wheel is standalone.

## Publication checklist

- [ ] Export minimal reviewed patches for `WORKSPACE`, `setup.py`, and the OpenCV BUILD file.
- [ ] Record full upstream commit SHA, build environment, toolchain versions and CPU architecture.
- [x] Collect initial upstream and dependency license notices in a separate audit-v2 archive (legal applicability review pending).
- [ ] Review redistributed OpenCV and transitive native dependencies and determine applicable license obligations.
- [x] Create SHA256 checksum for audit-v2 ZIP.
- [ ] Confirm checksums are available for every downloadable binary artifact.
- [ ] Verify all ELF linkages, wheel tags and clean-room installation.
- [ ] Fix old Protobuf-generated descriptors or document the required workaround.
- [ ] Test Objectron, actual images/video, and the chosen `cv2`/native OpenCV combination.
- [ ] Label release **experimental** until all dependencies are packaged and verified.

## References and credits

- [MediaPipe upstream source](https://github.com/google-ai-edge/mediapipe)
- [Bazel](https://bazel.build/) / [Bazelisk](https://github.com/bazelbuild/bazelisk)
- [OpenCV](https://opencv.org/)
- [auditwheel](https://github.com/pypa/auditwheel)

MediaPipe, OpenCV, and other third-party components retain their respective copyrights and licenses. This is an independent community effort without endorsement by their maintainers or Yahboom.
