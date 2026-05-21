---
title: "How I vibe-coded a 3D synthetic data extension for Edge Impulse"
date: 2026-05-21
author: Jenny Speelman
---

Once you've worked with developer tools long enough, you build up a mental list of ideas: *"someone should really build that, it would make my life/project a lot easier."* After five years at Edge Impulse, one O'Reilly book ([*AI at the Edge*](https://www.oreilly.com/library/view/ai-at-the/9781098120191/)), an Electrical Engineering degree from UT Austin, a lot of customer calls and developer conferences, my mental list is long. AI coding agents have become my ultimate paintbrush for generating real applications from that mental list, and the first project I've vibe-coded picks up on a thread I started working on for NVIDIA GTC in 2024.

## The Omniverse chapter

For **NVIDIA GTC 2024**, my colleagues Hannah Moshtaghi, Fernando Jiménez Moreno, and I built an [Edge Impulse extension for NVIDIA Omniverse](https://github.com/edgeimpulse/edge-impulse-omniverse-ext): a path-traced synthetic data generation pipeline that rendered photoreal images of USD scenes and pushed them, bounding boxes and all, straight into an Edge Impulse project. We wrote about it in two posts: [How to use USD assets for any edge AI use case with NVIDIA Omniverse](https://www.edgeimpulse.com/blog/how-to-use-usd-assets-for-any-edge-ai-use-case-with-nvidia-omniverse/) and [Creating a robust synthetic dataset using NVIDIA Omniverse — tips & tricks](https://www.edgeimpulse.com/blog/creating-a-robust-synthetic-dataset-using-nvidia-omniverse-tips-tricks/).

It is a fantastic tool, but Omniverse has some hard prerequisites: a hefty NVIDIA GPU or an NVIDIA cloud subscription, plus the Omniverse runtime and a non-trivial setup. Edge Impulse is the opposite, entirely cloud-based, accessible from any browser. So in practice, the workflow is always split into two: render and generate synthetic data on a beefy workstation, then context-switch to the Edge Impulse Studio in a browser tab to train an edge AI model. I kept coming back to the same question. What if you could do all of this in one browser tab?

## Synthetic Data Studio, vibe-coded

[**Synthetic Data Studio**](https://synthetic.jennyspeelman.dev/) is my answer to that question: a browser-based 3D tool for generating synthetic training data, with the Edge Impulse Ingestion API wired directly into the capture button. No Unity, no Blender, no Python, no Omniverse. Just Three.js, Rapier, and MuJoCo running on whatever GPU you already have, including integrated laptop graphics. Open the page, pick a mode, paste your Edge Impulse project API key, and start uploading samples.

![Synthetic Data Studio — Object detection mode](https://raw.githubusercontent.com/yennster/synthetic-data-studio/main/docs/screenshots/screenshot-detection.png)

*Object detection mode. Spawn labelled meshes (or import USDZ assets), point the virtual camera, and capture. Bounding boxes are automatically captured from the object's world-space, so no manual labelling is required.*

![Synthetic Data Studio — Motion mode](https://raw.githubusercontent.com/yennster/synthetic-data-studio/main/docs/screenshots/screenshot-motion.png)

*Motion mode. Pinch to grab the cube with your webcam; release, and physics takes over. The 6-channel IMU trace is recorded in the body frame, the way a real accelerometer and gyro would see it.*

I had never used Three.js before this project. I had never integrated Rapier. I had never written a USDZ importer. None of that mattered. I built the first version in a few days with [Claude Code](https://claude.com/product/claude-code) as my pair programmer. I'd describe what I wanted, write the first draft, run it, tell it what was wrong, and we'd iterate. The same workflow for every new feature: robotics mode, in-browser inference, the metadata pipeline, and USDZ import.

Two pieces of Edge Impulse infrastructure made this work cleanly:

- The [**Ingestion API**](https://docs.edgeimpulse.com/apis/ingestion). HTTP, predictable, browser-callable, with `x-label` and `x-metadata` headers that tag every sample with mode, environment, asset provenance, and split bucket.
- The [**Edge Impulse Studio API**](https://docs.edgeimpulse.com/apis/studio) and [**WebAssembly deployment**](https://docs.edgeimpulse.com/hardware/deployments/run-webassembly-browser). The Studio can deploy your trained model as a WASM bundle, unpack it in the browser, and run live inference on the virtual camera at around 5 Hz. The whole loop (generate, upload, train, deploy, inference) never leaves the browser tab.

![Edge Impulse auth card](https://raw.githubusercontent.com/yennster/synthetic-data-studio/main/docs/screenshots/card-detection-edge-impulse-auth.png)

*One project API key wires up upload, retraining triggers, and inference.*

Both APIs are well-documented and browser-callable, which makes them straightforward to wire up with an AI coding agent.

## A couple of conversations, and the MuJoCo refactor

The first version of the Synthetic Data Studio was useful but narrow: motion capture and basic object detection. I shared it on Slack and got two specific suggestions back from colleagues:

One pointed out that my rendered object-detection images looked too video-game-y to train a real model on, and suggested adding a realism filter with the kind of imperfections a real camera adds: chromatic aberration, supersampling, or JPEG round-trip. The other suggested using [MuJoCo](https://mujoco.org/) for the robotics work I had been sketching out: a differential-drive rover with chassis IMU and a lidar/ToF range ring, an [Arduino TinkerKit Braccio arm](https://store.arduino.cc/products/tinkerkit-braccio-robot) doing pick-and-place, with trajectory labels and contact-aware pickup outcomes.

The realism filter was a few days of work with Claude Code at the wheel: a Three.js Photo FX layer with per-effect sliders and a randomize toggle so the effects vary across batches. The MuJoCo suggestion turned out to reshape the whole project. My initial physics layer was [Rapier](https://rapier.rs/), which is great for rigid bodies and quick collisions, but it doesn't model articulated joints, motors, or contact-rich grasping the way you need to simulate a Braccio arm actually closing its gripper around a cube. MuJoCo does.

I pulled up the MuJoCo docs, the MJCF model format reference, and a handful of sensor-model examples, and handed them all to Claude Code with a specific ask: refactor the physics layer to use MuJoCo (WebAssembly build), keep the existing scene API stable, port the rover and arm to MJCF, wire up body-frame IMU sensors, and make sure pickup events actually record whether the gripper lifted the object.

A few days and [several commits](https://github.com/yennster/synthetic-data-studio/commits/main) later, MuJoCo was carrying the rover and arm. The arm had proper joint dynamics and contact-aware pickup outcomes; the rover had obstacle-driven collisions with `cruise` / `collision` / `stuck` labels. Once that landed, it made sense to extend MuJoCo further: motion mode now uses it for the body-frame Motion mode IMU traces too. What started as a robotics-only swap ended up reshaping the physics across the whole project.

![Robotics mode — Braccio arm simulated in MuJoCo](https://raw.githubusercontent.com/yennster/synthetic-data-studio/main/docs/screenshots/screenshot-robotics-arm.png)

*Braccio arm pick-and-place, simulated in MuJoCo. End-effector IMU records body-frame acceleration and gyro along each trajectory; the metadata captures whether the gripper actually lifted the object.*

AI coding agents are at their best when you give them the same context you'd give a human developer: the docs, the constraints, and what to preserve. Hand it "make it better," and you get vague output. Hand it: "Here is MuJoCo, here is our existing scene API, here is the IMU shape we need to preserve," and it ships a real refactor. The judgment about *what* to build still comes from talking to the real people who will actually use the application.

## Teaching the agent to speak Edge Impulse

Along the way, I packaged everything I had learned about the Edge Impulse APIs into an [**edge-impulse skill**](https://docs.edgeimpulse.com/tutorials/topics/ai-agents/create-edge-impulse-skill), a knowledge module Claude Code (and any compatible coding agent) can load to skip the "where do I find the endpoint URL" phase entirely. It bundles the auth pattern, the Ingestion vs. Studio API split, the `x-label` and `x-metadata` semantics, and the `info.labels` format. With the skill loaded, "upload these samples to my EI project" becomes a one-line ask instead of a research project for the AI agent.

The Edge Impulse documentation has a step-by-step guide for building one of these skills for your own AI agent:

> 📚 [**Create an Edge Impulse skill for AI coding agents**](https://docs.edgeimpulse.com/tutorials/topics/ai-agents/create-edge-impulse-skill)

It covers the skill file format, how to scope the knowledge so the agent triggers it at the right moment, and how to verify the agent is actually using it. The broader [AI agents section in the EI docs](https://docs.edgeimpulse.com/tutorials/topics/ai-agents) covers Claude Code, Gemini CLI, GitHub Copilot, and the others, with setup walkthroughs, prompt inspiration, and end-to-end builds.

![In-browser inference on the EI WebAssembly deployment](https://raw.githubusercontent.com/yennster/synthetic-data-studio/main/docs/screenshots/card-detection-inference-edge-impulse-model.png)

*Once your model trains, the Synthetic Data Studio downloads the WebAssembly deployment and runs inference live on the virtual camera. You can see within seconds whether your synthetic dataset performs well, all within your web browser.*

## What's coming next: bring your own web app into Edge Impulse Studio

Synthetic Data Studio is one example of the kind of tool people are building around Edge Impulse, and the natural next step is making it possible to embed it directly inside the Edge Impulse Studio.

We're extending Edge Impulse Studio so that **you can bring your own web app directly into your Edge Impulse project, embedded via an iframe**. You point your Edge Impulse project at your web app's URL (your own app, one a colleague built, or Synthetic Data Studio itself), and the Studio loads it inside the project page with your API key passed in as a URL parameter. Your app reads the key, talks to the Ingestion API, and uploads samples straight into your current Edge Impulse project. No copy-paste keys, no separate tabs, no second login.

The same pattern works for all kinds of project-scoped tools: a custom data-collection app, a domain-specific labelling UI, a hardware calibration flow. Anything you can ship to a URL can run inside the Studio with auth handled for you.

Synthetic Data Studio is the proof of concept. It already accepts a project API key via URL parameter and runs cleanly inside an iframe. Drop its URL into an EI project, and the entire 3D capture pipeline shows up next to your data view. The same pattern works for anything you can ship to a URL.

Stay tuned for more updates about Edge Impulse extensibility!

## Try it

> 🌐 **Live demo:** [synthetic.jennyspeelman.dev](https://synthetic.jennyspeelman.dev/)
> 📦 **Source:** [github.com/yennster/synthetic-data-studio](https://github.com/yennster/synthetic-data-studio)

If you do not have an Edge Impulse account, [sign up for free](https://studio.edgeimpulse.com/signup) today. And if you ship an Edge Impulse extension or app built with an AI agent, come tell us about it on the [Edge Impulse forum](https://forum.edgeimpulse.com/)!
