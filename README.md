# Deep SORT

## Introduction

This repository contains code for *Simple Online and Realtime Tracking with a Deep Association Metric* (Deep SORT).
We extend the original [SORT](https://github.com/abewley/sort) algorithm to
integrate appearance information based on a deep appearance descriptor.
See the [arXiv preprint](https://arxiv.org/abs/1703.07402) for more information.

## Dependencies

The code is compatible with Python 2.7 and 3. The following dependencies are
needed to run the tracker:

* NumPy
* sklearn
* OpenCV

Additionally, feature generation requires TensorFlow (>= 1.0).

To install an anaconda environment for this code, run the following:

```sh
conda create --name deepsort python=3.5 opencv numpy scikit-learn tensorflow ffmpeg
```

## Installation

First, clone the repository:
```
git clone https://github.com/nwojke/deep_sort.git
```
Then, download pre-generated detections and the CNN checkpoint file from
[here](https://drive.google.com/open?id=18fKzfqnqhqW3s9zwsCbnVJ5XF2JFeqMp).

*NOTE:* The candidate object locations of our pre-generated detections are
taken from the following paper:
```
F. Yu, W. Li, Q. Li, Y. Liu, X. Shi, J. Yan. POI: Multiple Object Tracking with
High Performance Detection and Appearance Feature. In BMTT, SenseTime Group
Limited, 2016.
```
We have replaced the appearance descriptor with a custom deep convolutional
neural network (see below).

## Running the tracker

The following example starts the tracker on one of the
[MOT16 benchmark](https://motchallenge.net/data/MOT16/)
sequences.
We assume resources have been extracted to the repository root directory and
the MOT16 benchmark data is in `./MOT16`:
```
python deep_sort_app.py \
    --sequence_dir=./MOT16/test/MOT16-06 \
    --detection_file=./resources/detections/MOT16_POI_test/MOT16-06.npy \
    --min_confidence=0.3 \
    --nn_budget=100 \
    --display=True
```
Check `python deep_sort_app.py -h` for an overview of available options.
There are also scripts in the repository to visualize results, generate videos,
and evaluate the MOT challenge benchmark.

## Generating detections

Beside the main tracking application, this repository contains a script to
generate features for person re-identification, suitable to compare the visual
appearance of pedestrian bounding boxes using cosine similarity.
The following example generates these features from standard MOT challenge
detections. Again, we assume resources have been extracted to the repository
root directory and MOT16 data is in `./MOT16`:
```
python tools/generate_detections.py \
    --model=resources/networks/mars-small128.pb \
    --mot_dir=./MOT16/train \
    --output_dir=./resources/detections/MOT16_train
```
The model has been generated with TensorFlow 1.5. If you run into
incompatibility, re-export the frozen inference graph to obtain a new
`mars-small128.pb` that is compatible with your version:
```
python tools/freeze_model.py
```
The ``generate_detections.py`` stores for each sequence of the MOT16 dataset
a separate binary file in NumPy native format. Each file contains an array of
shape `Nx138`, where N is the number of detections in the corresponding MOT
sequence. The first 10 columns of this array contain the raw MOT detection
copied over from the input file. The remaining 128 columns store the appearance
descriptor. The files generated by this command can be used as input for the
`deep_sort_app.py`.

**NOTE**: If ``python tools/generate_detections.py`` raises a TensorFlow error,
try passing an absolute path to the ``--model`` argument. This might help in
some cases.

## Training the model

To train the deep association metric model we used a novel [cosine metric learning](https://github.com/nwojke/cosine_metric_learning) approach which is provided as a separate repository.

## Highlevel overview of source files

In the top-level directory are executable scripts to execute, evaluate, and
visualize the tracker. The main entry point is in `deep_sort_app.py`.
This file runs the tracker on a MOTChallenge sequence.

In package `deep_sort` is the main tracking code:

* `detection.py`: Detection base class.
* `kalman_filter.py`: A Kalman filter implementation and concrete
   parametrization for image space filtering.
* `linear_assignment.py`: This module contains code for min cost matching and
   the matching cascade.
* `iou_matching.py`: This module contains the IOU matching metric.
* `nn_matching.py`: A module for a nearest neighbor matching metric.
* `track.py`: The track class contains single-target track data such as Kalman
  state, number of hits, misses, hit streak, associated feature vectors, etc.
* `tracker.py`: This is the multi-target tracker class.

The `deep_sort_app.py` expects detections in a custom format, stored in .npy
files. These can be computed from MOTChallenge detections using
`generate_detections.py`. We also provide
[pre-generated detections](https://drive.google.com/open?id=1VVqtL0klSUvLnmBKS89il1EKC3IxUBVK).

## Show results

```sh
usage: show_results.py [-h] --sequence_dir SEQUENCE_DIR --result_file
                       RESULT_FILE [--detection_file DETECTION_FILE]
                       [--update_ms UPDATE_MS] [--output_file OUTPUT_FILE]
                       [--show_false_alarms SHOW_FALSE_ALARMS]

Siamese Tracking

optional arguments:
  -h, --help            show this help message and exit
  --sequence_dir SEQUENCE_DIR
                        Path to the MOTChallenge sequence directory.
  --result_file RESULT_FILE
                        Tracking output in MOTChallenge file format.
  --detection_file DETECTION_FILE
                        Path to custom detections (optional).
  --update_ms UPDATE_MS
                        Time between consecutive frames in milliseconds.
                        Defaults to the frame_rate specified in seqinfo.ini,
                        if available.
  --output_file OUTPUT_FILE
                        Filename of the (optional) output video.
  --show_false_alarms SHOW_FALSE_ALARMS
                        Show false alarms as red bounding boxes.
```

```python
python3 show_results.py \
    --sequence_dir=./MOT16/test/MOT16-06 \
    --detection_file=./resources/detections/MOT16_POI_test/MOT16-06.npy \
    --result_file=./track_result/MOT16-06-result.txt \
    --output_file=MOT16-06.mp4
```

## Generating videos

```sh
usage: generate_videos.py [-h] --mot_dir MOT_DIR --result_dir RESULT_DIR
                          --output_dir OUTPUT_DIR
                          [--convert_h264 CONVERT_H264]
                          [--update_ms UPDATE_MS]

Siamese Tracking

optional arguments:
  -h, --help            show this help message and exit
  --mot_dir MOT_DIR     Path to MOTChallenge directory (train or test)
  --result_dir RESULT_DIR
                        Path to the folder with tracking output.
  --output_dir OUTPUT_DIR
                        Folder to store the videos in. Will be created if it
                        does not exist.
  --convert_h264 CONVERT_H264
                        If true, convert videos to libx264 (requires FFMPEG
  --update_ms UPDATE_MS
                        Time between consecutive frames in milliseconds.
                        Defaults to the frame_rate specified in seqinfo.ini,
                        if available.
```

```python
python generate_videos.py \
    --mot_dir=./MOT16/test/ \
    --result_dir=./track_result \
    --output_dir=./results \
    --convert_h264=true \
    --update_ms=5
```

## Citing DeepSORT

If you find this repo useful in your research, please consider citing the following papers:

    @inproceedings{Wojke2017simple,
      title={Simple Online and Realtime Tracking with a Deep Association Metric},
      author={Wojke, Nicolai and Bewley, Alex and Paulus, Dietrich},
      booktitle={2017 IEEE International Conference on Image Processing (ICIP)},
      year={2017},
      pages={3645--3649},
      organization={IEEE},
      doi={10.1109/ICIP.2017.8296962}
    }

    @inproceedings{Wojke2018deep,
      title={Deep Cosine Metric Learning for Person Re-identification},
      author={Wojke, Nicolai and Bewley, Alex},
      booktitle={2018 IEEE Winter Conference on Applications of Computer Vision (WACV)},
      year={2018},
      pages={748--756},
      organization={IEEE},
      doi={10.1109/WACV.2018.00087}
    }
