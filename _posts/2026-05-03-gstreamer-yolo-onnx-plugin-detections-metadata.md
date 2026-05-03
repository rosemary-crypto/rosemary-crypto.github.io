---
layout: post
title: "Adding Real Intelligence to Our Pipeline — YOLO on ONNX in C++, a GStreamer Plugin, and GstMeta"
date: 2026-05-03 11:00:00
description: Wrap the standalone YOLO ONNX detector from the previous article in a GStreamer plugin, draw boxes on the video, and—the part that really matters—attach detections to each buffer with GstMeta so the rest of the pipeline can reuse them.
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

Remember how in our [Lesson 2 article]({% post_url 2025-03-03-understanding-gstreamer-buffers-and-plugin-development %}) we built a frame monitor and got comfortable with buffers, pads, and the chain function? That was **Phase 1** on our roadmap. And in [last week's pre-Lesson detour]({% post_url 2026-04-30-yolo-onnx-standalone-detector-cpp %}) we built a tiny standalone YOLO ONNX detector in C++—the model bit, with no GStreamer in sight.

Today we're finally stepping into **Phase 2** by gluing the two together: a GStreamer plugin called **`yolonnx`** that runs the same kind of inference per frame, draws boxes on the video, and—the part I really want you to walk away with—**attaches the detections to each buffer as `GstMeta`** so the rest of the pipeline can read them without re-running the network.

This article is mostly about the **GStreamer side**—the plugin shape, the properties, and the meta API. If you want the model-loading and post-processing details, those live in the [previous article]({% post_url 2026-04-30-yolo-onnx-standalone-detector-cpp %}).

### Where This Fits on the Roadmap

1. **Phase 1 (done)**: Frame monitor—our hello world for buffers and plugins.
2. **Phase 2 (today)**: A YOLO ONNX detector wrapped as a GStreamer element, with detections attached as **GstMeta**.
3. **Phase 3**: Object tracking — now we have something meaningful to track on each buffer.
4. **Phase 4**: Event detection and analytics.
5. **Phase 5**: Distribution and visualization.

## Step 1 — Stand on `GstVideoFilter`'s Shoulders

In Lesson 2 we wrote our own chain function and pushed buffers around by hand. That was great for learning, but for a video element that produces video of the same shape it consumes, GStreamer has a much friendlier base class: **`GstVideoFilter`**. It handles caps negotiation, gives us a properly mapped **`GstVideoFrame`**, and lets us focus on the per-frame work.

Our element subclasses it and only has to implement two interesting hooks:

- **`set_info`** — called once when the upstream caps are known, so we can stash the video format and size.
- **`transform_frame_ip`** — called per frame, with the input frame already mapped read–write so we can draw on it in place.

A couple of practical notes from getting this wrong the first time:

- We declare caps as `video/x-raw,format=(string){ BGR, RGB }`. Either layout is fine, and the helper that draws boxes flips channel order based on which one we got. **`videoconvert`** upstream takes care of the messier formats the decoder might produce.
- The frame plane stride is **not** always `width * 3`. We always go through `GST_VIDEO_FRAME_PLANE_STRIDE` when drawing.
- When something looks off, `GST_DEBUG` is still our best friend, same as in Phase 1.

## Step 2 — Properties Over Hardcoding

A plugin that hardcodes its model path and thresholds is a plugin that ages badly. The element exposes seven properties so you can tune behaviour from `gst-launch` without recompiling:

| Property         | Default | Purpose                                                  |
|------------------|---------|----------------------------------------------------------|
| `model-path`     | `""`    | Path to the ONNX file. Empty = synthetic detection mode. |
| `conf-threshold` | `0.25`  | Drop predictions below this score.                       |
| `iou-threshold`  | `0.45`  | NMS overlap cutoff.                                      |
| `draw-boxes`     | `true`  | Paint rectangles directly on the video.                  |
| `attach-meta`    | `true`  | Attach `GstDetectionsMeta` to each buffer.               |
| `input-width`    | `640`   | Network input width.                                     |
| `input-height`   | `640`   | Network input height.                                    |

The `draw-boxes` and `attach-meta` split is on purpose. Set both to `true` while you're debugging, then flip `draw-boxes=false` once a downstream element starts consuming meta on its own—you don't want to re-render frames just to confirm something a sticky note already says.

A useful side benefit: if you launch the element with no `model-path`, it falls back to a **synthetic moving box**. That sounds silly, but it's the best teaching feature in the whole plugin—you can verify drawing, meta attach, and downstream readers without an ONNX file in sight.

While you're iterating, the pipeline looks like:

```bash
gst-launch-1.0 filesrc location=clip.mp4 ! decodebin ! videoconvert ! \
  video/x-raw,format=BGR ! yolonnx model-path=/path/to/yolo11n.onnx \
  draw-boxes=true attach-meta=true ! videoconvert ! autovideosink
```

## Step 3 — Inference, Without the Drama

Inside `transform_frame_ip` the work is exactly the recipe from the standalone post, just on a `GstVideoFrame` instead of a `cv::Mat`:

1. Lazy-load the ONNX session on the first frame, so we honour whatever `model-path` was set at element configuration time.
2. Resize the frame into the network's expected size, normalize to `float32 / 255.0` in NCHW, RGB.
3. Run the session.
4. Decode the YOLOv8-style output (`[1, 84, 8400]` for an 80-class model), apply NMS, and return a `std::vector<DetectionBox>`.
5. If `attach-meta` is set, append the boxes to a fresh `GstDetectionsMeta` on the buffer.
6. If `draw-boxes` is set, paint green rectangles in place.
7. Return `GST_FLOW_OK` and let the buffer continue down the pipeline.

A handy debugging hook: at the bottom of inference, you can drop a single line like

```cpp
g_print("[yolonnx] frame=%u class=%d score=%.2f box=(%d,%d,%d,%d)\n",
        self->frame_count, det.class_id, det.confidence,
        det.x, det.y, det.width, det.height);
```

inside the loop and immediately see in the terminal which boxes the model is producing—class id, score, position, size. Comment it out once you trust the numbers; leave it commented in the source so future-you can re-enable it the next time something goes weird.

### Drawing Now vs. Fancy Overlays Later

For learning, I honestly recommend **drawing directly on the video** first. It's the fastest feedback loop: either the box lands on the dog or it doesn't. Later you can get fancy with **`cairooverlay`** (a Cairo-based overlay element) or **`gdkpixbufoverlay`**, or a separate branch that only emits meta. But a few nested `for` loops to paint green lines in BGR is enough to carry you into Phase 3.

## Step 4 — GstMeta: Sticky Notes on the Same Package

Drawing answers the question, *does the neural net agree with my eyes?* **Metadata** answers a different question: *what does the rest of the pipeline know without running inference again?*

Think of **`GstMeta`** as a **sticky note** you slap on the same package that already holds the pixels. Downstream elements—trackers, recorders, MQTT bridges—can peel off that note and read structured detections without paying the cost of another forward pass.

[GStreamer's meta system](https://gstreamer.freedesktop.org/documentation/additional/design/gstmeta.html) lets you attach custom structs to a **`GstBuffer`** without copying the whole frame. The pattern looks like this:

1. **Register** your meta type once with `gst_meta_api_type_register` and `gst_meta_register`.
2. Write little helpers like `gst_buffer_add_detections_meta` and `gst_buffer_get_detections_meta` so the rest of your code stays readable.
3. After inference, **fill** a `GArray` of detection structs and attach it to the buffer.
4. Downstream, **get** the meta and consume it until the buffer is recycled.

Our two structs—the per-detection one and the meta itself—look like this:

```cpp
typedef struct _GstDetection {
  gfloat x;
  gfloat y;
  gfloat width;
  gfloat height;
  gfloat confidence;
  gint   class_id;
} GstDetection;

typedef struct _GstDetectionsMeta {
  GstMeta meta;
  GArray *detections;   /* each entry is a GstDetection */
} GstDetectionsMeta;
```

The two helpers we wrap around the GStreamer API are:

```cpp
GstDetectionsMeta *gst_buffer_add_detections_meta(GstBuffer *buffer);
GstDetectionsMeta *gst_buffer_get_detections_meta(GstBuffer *buffer);
```

A few lessons I have the bruises from:

- **`transform_func` is not optional.** The first time I dropped a `queue` between `yolonnx` and a downstream consumer, my meta quietly vanished. The fix is to register a transform callback with `gst_meta_register` so when GStreamer copies a buffer, it knows how to copy your meta along with it. We implement it up front, copying the `GArray` contents into a fresh meta on the destination buffer—learn from the bruises, do it on day one.
- **Lifetime.** Don't store pointers into the **mapped** pixel memory inside meta. Plain coordinates and scores are fine because they live in the meta's own `GArray`. If downstream might resize, store coordinates in **normalised** `[0, 1]` form, or clearly document that the numbers only make sense at the current buffer's width and height.
- **Threading.** Inference is slow. Some day we'll talk about workers and queues and keeping **PTS** aligned with delayed results; for today, keep it single-threaded until the logic is boringly correct.

## Putting It Together in the Repo

The full CMake setup and the `GstMeta` registration live in our companion repo **[build-with-gstreamer](https://github.com/rosemary-crypto/build-with-gstreamer)** on the `lesson-03-yolo-onnx` branch, sitting next to the `lesson-02-plugins` branch from Lesson 2 so you can diff between them.

The layout looks like this:

```text
gst/yolonnx/
  include/
    gstyolonnx.h           # the GstVideoFilter subclass
    gstdetectionsmeta.h    # the meta API
  src/
    gstyolonnx.cpp         # set_info + transform_frame_ip + inference
    gstdetectionsmeta.cpp  # register + add/get/transform helpers
    plugin.cpp             # GST_PLUGIN_DEFINE
  CMakeLists.txt           # finds ONNX Runtime, builds libgstyolonnx.so

gst-apps/lesson-03-yolo-onnx/
  src/main.cpp             # filesrc ! decodebin ! videoconvert ! yolonnx ! autovideosink
  CMakeLists.txt
```

The article you're reading is the map; the branch is the ground truth.

## Time to Experiment!

Now that you have the story end-to-end, here are some challenges I'd try if I were sitting next to you at the keyboard:

### 1. Run It With No Model
Launch the pipeline with no `model-path` set. The synthetic moving box should appear and `gst_buffer_get_detections_meta` should return non-empty results—a fast way to confirm your draw path and meta wiring without an ONNX file.

### 2. Meta Only, No Drawing
Set `draw-boxes=false attach-meta=true` and build a tiny `identity`-style element downstream that only reads `GstDetectionsMeta` and prints class ids. You'll know you understand meta when that works.

### 3. Survive a `queue`
Insert a `queue` between `yolonnx` and your downstream consumer and confirm the meta still arrives. It should—because we wired up `transform_func` from day one—but seeing the test pass is a nice confirmation that your meta is actually transform-safe.

### 4. Log Timing Like We Did for FPS
Reuse the clock tricks from the frame monitor article: wrap the `Session::Run` call inside `transform_frame_ip` and measure how long inference takes per frame at 720p versus 1080p. The numbers tell you when you actually need a GPU provider.

### 5. Debug the Usual Way
When boxes go sideways:

```bash
export GST_DEBUG=yolonnx:5
```

…and uncomment the per-detection `g_print` line in `run_detection` to see class ids, scores, and box coordinates streaming past in real time. Both tools want to help you if you ask loudly enough in debug logs.

## Conclusion

Today we connected the dots from **Phase 1** to **Phase 2**: take the standalone YOLO ONNX detector from the [previous article]({% post_url 2026-04-30-yolo-onnx-standalone-detector-cpp %}), wrap it in a **`GstVideoFilter`**-based plugin that draws boxes on the video, and attach structured results with **`GstMeta`** so the rest of the pipeline can grow without re-running the network on every hop.

The two pieces I want you to take away from this one are the **`draw-boxes` / `attach-meta` split**—because it shows the cost of inference paying for two different downstream stories at once—and the **`transform_func`** detail that keeps your meta alive across `queue`s. Skip either of those and the next phase gets harder than it needs to be.

Stay tuned for **Phase 3** where we finally start doing something with those boxes besides staring at them—object tracking. 🚀
