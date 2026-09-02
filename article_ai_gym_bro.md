# How I Built an In-Browser AI Workout Tracker Using React and MediaPipe 🏋️‍♂️🤖

> **TL;DR:** Learn how to build a real-time computer vision fitness coach that runs entirely on the client side using **React**, **Vite**, and **Google MediaPipe Pose**. Zero backend server latency, 100% user privacy, and 60 FPS performance in the browser.
>
> ⭐️ **Source Code:** [github.com/umersmx/ai-gym-bro-web](https://github.com/umersmx/ai-gym-bro-web)  
> 👤 **Author:** [Muhammad Umer Farooq (@umersmx)](https://github.com/umersmx)

---

## 🌟 Introduction & Motivation

Have you ever wondered if you're hitting proper depth on your squats or locking out correctly on your bicep curls? While smart wearables track your heart rate, they cannot assess your **physical posture and biomechanics**.

Traditionally, running computer vision models required heavy Python backends and expensive GPU cloud servers. But with modern WebAssembly (WASM) and **Google MediaPipe**, we can now execute high-precision 33-point body landmark detection **directly in the user's browser**.

In this article, I'll walk you through how I engineered **"The AI Gym Bro"** — an open-source real-time workout assistant that tracks exercises, counts repetitions, and provides live biomechanical feedback.

---

## 🏗️ System Architecture

The entire tracking pipeline runs in a continuous client-side loop:

```
┌─────────────────┐       ┌────────────────────────┐       ┌──────────────────────┐
│  Webcam Stream  │ ────> │  MediaPipe Pose Model  │ ────> │  33 Body 3D Keypoints │
│   (<video>)     │       │   (WASM / WebGL)       │       │    (x, y, z coords)  │
└─────────────────┘       └────────────────────────┘       └──────────────────────┘
                                                                       │
                                                                       ▼
┌─────────────────┐       ┌────────────────────────┐       ┌──────────────────────┐
│  React HUD /    │ <──── │  Rep State Machine     │ <──── │  Joint Angle Math    │
│  Form Feedback  │       │  (Threshold Hysteresis)│       │  (Trigonometry)      │
└─────────────────┘       └────────────────────────┘       └──────────────────────┘
```

---

## 1. Setting Up MediaPipe in React

First, install the official MediaPipe Vision tasks library:

```bash
npm install @mediapipe/tasks-vision
```

We initialize the `PoseLandmarker` using `WASM` files loaded from Google's CDN:

```javascript
import { PoseLandmarker, FilesetResolver } from "@mediapipe/tasks-vision";

export const initializePoseLandmarker = async () => {
  const vision = await FilesetResolver.forVisionTasks(
    "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@latest/wasm"
  );

  const poseLandmarker = await PoseLandmarker.createFromOptions(vision, {
    baseOptions: {
      modelAssetPath: `https://storage.googleapis.com/mediapipe-models/pose_landmarker/pose_landmarker_lite/float16/1/pose_landmarker_lite.task`,
      delegate: "GPU", // Hardware acceleration
    },
    runningMode: "VIDEO",
    numPoses: 1,
  });

  return poseLandmarker;
};
```

---

## 2. Calculating Joint Angles with Trigonometry

MediaPipe provides normalized coordinates `(x, y, z)` for 33 landmarks. To assess form (e.g., knee bend during a squat or elbow flex during a curl), we need to calculate the **interior angle formed by 3 connected joints** $(A \rightarrow B \rightarrow C)$.

Here is the 2D vector trigonometry formula implemented in JavaScript:

```javascript
export function calculateAngle(a, b, c) {
  // Vector BA and Vector BC
  const radians =
    Math.atan2(c.y - b.y, c.x - b.x) - Math.atan2(a.y - b.y, a.x - b.x);

  let angle = Math.abs((radians * 180.0) / Math.PI);

  if (angle > 180.0) {
    angle = 360.0 - angle;
  }

  return Math.round(angle);
}
```

* For **Bicep Curls**: $A = \text{Shoulder (11)}$, $B = \text{Elbow (13)}$, $C = \text{Wrist (15)}$
* For **Squats**: $A = \text{Hip (23)}$, $B = \text{Knee (25)}$, $C = \text{Ankle (27)}$

---

## 3. Finite State Machine for Repetition Counting

To prevent false counts caused by minor muscle twitches, we implement a **State Machine with Hysteresis**:

```javascript
// Exercise Thresholds for Bicep Curls
const STAGES = {
  UP: "up",
  DOWN: "down",
};

let currentStage = STAGES.DOWN;
let repCount = 0;

export function evaluateRepetition(elbowAngle) {
  // Contraction phase (curl up)
  if (elbowAngle < 35 && currentStage === STAGES.DOWN) {
    currentStage = STAGES.UP;
  }

  // Extension phase (curl down - rep completed)
  if (elbowAngle > 160 && currentStage === STAGES.UP) {
    currentStage = STAGES.DOWN;
    repCount += 1;
    triggerAudioFeedback("Good rep!");
  }

  return { count: repCount, stage: currentStage };
}
```

---

## 4. Performance Optimization: 60 FPS Without React Re-render Lag

A common mistake in React Computer Vision apps is calling `useState` on every single video frame (60 times a second), which creates significant render bottlenecks.

**The Solution:**
1. Keep the webcam frame processing loop inside `requestAnimationFrame`.
2. Store frame-to-frame coordinates in `useRef`.
3. Draw skeletal overlays directly on an HTML5 `<canvas>` using 2D context.
4. Only dispatch React state updates when the **Repetition Count** or **Form Warning** actually changes.

---

## 🎯 Key Takeaways & What's Next

1. **Client-Side AI is Production Ready**: You no longer need expensive GPU servers for real-time human pose tracking.
2. **Privacy by Design**: Video streams never leave the user's local machine.
3. **Sub-15ms Latency**: Delivering instant audio and visual feedback when form degrades.

---

## 🚀 Check Out the Project & Connect!

* ⭐️ **Star the GitHub Repository:** [github.com/umersmx/ai-gym-bro-web](https://github.com/umersmx/ai-gym-bro-web)
* 🌐 **My Portfolio:** [umersmx.vercel.app](https://umersmx.vercel.app/)
* 💼 **Connect on LinkedIn:** [linkedin.com/in/umersmx](https://linkedin.com/in/umersmx)
* 🐙 **Follow on GitHub:** [@umersmx](https://github.com/umersmx)

*If you found this guide helpful, drop a comment or a star on GitHub! Happy building!* 🚀
