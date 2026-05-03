---
layout: post
title: "Proving YOLO on ONNX in Standalone C++ — Before We Plug It Into GStreamer"
date: 2026-04-30 12:00:00
description: A short detour between Lesson 2 and Lesson 3—export a YOLO model to ONNX, run it from C++ with ONNX Runtime, and convince yourself the boxes are right before any GStreamer code touches the problem.
tags: C++, YOLO, ONNX, ONNX Runtime, deep learning, video analytics
categories: tutorials
featured: true
---

<style>
.post-content h2 {
  font-size: 2rem;
  font-weight: 600;
  color: #00543D;
  border-bottom: 2px solid #00543D;
  padding-bottom: 0.5rem;
  margin-top: 2rem;
  margin-bottom: 1rem;
}

.post-content h3 {
  font-size: 1.75rem;
  font-weight: 500;
  color: #264653;
  margin-top: 1.5rem;
  margin-bottom: 0.75rem;
}

.post-content h4 {
  font-size: 1rem;
  font-weight: 500;
  color: #e76f51;
  margin-top: 1.25rem;
  margin-bottom: 0.5rem;
  margin-left: 1.5rem;
}

.post-content h4 + * {
  margin-left: 1.5rem;
}

</style>

## A Quick Detour Before Lesson 3

In our [last article]({% post_url 2025-03-03-understanding-gstreamer-buffers-and-plugin-development %}) we built a frame monitor and got comfortable with buffers, pads, and the chain function. The next thing on our roadmap is **Phase 2**—a YOLO detector wrapped in a real GStreamer plugin.

But before I drop a neural network into a pipeline I want to know one thing for certain: **does the model even work outside GStreamer?** That is the entire point of this short detour. We'll export a YOLO checkpoint to ONNX, run it from C++ with ONNX Runtime on a few still images, and only then—in the next article—plug the same code into a GStreamer element.

If the boxes are wrong in a tiny console app, they'll still be wrong inside GStreamer, only by then you'll also be debugging caps negotiation and clocking at the same time. Trust me on this one.

## Why ONNX and ONNX Runtime?

When I first tried to jam TensorFlow or PyTorch directly into a GStreamer element, I spent more time fighting build systems than writing multimedia code. Think of **ONNX** as a **lingua franca** for models: you train or download weights in Python, you export once to a frozen graph, and your C++ code just loads that file.

**ONNX Runtime** is the engine on the other side. It has a reasonably friendly C++ API, it runs well on CPU, and when you're ready you can turn on GPU execution providers without rewriting your whole pipeline.

In production I usually reach for **TensorRT** on NVIDIA hardware, or **TensorFlow** where our tooling already lives there—but for this series I picked **ONNX Runtime** on purpose so you can follow along on a normal laptop.

## Step 1 — Getting a Model into ONNX

From a Python environment where you already have `ultralytics` installed, exporting feels almost too easy:

```bash
# Example: a tiny YOLO11—swap in yolov8n.pt or whatever you trained
yolo export model=yolo11n.pt format=onnx opset=12 simplify=True
```

