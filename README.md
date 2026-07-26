# Roadside obstacle detection and vegetation mapping from a survey vehicle

A computer vision pilot I built as a freelance project for a Dutch geodata
company. Their survey van drives rural and suburban roads with a camera, LiDAR
and GPS on the roof. My job: from the camera frames alone, find the fixed
obstacles that matter for roadside mowing (trees, posts, signs, fences,
benches) and classify the vegetation structure along the verge, then put both
on their online map. Fixed price, three milestones, all three delivered and
accepted.

The client data and the trained models are confidential, so this repository is
the case study, not the deliverable. Everything below describes work I did and
numbers I measured.

## The starting point

The client had already tried this with off-the-shelf models. A zero-shot
detector reached 0.113 AP50 on their merged obstacle classes, and a segmentation
model pretrained on generic scenes could recognise trees but scored 0.000 IoU
on short grass, so it could not answer the one question a mowing planner asks.
Their training pipeline also split train and test frames randomly, which leaks
near-identical neighbouring frames across the split and makes every score look
better than it is. Part of the job was doing the evaluation honestly enough
that the numbers could be trusted.

## What I built

**Phase 1: data engineering.** The footage came as ROS2 recordings, 5-second
bursts of frames with heavy overlap between consecutive frames. I measured the
actual overlap with sparse optical flow (Lucas-Kanade plus a RANSAC similarity
fit, after testing and rejecting phase correlation, which underestimates
forward vehicle motion badly), used that to pick frames with real new content,
and grouped all footage by GPS location so that training and test data come
from different streets. I defined the class scheme with the client, wrote the
annotation guidelines, and annotated boxes and pixel masks myself. One detail
that mattered: the car bonnet fills almost half of every frame and mirrors the
sky and trees, so I fitted a curved exclusion polygon and applied it
identically at training and inference. Measured later, that mask alone is
worth 0.038 AP50.

**Phase 2: models.** Two models, both chosen so the client can use them
commercially, which rules out more of the ecosystem than people expect. AGPL
model families and several popular pretrained weights with non-commercial
terms were out; I kept a licence register documenting the lineage of
everything delivered. Detection: RF-DETR Small, fine-tuned after a
hyperparameter sweep, with offline repeat-factor sampling and augmentation for
the rare classes. Segmentation: I raced two designs, a frozen DINOv2 backbone
with a small convolutional head against a fully fine-tuned UPerNet ConvNeXt
pretrained on an off-road driving dataset. The frozen-backbone model won
clearly on held-out streets. Both models beat the client's existing setup by a
wide margin on data they had never seen.

**Phase 3: the pipeline.** One program that runs both models on every frame
and writes machine-readable output: one JSON per frame with detections,
vegetation shares, capture time, position and a model registry with weight
hashes, plus GeoJSON layers that load straight into the client's online map.

Three problems in this phase were more interesting than they looked.

The GPS metadata carried one fix per 5-second chunk, but the van covers 30 to
87 metres in that time, so every detection in a chunk would have landed on one
map pin up to 87 metres from the real object. The raw recordings contained the
full 5 Hz GPS track, so I parsed the ROS2 bags and interpolated a position for
every single frame using the message timestamps.

Raw detections make an unreadable map. An avenue of trees produces hundreds of
overlapping pins over sixty metres of road. I added a clustered layer that
merges sightings of one class along a stretch of road, which cut 3699 points
to 142 without losing any real object, and kept the raw layer next to it as
evidence.

Vegetation cannot be pinpointed at all, because it stretches. A verge is tall
grass for eighty metres and then scrub for thirty, and a row of dots does not
express that. So the vegetation is also exported as line segments that follow
the driven route, produced by smoothing the per-frame class sequence and
merging runs below a minimum length. The result reads like what a roadside
manager actually needs: tall grass for 84 metres, starting here.

The detector also had one honest failure in review: a pedestrian walking a dog
detected as a boulder at 0.91 confidence, because moving objects were outside
the annotation scope and the model reached for the nearest class it knew. I
added a second detector with general object knowledge as a filter: any
obstacle box lying mostly inside a person, animal or vehicle is flagged and
kept out of the map layers, but stays in the JSON with the overlap fraction
recorded, so nothing is deleted invisibly and the threshold can be re-tuned
from the delivered files.

## Results

Measured on streets held out from all training and tuning:

| | Client's previous setup | Delivered |
|---|---:|---:|
| Obstacle detection, merged AP50 | 0.113 | 0.464 |
| Vegetation, present-class mIoU | 0.438 | 0.818 |

The final pipeline processed all 514 pilot frames in 11 minutes on a single
T4, every frame with its own GPS position. It is fully deterministic: the
same input produces byte-identical output, which cost 0.005 AP50 against the
non-deterministic setting, and I documented the trade rather than hiding it.
The delivery included a validation that reproduces the figures reported at
the earlier milestones under the same settings, a reproducibility proof, install and run
instructions, and a sample input with its exact output so the client can
verify any installation in one command.

One number from the map layer that I liked: of the 713 metres of verge
surveyed, 77 percent was tall grass or herbaceous growth and only 2 percent
was already mown. That is the question the whole survey exists to answer, and
it comes out of the pipeline as metres of road, not pixels.

## Stack

Python, PyTorch, RF-DETR, DINOv2, OpenCV, pycocotools, mcap for the ROS2
bags, GeoJSON per RFC 7946, labelme for annotation, training on Colab GPUs.
Every delivered component under a commercial-compatible licence, with the
lineage documented.

## What is not here

The frames, annotations, trained weights, GPS data and delivered reports
belong to the client and stay private. If you are considering working with me
and want to talk through any part of this in more depth, I am happy to.
