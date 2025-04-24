<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, nextTick } from 'vue';
import {
  FaceDetector,
  FilesetResolver,
  type Detection, // Changed: Added 'type' keyword
  type FaceDetectorResult // Changed: Added 'type' keyword
} from '@mediapipe/tasks-vision';
const colorMap: Record<number, string> = { 1: "#000000", 2: "#FFFFFF", 3: "#FF0000", 4: "#00FF00", 5: "#0000FF",6:"rgba(0,0,0,0)" };

// --- Refs for DOM elements ---
const videoRef = ref<HTMLVideoElement | null>(null);
const liveViewRef = ref<HTMLDivElement | null>(null);
const webcamButtonRef = ref<HTMLButtonElement | null>(null);
const imageContainerRef = ref<HTMLDivElement | null>(null); // Ref for the image container
const imageRef = ref<HTMLImageElement | null>(null); // Ref for the image element

// --- MediaPipe State ---
let faceDetector: FaceDetector | undefined = undefined;
let runningMode = ref<'IMAGE' | 'VIDEO'>('IMAGE');
let webcamRunning = ref(false);
let lastVideoTime = -1;
const children = ref<HTMLElement[]>([]); // Store dynamically added elements

// --- Initialization ---
onMounted(async () => {
  await initializeFaceDetector();
  // Initialize Material Design Components (if needed, typically done globally)
  // const button = new MDCRipple(webcamButtonRef.value!); // Example
});

onBeforeUnmount(() => {
  // Cleanup resources if needed (e.g., stop webcam stream)
  if (videoRef.value?.srcObject) {
    (videoRef.value.srcObject as MediaStream).getTracks().forEach(track => track.stop());
  }
  faceDetector?.close();
});

const initializeFaceDetector = async () => {
  try {
    const vision = await FilesetResolver.forVisionTasks(
      // Use the CDN path instead of the local '/wasm' path
      "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.0/wasm" // Or the specific version you are using
    );
    faceDetector = await FaceDetector.createFromOptions(vision, {
      baseOptions: {
        modelAssetPath: `https://storage.googleapis.com/mediapipe-models/face_detector/blaze_face_short_range/float16/1/blaze_face_short_range.tflite`,
        delegate: 'GPU',
      },
      runningMode: runningMode.value,
    });
    console.log('FaceDetector initialized');
    // Make the demos section visible now that the detector is ready
    const demosSection = document.getElementById('demos');
    if (demosSection) {
        demosSection.classList.remove('invisible');
    }
  } catch (error) {
    console.error('Failed to initialize FaceDetector:', error);
    alert('Failed to initialize FaceDetector. Check console for details.');
  }
};

// --- Image Detection Logic ---
const handleClick = async (event: MouseEvent) => {
  if (!faceDetector) {
    console.log('Wait for FaceDetector to load before clicking');
    return;
  }
  if (!imageRef.value || !imageContainerRef.value) return;

  // Clear previous detections
  clearDetections(imageContainerRef.value);

  // Ensure running mode is IMAGE
  if (runningMode.value === 'VIDEO') {
    await faceDetector.setOptions({ runningMode: 'IMAGE' });
    runningMode.value = 'IMAGE';
  }

  const targetImage = event.target as HTMLImageElement;

  // Detect faces
  const detections = faceDetector.detect(targetImage).detections;
  console.log('Image detections:', detections);

  // Display detections
  displayImageDetections(detections, targetImage);
};

const displayImageDetections = (detections: Detection[], resultElement: HTMLImageElement) => {
  if (!imageContainerRef.value) return;
  const container = imageContainerRef.value;
  const ratio = resultElement.height / resultElement.naturalHeight;

  console.log(`Detected ${detections.length} faces in the image.`); // Log number of faces

  detections.forEach((detection, index) => {
    if (!detection.boundingBox) return;

    // Log bounding box coordinates for each face
    console.log(`Image Face ${index + 1} Bounding Box:`, {
        originX: detection.boundingBox.originX,
        originY: detection.boundingBox.originY,
        width: detection.boundingBox.width,
        height: detection.boundingBox.height,
    });
  });
};

// --- Webcam Detection Logic ---
const enableCam = async () => {
  if (!faceDetector) {
    alert('Face Detector is still loading. Please try again.');
    return;
  }
  if (!videoRef.value || !webcamButtonRef.value) return;

  webcamRunning.value = true;
  webcamButtonRef.value.classList.add('removed'); // Hide button visually

  const constraints = { video: true };

  try {
    const stream = await navigator.mediaDevices.getUserMedia(constraints);
    videoRef.value.srcObject = stream;
    // The 'loadeddata' event listener is added in the template
  } catch (err) {
    console.error('getUserMedia error:', err);
    webcamRunning.value = false;
    webcamButtonRef.value.classList.remove('removed');
    alert('Could not access webcam.');
  }
};

const predictWebcam = async () => {
  if (!faceDetector || !videoRef.value || !liveViewRef.value) return;

  // Ensure running mode is VIDEO
  if (runningMode.value === 'IMAGE') {
    await faceDetector.setOptions({ runningMode: 'VIDEO' });
    runningMode.value = 'VIDEO';
  }

  const video = videoRef.value;
  if (video.readyState < 2) { // Check if video is ready
      console.log("Video not ready yet");
      requestAnimationFrame(predictWebcam); // Try again on next frame
      return;
  }


  let startTimeMs = performance.now();
  if (video.currentTime !== lastVideoTime) {
    lastVideoTime = video.currentTime;

    // Use await for async detection if available, otherwise handle promise
     try {
        const result: FaceDetectorResult = faceDetector.detectForVideo(video, startTimeMs);
        displayVideoDetections(result.detections);
    } catch (error) {
        console.error("Error during video detection:", error);
        // Handle potential errors during detection if necessary
    }
  }

  // Keep predicting if webcam is running
  if (webcamRunning.value) {
    requestAnimationFrame(predictWebcam);
  }
};

