# How the Egis 0570 contribution works

The driver has three jobs: collect a touch, extract its image features, and
compare those features with an enrollment. It changes libfprint, not fprintd,
PAM, or the login screen. The matcher adapts the LGPL ELAN reference credited
in the README; the USB commands come from the existing Egis/Acer work.

This guide describes the cleanup candidate, including patch 0004.
The 10-frame behavior was reported usable on SF114-32 before this cleanup.
The new worker and removal handling still need a hardware regression check.

## Read the code in this order

| File | Responsibility |
| --- | --- |
| `libfprint/drivers/egis0570touch.c` | USB operation, finger presence, enrollment, and reporting results |
| `libfprint/drivers/egis0570touch-match.cpp` | Image preparation, feature extraction, comparison, and template encoding |
| `libfprint/drivers/egis0570touch-match.h` | Small C interface to the C++ matcher; image dimensions |
| `libfprint/drivers/egis0570touch-proto.h` | Sensor command tables, USB endpoints, and packet sizes |
| `tests/test-egis0570touch.cpp` | Empty input, template round trips, and malformed storage |
| `tests/test-egis0570touch-driver.c` | Synthetic capture transitions and background-worker results/cancellation |

The Meson files select sources and OpenCV libraries. The existing swipe driver
keeps USB ID `1c7a:0571`; the touch driver claims `1c7a:0570`. The hardware
database follows that routing without changing the autosuspend values. The
separate pixman patch fixes a dependency of the old swipe driver.

The touch driver is included in the default driver set, so default builds now
require OpenCV. The runtime changes are specific to Egis 0570; other sensors'
matching code is unchanged.

## Why OpenCV remains

A separate experiment retried the existing NBIS matcher with background
subtraction, contrast and local-contrast processing, different image scales and
polarities, and alternative frame selection. It did not produce usable matching
between separate touches of the enrolled finger. This does not prove NBIS could
never work with this sensor, but no small, working replacement was found. The
working SIFT approach therefore remains, with its dependency stated explicitly.

## Authentication flow

1. `start_operation()` resets the operation counters. If the previous operation
   ended with a finger present, it waits for removal before accepting a new touch.
2. `run_state()` sends a command, consumes its reply, and receives image packets.
   Each image packet contains five 114-by-57 frames.
3. `process_packet()` subtracts the background for finger detection and collects
   ten foreground frames. It then pauses USB capture in `MATCH_FRAMES`.
4. `match_frames()` runs in a GLib worker. It selects one image from that batch,
   extracts features, and compares them against immutable template snapshots.
   There is no longer a list of five probe batches or a second comparison loop
   over that list.
5. `match_done()` runs on the main thread and reports the result. A successful
   touch does not need to be lifted before authentication completes.

The task holds the device alive, and the USB state machine does not modify the
frame buffers while the worker reads them. Cancellation is checked before
extraction and between template comparisons. It does not interrupt an OpenCV
call already in progress. USB cancellation waits for a complete request/reply
boundary to avoid leaving an unread reply for the next operation.

Verification asks whether one specified enrollment matches, so it stops at the
first sufficient sample. Identification chooses the highest-scoring enrollment
from a list, so it examines every candidate. Multiple enrollments therefore
remain more expensive than a single one.

## Enrollment flow

Enrollment still waits for a lift between touches. It keeps up to 50 foreground
frames, replacing the weakest foreground-signal frame when that buffer is full.
The matcher chooses the frame with the most features. Twelve accepted touches
become one saved enrollment. Feature count is a selection heuristic, not proof
of image quality. Enrollment data remains compatible with the earlier touch
packages; this cleanup does not require re-enrollment.

## Settings worth knowing

| Setting | Value | Meaning |
| --- | ---: | --- |
| `AUTH_FRAMES` | 10 | Frames collected for one authentication attempt |
| `MAX_FRAMES` | 50 | Maximum enrollment frame buffer |
| `ENROLL_STAGES` | 12 | Separate enrollment touches |
| `MIN_FEATURES` | 20 | Minimum extracted features before comparison |
| `MATCH_THRESHOLD` | 12 | Minimum geometrically consistent matches in both directions |
| `RELEASE_FRAMES` | 5 | Consecutive empty frames used to confirm removal |
| `EGIS0570_MIN_MEAN` | 20 | Foreground signal used for finger detection |

The counts mean different things. Ten frames are ten images, while a score of
twelve counts matching feature locations. Neither is a confidence percentage.
These values are preserved from the tested local implementation.

## What the matcher does

`prepare()` subtracts the empty-sensor reference, crops a three-pixel border,
normalizes contrast between the first and 99th percentiles, enlarges the image
twofold, and applies local contrast enhancement. SIFT then describes distinctive
image locations. Extraction chooses the frame with the most such locations.

Comparison finds nearby descriptors, rejects ambiguous nearest neighbours with
a 0.70 ratio test, and uses RANSAC to count correspondences consistent with a
rotation, translation, and uniform scale. The driver evaluates both directions
exactly once and uses the lower score. Previously, using the `MIN` macro directly
on function calls evaluated one expensive comparison a second time.

Version-1 storage contains a magic number, a version, a bounded feature count,
coordinates, and 128-byte descriptors. Loading checks the exact size, count,
version, and finite in-range coordinates. The format and matching parameters
are unchanged. Templates are biometric data even though they are not images.

C++ exceptions are contained at the C interface: failed extraction/storage
returns no result, and a failed comparison returns zero. Temporary C++ objects
are released automatically, including during exceptions.

## What is improved, and what is not established

The cleanup removes obsolete multi-batch bookkeeping, centralizes the frame and
quality constants, eliminates duplicate matcher calls, requires removal between
successive operations on the same device object, and moves expensive work off
the main loop. Thread coordination and meaningful tests add lines; fewer total
lines would not by itself make the driver easier to maintain or more correct.

Synthetic tests cover the ten-frame trigger, enrollment waiting for removal,
the enrollment buffer limit, refusal to reuse a held touch, empty-image retry,
pre-cancelled work, successful matching, and identification index handling.
They do not constitute USB replay coverage or a biometric accuracy study.

Known limits remain:

- On a newly created device object, the first calibration still assumes an
  empty sensor. A reference is retained across reopen when that same object
  knows a finger was left down; this is not a general calibration solution.
- Matching thresholds have not been validated for population false-accept or
  false-reject rates. Personal wrong-finger tests do not establish those rates.
- Full USB replay, OpenCV 4, suspend/resume, and broader hardware validation
  remain outstanding.
- Upstream still needs to review the OpenCV dependency and the shared legacy
  driver identifier. The cleanup preserves that identifier for existing templates.

The core design is bounded and understandable, but it is not described as
perfect, security-certified, or accepted upstream.
