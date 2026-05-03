---
layout: post
title: "Adding Real Intelligence to Our Pipeline — YOLO on ONNX in C++, a GStreamer Plugin, and GstMeta"
date: 2026-04-24 12:00:00
description: Continuing our video analytics journey—export an Ultralytics model to ONNX, prove it in standalone C++, wrap it in a GStreamer element that draws boxes, then attach detection metadata to buffers for downstream elements.
tags: GStreamer, development, C++, YOLO, ONNX, ONNX Runtime, deep learning, video analytics, GstMeta
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

## The Journey Ahead

If you followed the [standalone YOLO + ONNX Runtime detour]({% post_url 2026-04-17-yolo-onnx-standalone-detector-cpp %}), you already proved the model outside the pipeline. Before that, in our [buffer and frame monitor article]({% post_url 2025-03-03-understanding-gstreamer-buffers-and-plugin-development %}), we got comfortable with pads, caps, and the chain function. That was **Phase 1**. Today is **Phase 2**: run a detector on each frame, draw boxes so you can _see_ the result, then stash the same detections in **GstMeta** so later elements do not have to run the network again.

We'll use weights from **Ultralytics** (YOLOv8, YOLO11, whatever checkpoint you like). If you are new to that workflow, start with the official repo and docs: [ultralytics/ultralytics](https://github.com/ultralytics/ultralytics). Export to **ONNX**, then run inference with **ONNX Runtime** in C++—the same story as the standalone article, only the pixel source changes.

### Where This Fits on the Roadmap

Let me tie it back to what we promised earlier:

1. **Phase 1 (done)**: Frame monitor—our hello world for buffers and plugins.
2. **Phase 2 (today)**: ONNX YOLO in C++, draw boxes on the video, then stash detections in **GstMeta**.
3. **Phase 3**: Object tracking — now we have something meaningful to track on each buffer.
4. **Phase 4**: Event detection and analytics.
5. **Phase 5**: Distribution and visualization.

## Why ONNX and ONNX Runtime?

When I first tried to jam TensorFlow or PyTorch directly into a GStreamer element, I spent more time fighting build systems than writing multimedia code. Think of **ONNX** as a **lingua franca** for models: you train or download weights in Python, you export once to a frozen graph, and your C++ code just loads that file.

**ONNX Runtime** is the engine on the other side. It has a reasonably friendly C++ API, it runs well on CPU, and when you're ready you can turn on GPU execution providers without rewriting your whole pipeline. Inside our element, the story stays the same mental picture as before: **map the buffer**, turn pixels into a **tensor**, call **`Run()`**, turn raw outputs into **boxes**, then either **draw** on the frame or **attach meta** (or both).

When I am building **production** plugins for our video analytics stack, I usually reach for **TensorRT** and **TensorFlow**—TensorRT for the fused inference path on NVIDIA hardware, and TensorFlow where our graphs or tooling still live in that ecosystem. For **this** lesson I picked **ONNX Runtime** on purpose: you can follow along on a normal laptop without tying the whole series to one vendor stack, and the GStreamer shape of the problem—pads, buffers, decode, **`Run()`**, meta—is the same either way; only the runtime under the hood changes.

## Step 1 — Getting a Model into ONNX

From a Python environment where you already have `ultralytics` installed, exporting feels almost too easy:

```bash
# Example: a tiny YOLO11—swap in yolov8n.pt or whatever you trained
yolo export model=yolo11n.pt format=onnx opset=12 simplify=True
```

You should see something like `yolo11n.onnx` show up next to your checkpoint. Before you write a single line of C++, do yourself a favor and open that file in [Netron](https://netron.app/). I can't count how many hours I've lost assuming the input was named `images` when the exporter quietly picked something else.

### What Are We Actually Decoding?

Exported YOLO graphs usually hand you a **big pile of candidate boxes**—think of it like a sorting room where every parcel looks important until you read the labels. In code, the dance is almost always: maybe **sigmoid**, filter by **score threshold**, convert box parameterisation to **x1, y1, x2, y2**, then **NMS**.

Some Ultralytics exports give you an **end-to-end** graph with NMS baked in; others expose a **raw head**. Treat the layout as **version-specific**, inspect _your_ ONNX file, and pin the `ultralytics` version in a comment when it works.

## Step 2 — Standalone C++ First

Before I touch `gst_pad_push` again, I always prove the ONNX file on still images—see the [standalone article]({% post_url 2026-04-17-yolo-onnx-standalone-detector-cpp %}) for the full recipe. Once that returns sensible boxes, you are holding the same functions you will call from the chain function.

## Step 3 — The GStreamer Plugin

The **chain function** is still that worker on the conveyor belt: a buffer arrives, you map it, you do work, you pass it on. The only difference is the work is heavier—letterbox, infer, NMS, **draw** rectangles so your eyes confirm what the network saw.

Start with a simple raw format like **BGR** `video/x-raw` and let **`videoconvert`** upstream deal with messier decoder output. If you draw **in place**, map read–write and respect **`GstVideoFrame`** stride.

Example pipeline while iterating:

```bash
gst-launch-1.0 filesrc location=clip.mp4 ! decodebin ! videoconvert ! \
  video/x-raw,format=BGR ! yolonnx model-path=/path/to/model.onnx ! videoconvert ! autovideosink
```

The element name `yolonnx` matches what we register in **[build-with-gstreamer](https://github.com/rosemary-crypto/build-with-gstreamer)** on the `lesson-03-yolo-onnx` branch (or `main` once merged).

## Step 4 — GstMeta: Sticky Notes on the Same Package

Drawing answers _does the neural net agree with my eyes?_ **Metadata** answers _what does the rest of the pipeline know without running inference again?_

Think of **`GstMeta`** as a **sticky note** on the same package that already holds the pixels. Downstream elements can read structured detections without another forward pass. Register your meta type once, add helpers like **`gst_buffer_add_detections_meta`**, fill a `GArray` (or similar) of box structs after inference, and read it downstream with **`gst_buffer_get_meta`**.

Watch out for **`transform_func`** when you add **queue** elements—buffers get copied and meta can vanish unless you teach GStreamer how to copy it. Do not store pointers into mapped pixel memory inside meta; use normalised coordinates or document the frame size the coordinates assume.

## Putting It Together in the Repo

The CMake layout, letterbox math, NMS, plugin source, and meta registration live in **[build-with-gstreamer](https://github.com/rosemary-crypto/build-with-gstreamer)** next to the earlier lessons. This post is the map; the repo is the runnable ground truth.

## Time to Experiment!

1. Export two different `ultralytics` versions and compare output tensor shapes—keep a tiny Python reference script next to your C++.
2. Wrap **`Session::Run`** with clock measurements like we did for FPS in the frame monitor.
3. Build a downstream element that only reads **`GstDetectionsMeta`** and logs class ids—no drawing.
4. Add **queue** elements and fix meta **`transform`** once meta disappears.
5. When boxes go sideways: `export GST_DEBUG=yolonnx:5`.

Remember, the best way to learn is still by breaking things and fixing them.

## Conclusion

Today we connected **Phase 1** to **Phase 2**: ONNX export, standalone proof, a **GStreamer** element that draws boxes, and **GstMeta** so the rest of the pipeline can grow without re-running YOLO on every hop.

Stay tuned for tracking and heavier analytics. 🚀
