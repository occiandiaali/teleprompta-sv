<script>
  import { tick } from 'svelte';
  import Tesseract from 'tesseract.js';

  let videoEl;
  let container;
  let scriptText = "Upload a script to start...";
  let scrollSpeed = 30; // pixels per second
  let isScrolling = false;
  let offset = 0;
  let mediaRecorder;
  let recordedChunks = [];
  let mediaStream;
  let ocrProgress = 0;
  let textColour = 'purple'; // Default scroll text colour
  const textColors = ['red', 'green', 'blue', 'yellow', 'white'];
  function setTextColour(colour) {
    textColour = colour;
  }

  // Webcam setup
  // onMount(async () => {
  //   try {
  //     const stream = await navigator.mediaDevices.getUserMedia({ video: true });
  //     videoEl.srcObject = stream;
  //     videoEl.play();
  //   } catch (err) {
  //     console.error("Camera access denied", err);
  //   }
  // });

  // Scroll loop
  function startScroll() {
    isScrolling = true;
    const step = () => {
      if (isScrolling) {
        offset -= scrollSpeed / 60; // approx per frame
        requestAnimationFrame(step);
      }
    };
    requestAnimationFrame(step);
  }

  function stopScroll() {
    isScrolling = false;
  }
  async function resetScroll() {
    stopScroll();
    await tick();
    offset = 0; // animate back to top
  }

  async function handleUpload(event) {
    const file = event.target.files[0];
    if (!file) return;

    if (file.type === "text/plain") {
      const text = await file.text();
     // file.text().then(text => scriptText = text);
     scriptText = text;
     await tick();
     offset = 0; // reset when new script is loaded
    } else if (file.type.startsWith("image/")) {
      ocrProgress = 0;
      // Image file -> OCR
      const {data: {text}} = await Tesseract.recognize(file, 'eng', {
        logger: m => {
          if (m.status === 'recognizing text') {
            ocrProgress = Math.round(m.progress = 100);
          }
        }
      });
      scriptText = text;
      ocrProgress = 100;
    } else {
      alert("Unsupported file type. Please, upload .txt or image file.")
    }
  }

    async function startCamera() {
   // const stream = await navigator.mediaDevices.getUserMedia({ video: true });
    mediaStream = await navigator.mediaDevices.getUserMedia({ video: true });
    videoEl.srcObject = mediaStream;
    videoEl.play();

    // Setup recorder
    mediaRecorder = new MediaRecorder(mediaStream);
    mediaRecorder.ondataavailable = e => {
      if (e.data.size > 0) recordedChunks.push(e.data);
    };
  }
  function closeCamera() {
    if (mediaStream) {
      scriptText = null;
      const tracks = mediaStream.getTracks();
      tracks.forEach(track => track.stop());   
      videoEl.srcObject = null; 
    }
  }

  function startRecording() {
    recordedChunks = [];
    mediaRecorder.start();
  }

  function stopRecording() {
    mediaRecorder.stop();
    mediaRecorder.onstop = () => {
      const blob = new Blob(recordedChunks, { type: 'video/webm' });
      const url = URL.createObjectURL(blob);

      // Create a download link
      const a = document.createElement('a');
      a.href = url;
      a.download = 'teleprompter-recording.webm';
      a.click();
      URL.revokeObjectURL(url);
    };
    scriptText = "Upload a script to start...";
  }
</script>

<style>
  button {
    border: 1px solid gray;
    margin: 2px;
  }
  .teleprompter {
    position: absolute;
    top: 0;
    /* left: 35%; */
    left:0;
    width: 100%;
    height: 100%;
    /* background-color: red; */
    border: 1px solid orangered;
    pointer-events: none;
    display: flex;
    justify-content: center;
    align-items: flex-start;/* start at the top */
    overflow: hidden;
    /* color: rgb(231, 217, 15); */
    font-size: 2rem;
    text-align: center;
    
  }
  .scrolling {
    transform: translateY(var(--offset));
    transition: transform 0.6s ease; /* smooth animation */
  }
  .out-div {
    position:relative;
    left:20%; 
    width:60%;
    height:60vh;
    border:1px solid green;
    margin:4px
  }

  @media (max-width: 600px) {
    .out-div {
      width: 120%;
      left: 0;
    }
    .teleprompter {
      font-size: 1rem;
      min-width: 254px;
      left:0;
      height: 100%;
    }
    video {
      height: 80%;
    }
  }
</style>

<div class="out-div">
  <video bind:this={videoEl} autoplay playsinline style="width:100%; height:100%; object-fit:cover;"></video>
  <div class="teleprompter" bind:this={container}>
    <div class="scrolling" style="--offset: {offset}px;">
      {#each scriptText.split('\n') as line}
        <p style="color: {textColour};">{line}</p>
      {/each}
    </div>
  </div>
</div>


  <input type="file" accept=".txt, image/*" on:change={handleUpload} />
  {#if ocrProgress > 0 && ocrProgress < 100}
    <p>Image processing: {ocrProgress}%</p>
  {/if}
   <label for="textColourSelect">Choose a text colour:</label>
  <select id="textColourSelect" bind:value={textColour}>
    {#each textColors as colour}

    <option value={colour}>{colour}</option>
    {/each}
  </select>
  <button on:click={startScroll}>Start</button>
  <button on:click={stopScroll}>Stop</button>
  <button on:click={resetScroll}>Reset</button>
  <label>Speed: <input type="range" min="10" max="200" bind:value={scrollSpeed} />{scrollSpeed}</label>
  <!-- <video bind:this={videoEl} autoplay playsinline></video> -->
  <button on:click={startCamera}>Start Camera</button>
  <button on:click={closeCamera}>Close Camera</button>
  <button on:click={startRecording}>Record</button>
  <button on:click={stopRecording}>Stop & Download</button>


