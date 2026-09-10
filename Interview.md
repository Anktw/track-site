
The system uses YOLOv8 for object detection, ByteTrack for tracking and a custom line-crossing logic to count entries.


## ▶️ Usage


Place your input video inside the video folder and run the script:

python process_and_count.py

The processed video with analytics overlay will be saved to:

output/counting_video.mp4

The YOLO model will be automatically downloaded by Ultralytics on first run.

    ⚠️ Important:
    The counting line is currently fixed at the vertical center of the frame. For correct counting, the input video should be aligned, so that people cross near the middle of the frame. 

    Alternatively, you can manually adjust the line position in the script:   
    
        line_y = int(height * 0.6)  # example

## Possible Extensions

This project can be extended with:

    * real-time camera processing

    * entry/exit counting

    * REST API for analytics

    * dashboard visualization

    * deployment on edge devices (Raspberry Pi)

## Interview Questions and Answers

### 1. What problem does this project solve?

It detects and tracks people in a video and counts how many tracked people cross a virtual horizontal line. The result is saved as an annotated video with CCTV-style analytics.

### 2. What is the main processing pipeline?

The script opens the input video, reads frames, runs YOLO detection and tracking, filters for people, checks line crossings, draws overlays, writes the processed frame, and releases the video resources.

### 3. Why did you use YOLOv8?

YOLOv8 provides fast single-stage object detection with good accuracy and a convenient Ultralytics API. It is appropriate for near-real-time video processing and supports tracking through the same interface.

### 4. What does the `yolov8s.pt` model represent?

It is the YOLOv8 small pretrained model. The small variant offers a useful balance between detection accuracy, model size, and inference speed compared with larger YOLOv8 variants.

### 5. What dataset was the model trained on?

The pretrained YOLOv8 model is commonly trained on the COCO dataset, which includes a `person` class. This project uses that existing class rather than training a custom detector.

### 6. How does the script identify people?

Each detected box has a class index. The script converts that index to an integer and keeps only class `0`, which is the person class in the COCO label mapping.

### 7. What is the difference between detection and tracking?

Detection finds objects independently in each frame. Tracking associates detections across frames and assigns a persistent track ID, allowing the program to know whether the same person crossed the line.

### 8. Why is tracking needed for counting?

Counting detections in every frame would count the same person many times. Tracking gives each person an ID, so the program can count one crossing event per tracked object.

### 9. What does `persist=True` do?

It tells the Ultralytics tracker to maintain tracking state between successive frames. Without persistent state, IDs could be reinitialized for every frame.

### 10. Which tracker is used by this project?

The project uses the tracker selected by the Ultralytics `model.track` defaults, typically ByteTrack unless another tracker configuration is supplied. The README identifies ByteTrack as the intended tracker.

### 11. What is ByteTrack?

ByteTrack is a multi-object tracking algorithm that associates detections across frames using motion and bounding-box similarity. It can use both high- and lower-confidence detections to improve track continuity.

### 12. How is a person's position represented?

The script calculates the center of the bounding box using `cx = (x1 + x2) // 2` and `cy = (y1 + y2) // 2`. The vertical center coordinate is used for line-crossing detection.

### 13. How is the counting line positioned?

It is set to `height // 2`, so the line is placed at the vertical center of the frame. The position can be changed to another fraction of the frame height.

### 14. How does the line-crossing logic work?

For a tracked person, the script stores the previous center y-coordinate. It counts a crossing when the previous position is above the line and the current position is on or below it.

### 15. What does `previous_positions` store?

It maps each track ID to the most recent center y-coordinate. This provides the previous position needed to compare movement across the counting line.

### 16. Why is `counted_ids` a set?

A set provides fast membership checks and naturally stores unique track IDs. It prevents the same ID from increasing the count more than once.

### 17. What kind of movement does the current logic count?

It counts downward crossings only: the center must move from above the line to on or below it. Upward crossings are not counted by the current implementation.

### 18. How would you count exits as well as entries?

I would maintain separate counters. A transition from above to below would increment entries, while a transition from below to above would increment exits, with suitable debouncing and per-track state.

### 19. What is a weakness of using only the center point?

The center can jitter near the line, and a person may be counted at an unintuitive moment depending on bounding-box size. A small crossing band or a foot-point coordinate can make the event definition more reliable.

### 20. How could duplicate counts still happen?

If the tracker loses an object and later assigns a new ID, the same person could be counted again. Long occlusions, poor lighting, and crowded scenes increase this risk.

### 21. How would you reduce duplicate counts?

I would improve tracker configuration, use a crossing cooldown, retain short-term track history, and potentially use appearance features or a stronger re-identification tracker to reconnect lost tracks.

### 22. What happens when a detection has no track ID?

The script skips that box because it cannot reliably associate it with a person across frames. This avoids counting untracked detections as crossing events.

### 23. Why is `box.id` checked before reading the ID?

Tracking IDs may be unavailable when tracking fails or when a result has no associated track. Checking for `None` prevents invalid access and runtime errors.

