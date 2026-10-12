# Licenses, Attribution, and Redistribution Checklist

> **Unofficial experimental binary distribution.** This document describes known upstream licensing and the actions that still need verification. **It is not a substitute for the actual license texts or a completed third-party notice audit.** This is informational guidance, not legal advice.

## Distribution covered

Repository: <https://github.com/jcarpio/MediaPipe-0.10.21-Linux-ARM64>

Release: `v0.10.21-arm64-alpha.1`, containing:

- `mediapipe-0.10.21-cp310-cp310-linux_aarch64.whl` (compiled MediaPipe Python/C++ code, model files, generated Python Protobuf files and native extensions).
- `mediapipe-0.10.21-opencv34-runtime-linux-aarch64.zip` (eight native OpenCV 3.4 libraries built through MediaPipe's Bazel/CMake dependency graph).
- `mediapipe-0.10.21-opencv34-runtime-linux-aarch64.zip.sha256`.

These files were built and tested on Ubuntu 22.04 Linux aarch64. **The release is experimental and may need supplementary notices inside its binaries/archives before it can be considered a complete redistributable package.**

## Primary upstream projects and licenses

| Component | Known upstream license | Official/source reference | Additional action |
| --- | --- | --- | --- |
| Google MediaPipe v0.10.21 | **Apache License 2.0** | [MediaPipe repository](https://github.com/google-ai-edge/mediapipe), [version tag](https://github.com/google-ai-edge/mediapipe/tree/v0.10.21), [LICENSE](https://github.com/google-ai-edge/mediapipe/blob/v0.10.21/LICENSE) | Distribute Apache-2.0 license text; retain upstream notices/attributions and identify local modifications. Review the tagged source's notices and bundled models. |
| OpenCV **3.4.x** (MediaPipe's Bazel archive was configured for `opencv-3.4.11`) | **3-clause BSD license** for OpenCV 3.4 | [OpenCV 3.4.11 source](https://github.com/opencv/opencv/tree/3.4.11), [LICENSE](https://github.com/opencv/opencv/blob/3.4.11/LICENSE) | Include **the exact applicable BSD-3-Clause license and copyright/disclaimer** with the binary runtime; audit compiled modules and third-party code. |
| Python OpenCV 4.5.4 used for testing | Ubuntu-installed system package; **not included** in either release asset | Ubuntu `python3-opencv` distribution | Document as an external requirement, not part of bundled binaries. |
| TensorFlow Lite, XNNPACK, Abseil, Protobuf, and other transitive code | **Component-specific**; not all verified in this release | Check exact third-party revisions/licenses from MediaPipe/Bazel dependency graph | Identify statically linked and bundled code and preserve required notices/licenses. Do not assume all dependencies are covered solely by the MediaPipe or OpenCV licenses. |
| `.tflite` models and `.binarypb` assets | **Model/resource-specific attribution must be verified** | MediaPipe v0.10.21 source and model download references | Record each model's provenance and its distribution conditions; ensure downloadable-model rights before redistributing. |

**Version-specific note:** OpenCV licensing changed in more recent releases. Do not use the license of an unrelated recent OpenCV version for these **3.4** binaries. Likewise, do not treat the `opencv-python` wheel packaging license as a substitute for the core OpenCV license. A BSD-3-Clause copyright notice must not be paraphrased in place of the full required text.

## Origin and modifications

- MediaPipe upstream release/tag: `v0.10.21`.
- MediaPipe's Bazel build used OpenCV archive `opencv-3.4.11` in `WORKSPACE` and produced native `libopencv_*.so.3.4` files for **ARM64**.
- Custom build work included: packaging-version adjustment in `setup.py`; adding the Protobuf `--experimental_allow_proto3_optional` generation flag for `protoc 3.12.4`; enabling Linux ARM64 OpenCV include paths for experiments; a verified KleidiAI archive/checksum adaptation following download issues; and CPU-mode Bazel build settings.
- Binary wheel tests required external OpenCV 3.4 and `PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python` with Python package `protobuf==4.25.8`.
- The distribution is **not official** and does not claim any upstream endorsement.

Before claiming exact reproducibility, publish the corresponding patches, pinned source revisions and checksums, Bazel command lines, model download list, and complete build provenance. GitHub-generated `Source code (zip)` / `Source code (tar.gz)` files contain a snapshot of **this repository at its tag**, **not automatically the full upstream MediaPipe 0.10.21 source or the precise source of all binaries**.

## Required actions before declaring the release license-complete

- [ ] Copy the **full** upstream MediaPipe Apache-2.0 license text into the repository (e.g. `LICENSES/MEDIAPIPE_APACHE-2.0.txt`) and any applicable `NOTICE` content. Preserve original notices and add prominent notices of modifications.
- [ ] Copy the **full** OpenCV 3.4.11 BSD-3-Clause license text (e.g. `LICENSES/OPENCV_BSD-3-CLAUSE.txt`) with the original copyright statements, conditions and disclaimer. Ensure it **accompanies the OpenCV runtime binary ZIP** or is supplied with it in a manner satisfying redistribution requirements; a link alone should not be assumed sufficient.
- [ ] Identify the exact provenance of all eight OpenCV `.so` files (upstream source revision/archive hash, build settings, checksums).
- [ ] Examine bundled and/or statically linked third-party C/C++ code using MediaPipe's dependency pins and `readelf`, package manifest and Bazel graph. Collect the **actual** required copyright, license and notice files for each.
- [ ] Review MediaPipe models, graphs, generated bindings and runtime assets for separate licenses, usage restrictions and attributions.
- [ ] Document the project-specific glue scripts/patches with their own applicable license and ownership; do **not** silently apply that license to Google/OpenCV code.
- [ ] Include an up-to-date `THIRD_PARTY_NOTICES.md` and full license files, and distribute accompanying notices with the relevant release ZIP/wheel assets as needed.
- [ ] Verify any obligations for dynamically versus statically linked dependencies, and confirm that no incompatible licenses are inadvertently bundled.
- [ ] Keep release notes accurate: Python 3.10, Linux aarch64, Ubuntu 22.04 tested, external OpenCV runtime, Protobuf workaround, no official affiliation.
- [ ] Rebuild updated runtime ZIP/wheel if necessary so license files and notices are delivered with the binaries; replace the release assets or publish a clearly identified successor release.

## Recommended repository layout

```text
README.md
INSTALLATION.md
LICENSES_AND_ATTRIBUTION.md
THIRD_PARTY_NOTICES.md       # Populate with verified component-specific notices
LICENSES/
  MEDIAPIPE_APACHE-2.0.txt  # Full exact upstream text; pending addition
  OPENCV_BSD-3-CLAUSE.txt   # Full exact 3.4.11 text; pending addition
patches/                     # Minimal auditable modifications
scripts/                     # Build, package and validation helpers
```

The sample filenames above indicate a **proposed** structure. They do not assert that those files are currently present or legally complete.

## Third-party notice template (do not fill speculatively)

For each binary-included component, use a record such as:

```text
Component: <exact project/module>
Upstream URL: <source repository>
Version or source commit: <exact pin>
How included: <shared / static / model / source / script>
Copyright holders: <as stated in original>
License identifier and full-text file: <verified>
Required NOTICE/attribution: <verbatim applicable text>
Modifications: <description or none>
Source archive / binary hash: <SHA-256>
```

Do not invent copyright owners, use a license identifier without confirming the correct source revision, or imply that a third-party audit has occurred when it has not.

## Useful authoritative references

- [MediaPipe source and license](https://github.com/google-ai-edge/mediapipe/tree/v0.10.21)
- [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)
- [OpenCV 3.4.11 source](https://github.com/opencv/opencv/tree/3.4.11)
- [OpenCV 3.4.11 LICENSE](https://github.com/opencv/opencv/blob/3.4.11/LICENSE)
- [GitHub: about releases and source archives](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases)

**Disclaimer:** No warranty or representation is made concerning compatibility, fitness, model accuracy, ABI stability, legal completeness, or the rights of third parties. End users must assess suitability and applicable licensing for their own deployments.
