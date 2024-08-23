---
layout: post
title: "Getting Started with GStreamer Development – A Beginner’s Guide"
date: 2024-08-14 23:00:00
description: An introduction to GStreamer for building multimedia applications.
tags: GStreamer, development, C++, AI, video analytics
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

## Introduction

Around a year and a half ago, I found myself tasked with rebuilding an AI video analytics platform. The goal? To transition it to use the GStreamer framework. At first, it seemed like an insurmountable challenge. I spent countless nights grappling with GStreamer’s intricacies—confused by the difference between a sink and a source, haunted by the complexities of GStreamer buffers. But as I pushed through, I started to see the beauty in it. I can't believe I’m saying this, but GStreamer has made me fall deeper in love with C and C++.

In this guide, I’ll introduce you to the basics of GStreamer, sharing what I’ve learned along the way to help you get started with building your own multimedia applications. Let’s dive in!

### What is GStreamer?

GStreamer is a powerful multimedia framework used for building media processing pipelines. Whether you’re dealing with video, audio, or any other type of media, GStreamer provides the tools you need to manipulate and process data efficiently.

At its core, GStreamer is built around the concept of pipelines—sequences of elements that process data from one end to another. These elements can be sources (e.g., reading from a file), filters (e.g., converting formats), or sinks (e.g., outputting to a display or speaker). Think of it as a factory assembly line, where raw materials (your media) are processed and refined at various stations (the elements), eventually leading to a finished product (your output).

### What Am I Using GStreamer For?

For my AI video analytics platform, GStreamer acts as the backbone. The simplest version of the process is relatively straightforward: reading from a source (whether that’s video files, RTSP streams, etc.), performing object detection, tracking, and finally displaying the results. However, the full platform involves much more. It includes decoding the video streams, pre-processing the data, performing advanced object detection and tracking, video classification, adding custom metadata, sending data over MQTT, streaming via RTSP, and even detecting specific events. This complex pipeline allows for real-time analytics and decision-making, making GStreamer an essential tool in building this robust and scalable solution.

You might wonder, “Why not just use something like NVIDIA’s DeepStream SDK?” It’s a valid question, and honestly, DeepStream could simplify things. But where’s the fun in that? GStreamer is open-source, allowing me to tweak and modify the code as needed. When existing plugins don’t meet my requirements, I have the freedom to build my own. This series is about giving you that same power: developing your own custom GStreamer elements for Linux, Windows, and macOS. So, let’s continue with the basics.

## The Set-Up

### How Do You Start?