### 24. How does the script read the input video?

It creates an OpenCV `VideoCapture` using the path `video/input.mp4` and repeatedly calls `cap.read()` until a frame cannot be read.

### 25. How does the script detect the end of the video?

`cap.read()` returns a boolean named `ret`. When `ret` is false, the loop breaks because there is no valid frame to process.

### 26. How is the output video created?

The script uses `cv2.VideoWriter` with the MP4V codec, the source FPS, and the original frame dimensions. Each processed frame is written with `out.write(frame)`.

### 27. Why must output dimensions match the writer dimensions?

Video writers expect frames with the configured width and height. A mismatch can produce corrupted output, dropped frames, or a video that cannot be opened correctly.

### 28. Why is `os.makedirs` used?

It ensures that the `output` directory exists before the writer tries to create the output file. `exist_ok=True` makes the operation safe when the directory already exists.

### 29. Why are `cap.release()` and `out.release()` important?

They close the input and output resources and flush the final encoded frames. Without them, files may be incomplete and camera or file handles may remain open.

### 30. Is the displayed FPS the source FPS?

No. The overlay FPS is calculated from wall-clock time between processing iterations, so it represents approximate processing speed rather than the original video frame rate.

### 31. Why can the displayed FPS be unstable?

Inference time varies with frame complexity, hardware, model size, and system load. The current calculation uses only the immediately preceding interval, so it can fluctuate significantly.

### 32. How would you calculate smoother FPS?

I would measure a rolling average over several frames or use total processed frames divided by elapsed processing time. This produces a more stable performance indicator.

### 33. What is the purpose of the transparent overlays?

They make the counting line and counter panel visible without completely hiding the video content. OpenCV blending combines an overlay with the original frame using an alpha value.

### 34. How does `cv2.addWeighted` work here?

It computes a weighted combination of two images. The alpha value controls how strongly the overlay contributes, while the complementary weight preserves visibility of the original frame.

### 35. Why use a custom corner bounding box?

The corner style is a visual choice that keeps the annotation less visually heavy than a complete rectangle while still showing the detected object's location.

### 36. What information is shown in the video header?

The header shows a camera identifier, the current timestamp, and the calculated processing FPS. This imitates information commonly displayed by CCTV monitoring systems.

### 37. What are the main dependencies?

The project uses Ultralytics for YOLO, OpenCV for video and drawing operations, NumPy and PyTorch-related packages for the ML stack, and LAP for tracking-related assignment operations.

### 38. Does this project train a model?

No. It performs inference with a pretrained model. Training or fine-tuning would be needed if the camera environment contains objects or conditions that the pretrained model handles poorly.

### 39. When would custom training be useful?

Custom training would help with unusual camera angles, severe occlusion, low resolution, domain-specific clothing, or a need to detect classes not represented adequately by the pretrained model.

### 40. What metrics would you use to evaluate detection quality?

I would use precision, recall, mAP at relevant IoU thresholds, and per-class performance. For counting specifically, I would also measure counting accuracy, missed crossings, false crossings, and ID switches.

### 41. What is IoU?

Intersection over Union compares the area where a predicted box and ground-truth box overlap with their combined area. It is commonly used to determine whether a detection matches the ground truth.

### 42. What can cause missed counts?

Missed detections, low confidence, occlusion, motion blur, a poorly placed line, tracker ID loss, and people moving too quickly between processed frames can all cause missed counts.

### 43. How would you improve performance on a CPU?

I would use a smaller model, reduce the input resolution, process every nth frame when acceptable, and avoid unnecessary drawing or copying. I would measure accuracy after each optimization.

### 44. How would you improve GPU performance?

I would use an available CUDA device, batch work where the workflow permits it, select an appropriate image size, and consider exporting the model to an optimized runtime such as TensorRT.

### 45. What is a tradeoff of reducing image resolution?

Inference becomes faster and uses less memory, but small or distant people may become harder to detect. The correct resolution depends on the camera view and the minimum object size that must be counted.

### 46. How would you process a live camera feed?

I would replace the file path with a camera index or stream URL, handle reconnection and dropped frames, and separate capture, inference, display, and persistence concerns so a slow inference step does not permanently block capture.

### 47. How would you make the line configurable?

I would move the line position to a configuration value or command-line argument, validate that it lies within the frame, and optionally support two points for an angled line.

### 48. How would you make the script more production-ready?

I would add configuration management, structured logging, input validation, error handling, model and device selection, output metadata, automated tests for counting logic, and monitoring for processing failures.

### 49. What tests would you write first?

I would test upward and downward transitions, no movement, repeated frames at the line, duplicate IDs, missing IDs, and multiple people crossing independently. These tests can exercise counting logic without running the full detector.

### 50. What are the next useful extensions for this project?

Useful extensions include separate entry and exit counts, configurable polygonal regions, live camera support, a REST API, dashboard analytics, event timestamps, database storage, model performance monitoring, and deployment on edge devices.