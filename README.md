# project
open source


import cv2
import numpy as np

# 1. Load pre-trained YOLO model and labels
model_cfg = "yolov4.cfg"  # YOLO config file
model_weights = "yolov4.weights"  # YOLO weights file
coco_labels = "coco.names"  # COCO dataset labels

# Load class labels
with open(coco_labels, "r") as f:
    labels = [line.strip() for line in f.readlines()]

# Define categories for clustering
categories = {
    "vehicles": ["car", "bus", "truck", "motorbike", "bicycle"],
    "animals": ["dog", "cat", "bird", "horse"],
    "household_items": ["chair", "sofa", "bed", "tvmonitor"],
}

# Initialize YOLO model
net = cv2.dnn.readNetFromDarknet(model_cfg, model_weights)
net.setPreferableBackend(cv2.dnn.DNN_BACKEND_OPENCV)
net.setPreferableTarget(cv2.dnn.DNN_TARGET_CPU)

# 2. Load image or video
image_path = "input.jpg"  # Replace with your image or video file path
image = cv2.imread(image_path)
(h, w) = image.shape[:2]

# 3. Prepare the image for YOLO
blob = cv2.dnn.blobFromImage(image, 1 / 255.0, (416, 416), swapRB=True, crop=False)
net.setInput(blob)

# Get YOLO layer names
ln = net.getLayerNames()
ln = [ln[i - 1] for i in net.getUnconnectedOutLayers()]

# Perform forward pass
layer_outputs = net.forward(ln)

# 4. Process detections
boxes, confidences, class_ids = [], [], []
for output in layer_outputs:
    for detection in output:
        scores = detection[5:]
        class_id = np.argmax(scores)
        confidence = scores[class_id]
        if confidence > 0.5:
            box = detection[0:4] * np.array([w, h, w, h])
            (center_x, center_y, width, height) = box.astype("int")

            x = int(center_x - (width / 2))
            y = int(center_y - (height / 2))

            boxes.append([x, y, int(width), int(height)])
            confidences.append(float(confidence))
            class_ids.append(class_id)

# Apply Non-Maxima Suppression (NMS)
idxs = cv2.dnn.NMSBoxes(boxes, confidences, 0.5, 0.4)

# Group detected objects by categories
detected_objects = {}
for key in categories:
    detected_objects[key] = []

if len(idxs) > 0:
    for i in idxs.flatten():
        label = labels[class_ids[i]]
        for category, items in categories.items():
            if label in items:
                detected_objects[category].append(label)

# Print categorized results
for category, items in detected_objects.items():
    print(f"{category.capitalize()}: {items}")

# 5. Draw bounding boxes on the image
for i in idxs.flatten():
    x, y, w, h = boxes[i]
    color = (0, 255, 0)
    label = f"{labels[class_ids[i]]}: {confidences[i]:.2f}"
    cv2.rectangle(image, (x, y), (x + w, y + h), color, 2)
    cv2.putText(image, label, (x, y - 10), cv2.FONT_HERSHEY_SIMPLEX, 0.5, color, 2)

# Show the output image
cv2.imshow("Output", image)
cv2.waitKey(0)
cv2.destroyAllWindows()
