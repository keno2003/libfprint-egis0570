Title: Draft: egis0570: add full-frame touch capture and feature matching

Reliable Linux fingerprint authentication is urgently needed by affected Acer
Swift owners. This contribution was tested on an Acer Swift 1 (Swift SF114-32)
with Egis USB sensor 1c7a:0570. Reports from other Swift models, including laptops
from 2019–2021, are welcome; support across that lineup has not been established.

On an Acer Swift SF114-32 with USB sensor 1c7a:0570, enrollment using the existing
swipe driver succeeds but subsequent verification fails. Full-frame stationary
captures contain ridge detail, while the NBIS path yields too few usable minutiae
in our captures.

Add a touch capture path using the existing USB protocol with Acer initialization,
12 enrollment samples, and OpenCV SIFT descriptors filtered by a similarity
transform. Route 0570 to this path and preserve the existing 0571 implementation.
Include bounded template decoding and storage tests. Old image templates need
re-enrollment.

Tested successfully on Acer Swift 1 (Swift SF114-32), including fingerprint
authentication after installation, as reported by the hardware tester.

Core and storage unit tests and hwdb validation passed. The broader suite had an
ETU905 replay failure whose baseline status has not been established. This draft
still needs a new touch replay test, full-suite validation, OpenCV 4 testing,
and review of enrollment quality, initial calibration and the matching threshold.

The matcher is adapted from LGPL code in smox/libfprint-elan-04f3-0c63 at commit
0f18837da8d691946673710d63a81b39d0782fca, with attribution retained. No biometric
captures or templates are included.

The cleaned review branch builds against upstream master at 6f9479c3; all four
core/storage unit-test targets pass on that revision.