const displayVideoDetections = (detections: Detection[]) => {
  if (!liveViewRef.value || !videoRef.value) return;
  const liveView = liveViewRef.value;
  const video = videoRef.value;

  // Clear previous detections from live view
  clearDetections(liveView);

  // Log number of faces detected in the current frame
  if (detections.length > 0) {
      console.log(`Detected ${detections.length} faces in the video frame.`);
  }

  detections.forEach((detection, index) => {
     if (!detection.boundingBox) return;

     // Log bounding box coordinates for each face
     console.log(`Video Face ${index + 1} Bounding Box:`, {
        originX: detection.boundingBox.originX,
        originY: detection.boundingBox.originY,
        width: detection.boundingBox.width,
        height: detection.boundingBox.height,
        // You can also log keypoints if needed:
        // keypoints: detection.keypoints
     });
     const p = document.createElement('p');
     const confidenceScore = detection.categories && detection.categories.length > 0
       ? Math.round(detection.categories[0].score * 100)
       : 'N/A';
     // Adjust positioning based on video dimensions and mirroring if needed
     const displayWidth = video.offsetWidth;
     const displayHeight = video.offsetHeight; // Use offsetHeight for displayed size
     const scaleX = displayWidth / video.videoWidth; // Calculate scale if needed, or assume 1 if display matches video resolution
     const scaleY = displayHeight / video.videoHeight;

     const mirroredOriginX = video.videoWidth - detection.boundingBox.originX - detection.boundingBox.width;
     p.style.position = 'absolute';
     p.style.left = `${mirroredOriginX * scaleX}px`; // Use mirrored X
     p.style.top = `${detection.boundingBox.originY * scaleY}px`; // Adjust Y position
     p.style.width = `${detection.boundingBox.width}px`;
     p.style.height = `${(detection.boundingBox.height) * scaleY}px`;
     p.style.color = 'white'; // Make text visible on video
     p.style.backgroundColor = 'rgba(0, 0, 0, 0.5)'; // Add background for readability
     p.style.padding = '2px';
     p.style.fontSize = '12px';
     liveView.appendChild(p);
     children.value.push(p); // Track element for cleanup
  });
};

// --- Utility Functions ---
const clearDetections = (container: HTMLElement) => {
  // Remove dynamically added children
  children.value.forEach(child => {
    if (container.contains(child)) {
        container.removeChild(child);
    }
  });
  children.value = []; // Clear the tracking array
};

// Helper to check if getUserMedia is supported
const hasGetUserMedia = () => !!navigator.mediaDevices?.getUserMedia;
</script>

<template>
  <section id="demos" class="invisible"> <!-- Start invisible until initialized -->
    <div ref="liveViewRef" id="liveView" class="videoView" style="position: relative;">
      <button ref="webcamButtonRef" id="webcamButton" class="mdc-button mdc-button--raised" @click="enableCam" :disabled="!hasGetUserMedia()">
        <span class="mdc-button__ripple"></span>
        <span class="mdc-button__label">START</span>
      </button>
    <video ref="videoRef" id="webcam" autoplay playsinline @loadeddata="predictWebcam" style="transform: scaleX(-1);"></video>
      <!-- Detection results will be appended here -->
    </div>
  </section>
</template>

<style scoped>
.invisible {
  opacity: 0.2;
}

.removed {
  display: none !important; /* Use important to override MDC styles if necessary */
}

.detectOnClick,
.videoView {
  position: relative;
  margin-top: 1em;
  border: 1px solid #cacaca;
  padding: 10px;
  background-color: #fff;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.12), 0 1px 2px rgba(0, 0, 0, 0.24);
}

.detectOnClick img {
  display: block;
  max-width: 100%;
  height: auto;
}

.videoView {
  min-height: 200px; /* Ensure the container has some height */
  display: flex; /* Use flexbox for centering */
  justify-content: center; /* Center horizontally */
  align-items: center; /* Center vertically */
  flex-direction: column; /* Stack elements vertically */
}

#webcam {
  display: block; /* Make video a block element */
  width: 100%; /* Make video responsive */
  max-width: 640px; /* Optional: Set a max width */
  height: auto; /* Maintain aspect ratio */
  margin-top: 10px; /* Add some space above the video */
}

#webcamButton {
  position: absolute; /* Position button absolutely within the container */
  top: 50%; /* Center vertically */
  left: 50%; /* Center horizontally */
  transform: translate(-50%, -50%); /* Fine-tune centering */
  z-index: 10; /* Ensure button is above the video initially */
}
/* Material Components Web Button Styles (ensure MDC CSS is loaded) */
.mdc-button {
  /* Basic styling if MDC CSS isn't fully applied */
  padding: 8px 16px;
  border: none;
  background-color: #6200ee; /* Example color */
  color: white;
  cursor: pointer;
  border-radius: 4px;
  text-transform: uppercase;
  font-weight: 500;
  box-shadow: 0 2px 2px 0 rgba(0,0,0,0.14), 0 3px 1px -2px rgba(0,0,0,0.12), 0 1px 5px 0 rgba(0,0,0,0.2);
  transition: box-shadow 0.2s ease;
}

.mdc-button:hover {
  box-shadow: 0 3px 3px 0 rgba(0,0,0,0.14), 0 1px 7px 0 rgba(0,0,0,0.12), 0 3px 1px -1px rgba(0,0,0,0.2);
}

.mdc-button:disabled {
    background-color: rgba(0, 0, 0, 0.12);
    color: rgba(0, 0, 0, 0.37);
    cursor: default;
    box-shadow: none;
}
</style>