You should see something like `yolo11n.onnx` show up next to your checkpoint. Before you write a single line of C++, do yourself a favor and open that file in [Netron](https://netron.app/). I can't count how many hours I've lost assuming the input was named `images` when the exporter quietly picked something else, or guessing output shapes from an old blog post while my actual graph had already moved on.

### What Are We Actually Decoding?

Exported YOLO graphs usually hand you a **big pile of candidate boxes**—think of it like a sorting room where every parcel looks important until you read the labels. In code, the dance is almost always:

1. Maybe apply **sigmoid** (depends whether the export already folded that in).
2. Throw away predictions below a **score threshold**.
3. Turn whatever format the head uses (centre and size, corners, etc.) into plain **x1, y1, x2, y2** in your image space.
4. Run **NMS** so the same car doesn't generate fifteen overlapping boxes.

Keep your thresholds and NMS in **one shared module**. You'll thank yourself later when the standalone tester and the GStreamer element both call the same functions.

Here is one more gotcha I learned the hard way: some Ultralytics exports give you an **end-to-end** graph where NMS already lives inside ONNX, and the output looks like a tidy `[1, N, 6]` table of boxes and scores. Others give you the **raw head**—a wide tensor like `[1, 84, 8400]` (YOLOv8) where four box coordinates and the class scores are all stacked along one axis. Treat the layout as **version-specific**, wire your decoder after you look at _your_ file, and pin the `ultralytics` version in a comment when it works. Upgrades are wonderful until they silently change a tensor name.

## Step 2 — The Standalone C++ Recipe

The recipe is always the same story:

1. **Resize or letterbox** into the network's expected height and width (match how the model was trained if you care about accuracy).
2. **Normalise** to `float32` in **NCHW** order—again, **match the export**, don't assume `[0,1]` if your graph expects something else.
3. Build an **`Ort::Value`** tensor and call **`Session::Run`**.
4. Feed the output tensors into your **decoder + NMS** and end up with something like:

```cpp
struct Detection {
  float x, y, width, height;
  float confidence;
  int class_id;
};
```

Here is a tiny skeleton of the Runtime call—error handling omitted so we can see the forest for the trees, but please add real checks in your repo:

```cpp
#include <onnxruntime_cxx_api.h>
#include <vector>

static std::vector<Detection> run_yolo_session(
    Ort::Session& session,
    const Ort::MemoryInfo& mem_info,
    const float* nchw_input,  // length = 1*3*H*W
    int H, int W)
{
  Ort::AllocatorWithDefaultOptions alloc;
  auto in_name = session.GetInputNameAllocated(0, alloc);
  std::vector<const char*> in_names{in_name.get()};

  std::array<int64_t, 4> shape{1, 3, H, W};
  Ort::Value in_tensor = Ort::Value::CreateTensor<float>(
      mem_info, const_cast<float*>(nchw_input), 1 * 3 * H * W,
      shape.data(), shape.size());

  auto out_name = session.GetOutputNameAllocated(0, alloc);
  std::vector<const char*> out_names{out_name.get()};

  auto outputs = session.Run(Ort::RunOptions{nullptr},
                             in_names.data(), &in_tensor, 1,
                             out_names.data(), 1);
  // Parse outputs[0] into Detection vector + NMS ...
  return {};
}
```

### Decoding a YOLOv8 Head

For a typical YOLOv8 export the output tensor is shaped `[1, 84, 8400]`—84 is `4 box coords + 80 class scores`, and 8400 is the number of candidate anchors after the model's internal grid math. The decode loop looks like:

```cpp
// Pseudocode shape: out_data is [1, num_attrs, num_anchors]
const int num_classes = num_attrs - 4;
for (int i = 0; i < num_anchors; ++i) {
  float cx = out_data[0 * num_anchors + i];
  float cy = out_data[1 * num_anchors + i];
  float bw = out_data[2 * num_anchors + i];
  float bh = out_data[3 * num_anchors + i];

  int   best_class = -1;
  float best_score = 0.0f;
  for (int c = 0; c < num_classes; ++c) {
    float s = out_data[(4 + c) * num_anchors + i];
    if (s > best_score) { best_score = s; best_class = c; }
  }
  if (best_score < conf_threshold) continue;

  // (cx, cy, w, h) in network coords -> (x, y, w, h) in original frame
  // ... scale by frame_w / net_w, frame_h / net_h ...
  // ... append to candidates, then NMS by class_id.
}
```

Once you have your candidate vector, sort by confidence and run a simple per-class IoU pass to drop overlapping boxes. If you already depend on **OpenCV**, `cv::dnn::NMSBoxes` saves you from reimplementing this for the tenth time in your life.

### A Word on Dependencies

On Ubuntu I usually pull **ONNX Runtime** from Microsoft's packages or vendor a known-good tarball into the repo so CI and my laptop agree. **OpenCV** is optional but lovely for the standalone tester—you get image loading, resizing, drawing, and `NMSBoxes` for free. Inside the GStreamer plugin we'll keep the dependency surface smaller and draw rectangles with plain pixel loops; for now, the standalone tester is allowed to be cushy.

## Step 3 — A Tiny Tester

The shape of the standalone program is intentionally boring:

1. Load a JPEG with OpenCV (or any image library).
2. Resize to your model's input size (`640x640` for typical YOLO exports).
3. Normalize to `float32 / 255.0` in NCHW order, RGB.
4. Run the session.
5. Decode + NMS.
6. Draw boxes on the original image and save to disk.

Open the output. Either the box lands on the dog, or it doesn't. That is the whole game. If it doesn't land where you expect, suspect (in this order): **input channel order** (BGR vs RGB), **normalization** (`/255` vs `/127.5 - 1`), **box format** (cx,cy,w,h vs x1,y1,x2,y2), and finally **the export itself**. About 90% of my "the model is broken" moments were one of those four.

## What Comes Next

Once your standalone tester puts boxes in the right places on still images, you are holding the same functions you'll call from a GStreamer chain function—only the **source** of the pixels changes. In the next article we'll wrap exactly this code into a `GstVideoFilter` plugin called `yolonnx`, draw boxes directly on the video, and attach the detections to each buffer as `GstMeta` so downstream elements (trackers, recorders, MQTT bridges) can read them without re-running inference.

The companion repo is **[build-with-gstreamer](https://github.com/rosemary-crypto/build-with-gstreamer)**. The standalone work for this article lives next to the plugin code so you can diff the two and see how little has to change.

See you in [Lesson 3]({% post_url 2026-05-03-gstreamer-yolo-onnx-plugin-detections-metadata %}) where we finally put this on a conveyor belt. 🚀