There’s no better place to understand GStreamer concepts than by exploring the [GStreamer Application Development Manual](https://gstreamer.freedesktop.org/documentation/application-development/?gi-language=c). It’s a treasure trove of information, and while I won’t repeat the concepts here, I’ll guide you through navigating everything, making your learning process smoother.

### Installing GStreamer

Before diving into examples, you’ll need to install GStreamer on your system. I suggest following the installation instructions in the [GStreamer Application Development Manual](https://gstreamer.freedesktop.org/documentation/installing/index.html?gi-language=c) for your specific environment, whether that’s Linux, Windows, or macOS.

## Creating a Simple Pipeline

To understand GStreamer, it’s crucial to get hands-on experience. Let's start by creating a simple pipeline to display a video.

### Pipeline Diagram

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="/assets/img/build_with_gstreamer/basic-pipeline-lesson-1.png" width="70%" class="img-fluid rounded z-depth-1" %}
    </div>
</div>


This diagram shows a simple pipeline with elements like `filesrc`, `decodebin`, and `autovideosink` connected together.

### Understanding the Basics

- **Elements:** These are the basic building blocks of a GStreamer pipeline. Each element has a specific role, such as reading data (source), processing it (filter), or outputting it (sink). For example:
  - `filesrc`: Reads data from a file.
  - `decodebin`: Automatically detects and decodes the data.
  - `autovideosink`: Displays the video on your screen.
- **Pads:** A pad is a point of connection in a GStreamer element, where media data flows in or out. Every element has two types of pads: source pads (where data exits the element) and sink pads (where data enters). This concept mirrors real-world scenarios—a water faucet (source) provides water, while a drain (sink) receives it.

- **Bins:** A bin is a container for elements. It groups multiple elements together, managing them as a single entity. This is particularly useful when dealing with complex pipelines. For example, `decodebin` is a bin because it automatically detects the format of incoming data, creates the necessary elements to decode it, and manages them internally. You don't have to worry about what's inside; it’s like a black box that simplifies your pipeline design.

- **Pipelines:** A pipeline is a special type of bin that manages the flow of data between elements. It controls the state of the elements and ensures data flows correctly from source to sink. Pipelines can contain multiple elements and bins, working together to process and deliver media.

### Command Line

Let’s create the pipeline shown in the diagram using GStreamer’s command-line tool:

```bash
gst-launch-1.0 filesrc location=/path/to/video.mp4 ! decodebin ! autovideosink
```

- `filesrc location=/path/to/video.mp4`: This element reads the video file.
- `decodebin`: Automatically detects and decodes the video format.
- `autovideosink`: Displays the video on your screen.
- `gst-launch-1.0`: A command-line tool provided by GStreamer to run pipelines.
- `!`: Links elements together within the pipeline.

### Writing the Equivalent GStreamer Application in C++

While using the command line is great for quick tests, writing your own applications using GStreamer’s API is where the real fun begins. Below is the part of the code to create a basic video player in C++:

```cpp
int main(int argc, char* argv[]) {
    GMainLoop* loop;
    GstBus* bus;
    guint bus_watch_id;

    // Initialize GStreamer
    gst_init(&argc, &argv);

    // Create a main loop, which will run until we explicitly quit
    loop = g_main_loop_new(NULL, FALSE);

    // Create the elements
    GstElement* pipeline = gst_pipeline_new("video-player");
    GstElement* source = gst_element_factory_make("filesrc", "file-source1");
    GstElement* decoder = gst_element_factory_make("decodebin", "decode-bin1");
    GstElement* autovideosink = gst_element_factory_make("autovideosink", "video-output1");

    // Check that all elements were created successfully
    if (!pipeline || !source || !decoder || !autovideosink) {
        g_printerr("Not all elements could be created.\n");
        return -1;
    }

    // Set the source file location
    g_object_set(G_OBJECT(source), "location", "../resources/videos/big_buck_bunny_scene.mp4", NULL);

    // Add all elements to the pipeline
    gst_bin_add_many(GST_BIN(pipeline), source, decoder, autovideosink, NULL);

    // Link the source to the decoder (decoder will dynamically link to sink)
    if (!gst_element_link(source, decoder)) {
        g_printerr("Source and decoder could not be linked.\n");
        gst_object_unref(pipeline);
        return -1;
    }

    // Connect the pad-added signal for the decoder
    g_signal_connect(decoder, "pad-added", G_CALLBACK(on_pad_added), autovideosink);

    // Set the pipeline to the playing state
    gst_element_set_state(pipeline, GST_STATE_PLAYING);

    // Start the main loop and wait until it is quit (e.g., via EOS or an error)
    g_main_loop_run(loop);

    // Clean up: set the pipeline to NULL state, remove bus watch, and unref
    gst_element_set_state(pipeline, GST_STATE_NULL);
    gst_object_unref(GST_OBJECT(pipeline));
    g_main_loop_unref(loop);

    return 0;
}

// Callback function to dynamically link pads
void on_pad_added(GstElement* src, GstPad* new_pad, GstElement* sink) {
    GstPad* sink_pad = gst_element_get_static_pad(sink, "sink");

    // Check if the sink pad is already linked
    if (gst_pad_is_linked(sink_pad)) {
        g_object_unref(sink_pad);
        return;
    }

    // Attempt to link the newly created pad with the sink pad
    GstPadLinkReturn ret = gst_pad_link(new_pad, sink_pad);
    if (GST_PAD_LINK_FAILED(ret)) {
        g_printerr("Type is '%s' but link failed.\n",
            gst_structure_get_name(gst_caps_get_structure(gst_pad_get_current_caps(new_pad), 0)));
    }

    g_object_unref(sink_pad);
}
```

- `gst_init`: Initializes GStreamer and parses any command-line arguments.
- `gst_element_factory_make`: Creates the elements used in the pipeline, such as the source, decoder, and video sink.
- `gst_bin_add_many`: Adds all elements to the pipeline.
- `gst_element_link`: Links the source element to the decoder. The decoder will dynamically link to the video sink when it determines the correct format.
- `g_signal_connect`: Connects the "pad-added" signal for the decoder, allowing the new pad to be linked to the sink dynamically.
- `gst_element_set_state`: Sets the pipeline’s state to PLAYING to start processing and displaying the video.
- `g_main_loop_run`: Runs the main loop, keeping the application alive until an error occurs or the video finishes.
- `gst_element_set_state`: Sets the pipeline's state to NULL, stopping all elements when finished.
- `gst_object_unref`: Frees up resources when the application is done.

#### Understanding the `on_pad_added` Function

In GStreamer, some elements, like `decodebin`, dynamically create pads during runtime based on the media stream they are processing. They can't link all their pads at the beginning because they don't know what streams they will encounter (e.g., a video file might contain both audio and video streams).\
The `on_pad_added` function is a callback that gets triggered when the `decodebin` element adds a new pad. This is crucial because you need to link this newly created pad to the next element in your pipeline (in this case, `autovideosink`), which handles the video output.

The complete code and detailed explanations, including installation instructions and how to run the program, are available on [GitHub](https://github.com/rosemary-crypto/build-with-gstreamer). This repository will be updated with each article in the series, making it easier for you to follow along.

### GitHub Repository and Following Along

To make the learning process even more interactive, I’ve created a [GitHub repository](https://github.com/rosemary-crypto/build-with-gstreamer) where you can find all the code examples discussed in this series. Each module in this tutorial will be represented by a separate branch in the repository, allowing you to check out the specific branch to follow along with that module. As the course progresses, the `master` branch will be updated with the latest version of the project.

- **Start with Module 1:** Check out the branch `lesson-1-getting-started` to see the code and examples for this introductory lesson.
- **Stay Updated:** As you progress through the series, switch to the corresponding branch for each module to get the most relevant code and updates.
- **Explore the Code:** Feel free to explore, modify, and experiment with the code in each branch to reinforce your learning.

### Compiling and Running

Save the code to a file named `simple_player.cpp`. Then, compile it using `gcc`:

```bash
gcc simple_player.cpp -o simple_player $(pkg-config --cflags --libs gstreamer-1.0)
```

Finally, run the compiled program:

```bash
./simple_player
```

## Conclusion

This guide has introduced you to the basics of GStreamer, from understanding its architecture to creating a simple pipeline and writing a basic video player in C++. GStreamer is a powerful tool for multimedia development, offering the flexibility to build both simple and complex media applications.

In the upcoming articles, we’ll explore GStreamer plugins in greater detail, including how to develop custom plugins for your specific needs—whether you’re aiming to build a media player, a streaming server, or an AI video analytics platform.

### Next Steps

- **Access the Code:** All the code examples, detailed explanations, and installation instructions are available in the [GitHub repository](https://github.com/rosemary-crypto/build-with-gstreamer). Make sure to check it out to follow along with each article in the series.
- **Experiment:** Try experimenting with different GStreamer elements and building more complex pipelines.
- **Explore the Documentation:** Dive into the GStreamer documentation to explore additional features and plugins.
- **Stay Tuned:** Look forward to the next article, where we’ll discuss GStreamer plugins and how they work.
