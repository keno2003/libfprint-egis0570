# Egis 0570 touch fingerprint support for Linux

Full-frame touch capture and matching for the **Egis `1c7a:0570`** fingerprint
sensor in libfprint. Tested successfully on **Acer Swift 1 (Swift SF114-32)**,
including fingerprint authentication after installation.

Reliable Linux fingerprint authentication is urgently needed by affected Acer
Swift owners. The relevant model-year range is 2018–2020. Testing is especially
welcome on **Acer Swift 3 SF314-52 (2018–2019)** and **Acer Swift 3 SF314-57
(2020)**, which have been reported with this sensor. Both Swift 3
models still need testing with these patches. 

## The problem and the change

The existing driver treats the sensor as a swipe reader. Enrollment can succeed
while later verification fails. It crops sensor frames into narrow strips and
uses the NBIS minutiae matcher. Our stationary captures contain ridge detail but
do not provide enough usable minutiae for that path.

The new path captures the full 114×57 frame during a stationary touch, subtracts
an empty-sensor reference, and compares SIFT image features with geometric
verification. Enrollment stores 12 touch samples. USB ID `1c7a:0571` stays on the
legacy driver. Existing fingerprints must be enrolled again after installation.

## What is in this repository

This is a patch repository, not a complete libfprint fork or an upstream release.
The patches apply to upstream commit
`6f9479c3d55f847c1b3769f28ceb99227f9858cf`.

- `0001-egis0570-pixman.patch`: independently fixes the legacy driver's
  missing pixman build dependency.
- `0002-egis0570-touch.patch`: adds touch capture, matching, template
  storage, build integration and storage tests.
- `UPSTREAM-DRAFT.md`: proposed upstream merge-request description.

No fingerprint images, enrollment templates, private capture logs or package
binaries are included.

## Build and test

Install a C/C++ toolchain, Git, Meson, Ninja, pkg-config, GLib/GIO development
files, libgusb, GObject Introspection and OpenCV development files. For a full
default-driver build, libfprint also needs the other drivers' dependencies such
as pixman, OpenSSL and libgudev. OpenCV 5 was tested; the OpenCV 4 build path is
not yet validated.

From a checkout of this patch repository:

```sh
git clone https://gitlab.freedesktop.org/libfprint/libfprint.git libfprint-source
git -C libfprint-source checkout -b egis0570-touch 6f9479c3d55f847c1b3769f28ceb99227f9858cf
git -C libfprint-source apply ../0001-egis0570-pixman.patch
git -C libfprint-source apply ../0002-egis0570-touch.patch
meson setup libfprint-source/build libfprint-source -Ddrivers=egis0570touch -Ddoc=false -Dinstalled-tests=false
meson compile -C libfprint-source/build
meson test -C libfprint-source/build --suite unit-tests --print-errorlogs
```

This builds an isolated sensor-only evaluation library; it does not install it.
Use a distribution package built with the default driver set for system testing,
so other fingerprint drivers remain available. Keep the distribution's original
package for rollback. Restart fprintd after replacing its library and re-enroll
through your normal fingerprint settings. Keep the sensor empty when starting
capture; vary finger placement slightly between stationary enrollment touches.

To roll back, reinstall the distribution's original libfprint package, restart
fprintd, and re-enroll for the original driver if needed. Do not remove password
authentication while evaluating the driver.

## Status

The cleaned patch builds against the pinned upstream revision and passes all
four core/storage unit-test targets. Hardware testing was performed on the
Swift SF114-32. It has not been accepted upstream.

Remaining work includes automated touch-driver USB replay coverage, broader
matcher evaluation, initial-calibration robustness, suspend/resume and reboot
testing, OpenCV 4 verification, and maintainer review of the new dependency.
A broader earlier suite run encountered an ETU905 replay failure; its baseline
status has not been established. The full test suite is not claimed to pass.

Upstream contributions belong on
[freedesktop GitLab](https://gitlab.freedesktop.org/libfprint/libfprint/-/merge_requests).
There is no upstream merge request yet.

## Attribution and license

The USB protocol is derived from libfprint's existing Egis driver and the Acer
Swift initialization patch. The matcher adapts LGPL code from
[smox/libfprint-elan-04f3-0c63](https://github.com/smox/libfprint-elan-04f3-0c63)
at commit `0f18837da8d691946673710d63a81b39d0782fca`.
Original copyright notices are retained in the patches. Driver additions are
licensed under LGPL-2.1-or-later; see `COPYING`.
