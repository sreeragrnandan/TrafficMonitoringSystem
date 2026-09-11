# Intelligent Traffic Monitoring System

**Adaptive traffic-light timing based on real-time vehicle counts, using OpenCV background subtraction and contour tracking.**

OpenCV image filters. Object detection by contours. Building a processing pipeline for further data manipulation.

## Problem Statement

Develop an intelligent traffic monitoring system that can control traffic lights based on the number of vehicles on each road.

## Project in a Nutshell

- Uses computer vision to count the number of vehicles on a road and adjust traffic-light timing accordingly.
- Uses the **MOG2 background-subtraction algorithm** on CCTV-style footage from a traffic post to detect moving vehicles.
- Main tools used: **OpenCV, scikit-video (sk-video), MySQL** (via `pymysql`).

## How It Works

The repo contains two companion notebooks that share the same detection pipeline (adapted from [creotiv/object_detection_projects – opencv_traffic_counting](https://github.com/creotiv/object_detection_projects/tree/master/opencv_traffic_counting)), applied to two different exit zones on the same video: [`TrafficIn.ipynb`](TrafficIn.ipynb) counts vehicles entering a junction, and [`TrafficOut.ipynb`](TrafficOut.ipynb) counts vehicles leaving it.

1. **Video source** – Both notebooks download the same sample traffic video (`road.mp4`) from S3 and read it frame-by-frame with `skvideo.io.vreader`.
2. **Background subtraction (MOG2)** – `cv2.createBackgroundSubtractorMOG2` is trained on the first 500 frames, then used on every subsequent frame to produce a foreground mask: `foreground_objects = current_frame - background_layer`.
3. **Filtering** – The raw foreground mask is thresholded and cleaned up with morphological closing, opening, and dilation (`filter_mask`) to fill small holes, remove noise, and merge adjacent blobs into solid shapes.
4. **Contour-based object detection** – `ContourDetection` runs `cv2.findContours` on the cleaned mask, filters out boxes smaller than a minimum width/height, and returns each detected object's bounding box and centroid.
5. **Vehicle tracking and counting** – `VehicleCounter` links centroids across frames into paths (matching each new point to the closest existing path within `max_dst`), and increments a running vehicle count whenever a tracked path crosses from outside into a predefined **exit zone** (`EXIT_PTS`/`exit_mask`).
6. **Pipeline runner** – A small `PipelineRunner`/`PipelineProcessor` framework chains `ContourDetection → VehicleCounter → Visualizer → CsvWriter` so each frame flows through detection, counting, drawing, and logging in order. `Visualizer` overlays bounding boxes, tracked paths, and a running vehicle count onto the output video; `CsvWriter` logs `time, vehicles` to `report.csv`.
7. **Simulating 4 lanes and writing to MySQL** – The vehicle count from the (single) processed video is written via `pymysql` into 4 tables (`road1`–`road4`) representing 4 lanes of a junction, scaled by a per-lane multiplier (`count`, `count*2`, `count*3`, `count*2`) to simulate different traffic volumes per lane. `TrafficIn.ipynb` **inserts** new `trafficin` rows keyed by frame number; `TrafficOut.ipynb` **updates** the matching row for that frame with `trafficout`, then reads back both `trafficin` and `trafficout` for each of the 4 roads to compute `inside = abs(trafficin - trafficout)` — an estimate of how many vehicles are currently waiting in each lane.
8. **Traffic-light decision (described, not yet implemented in code)** – Per the intended design, the `inside` vehicle count per lane is meant to drive a round-robin allocation of green-light time across the lanes, so busier lanes get proportionally more time; this logic generalizes to junctions with a different number of intersecting roads.

## Solution in Detail

- **MOG algorithm** — For background subtraction.
- `Foreground_objects = current_frame - background_layer`
- **Filtering** — Removing unwanted noise to make the object look bolder.
- **Object detection by contours** — using the standard `cv2.findContours` method.
- Contour detection is merged together with the background-subtraction/filtering step and the detection step into a single pipeline.
- A processor links detected objects across frames to build paths, and counts vehicles as they cross the exit zone.
- Vehicles entering and exiting the detection zone are counted, giving the number of vehicles currently in each zone.
- The number of vehicles inside each lane is used to decide the current and upcoming traffic light allocation.
- A **round-robin** distribution of the light cycle across lanes is used, and the approach generalizes to junctions with a different number of intersecting roads.

<!--
The recorded CCTV data is also intended to be used for a variety of other use cases like
pedestrian detection and signal allocation for pedestrians,
vehicle classification to intelligently map green-light duration,
and number-plate detection for other traffic-rule violations.
-->

## Repository Structure

| File | Description |
|---|---|
| [`TrafficIn.ipynb`](TrafficIn.ipynb) | Detection/tracking pipeline for vehicles entering the junction; writes `trafficin` counts to MySQL. |
| [`TrafficOut.ipynb`](TrafficOut.ipynb) | Same pipeline for vehicles leaving the junction; writes `trafficout` and derives the `inside` (in-lane) vehicle count. |
| `LICENSE` | MIT License. |
| `README.md` | This file. |

## Getting Started

Both notebooks are self-contained and were built to run in Google Colab.

1. Open `TrafficIn.ipynb` (and/or `TrafficOut.ipynb`) in Colab.
2. Run the setup cell — it installs `sk-video` and downloads the sample video (`road.mp4`) automatically.
3. Set up a MySQL database matching the connection details in the notebook (`host`, `port`, `user`, `passwd`, `database="Traffic"`) with tables `road1`–`road4` (columns including `frame`, `trafficin`, `trafficout`, `inside`), since the pipeline writes vehicle counts there as it runs.
4. Run the remaining cells in order to train the background subtractor, process the video, and write vehicle counts to the database and to `report.csv`.

### Dependencies

- OpenCV (`cv2`)
- scikit-video (`sk-video`)
- NumPy
- pymysql (MySQL connectivity)

## References

- Object detection: [opencv.org](https://opencv.org/)
- Vehicle count collection: [github.com/creotiv/object_detection_projects](https://github.com/creotiv/object_detection_projects)
- MySQL database connections: [journaldev.com/15539/python-mysql-example-tutorial](https://www.journaldev.com/15539/python-mysql-example-tutorial)

## License

This project is licensed under the MIT License — see [`LICENSE`](LICENSE) for details.
