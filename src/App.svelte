<script>
  import { onMount } from 'svelte';
  import { fade } from 'svelte/transition';
  import * as faceapi from 'face-api.js';


  // lots of code borrowed from https://github.com/justadudewhohacks/face-api.js/blob/master/examples/examples-browser/views/webcamFaceLandmarkDetection.html

  // html elements populated onMount
  var canvas;
  var videoElement;
  var debugCanvasElement;

  // state!
	var status_message = 'waiting for webcam...';
  var show_instructions = false;
  var distance = undefined;
  var face_detection_timeout_threshhold = 700; // ms
  var requested_distance = undefined;
  var lines_pointing_left = Math.random() > .5;
  var move_face_back_amount = 3;
  var move_face_forward_amount = 6;
  var error_in_face_distance_allowed = 3;
  var face_movement_request_timeout_threshhold = 300; // ms
  var key_pressed = null;

  // helpers!
  function timer(milliseconds) {
    return new Promise(function(resolve) {
      setTimeout(function() {
        resolve()
      }, 200)
    })
  }
  function radians(degrees) {
    return degrees * (Math.PI/180)
  }

  function clear_canvas(ctx) {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
  }
  function draw_lines(ctx) {
    const coarseness = 5;
    const angle = lines_pointing_left ? 45 : -45;

    clear_canvas(ctx);

    // draw a circle mask that only shows things drawn within the circle
    let radius = canvas.width * (3/5) * (1/2); // circle with is 3/5 of canvas
    ctx.beginPath();
    ctx.arc(canvas.width/2, canvas.height/2, radius, 0, Math.PI * 2, true);
    ctx.clip();

    // draw lines across entire canvas but only ones within circle will show
    ctx.lineWidth = coarseness/2;

    for (var x=-canvas.height; x<canvas.width*2+canvas.height; x+=coarseness) {
      ctx.beginPath()
      ctx.moveTo(x, 0)
      ctx.lineTo(x+canvas.height*Math.tan(radians(angle)), canvas.height);
      ctx.stroke()
    }
  }

  async function load_webcam_and_face_models() {
    try {
      videoElement.srcObject = await navigator.mediaDevices.getUserMedia({ video: {} }) // get webcam video data
      status_message = 'loading face detection...'
    } catch (e) {
      status_message = 'please enable webcam access for this site'
      return
    }
    try {
      await faceapi.nets.tinyFaceDetector.loadFromUri('face-models/') // load face detector model data
      await faceapi.nets.faceLandmark68Net.loadFromUri('face-models/') // load facial landmarks model data
    } catch (e) {
      status_message = "couldn't load face detection. does your internet work?"
      return
    }

  }


  const faceOptions = new faceapi.TinyFaceDetectorOptions();
  // 68 facial points reference: https://www.pyimagesearch.com/wp-content/uploads/2017/04/facial_landmarks_68markup-768x619.jpg
  const TOP_LEFT_EYE_INDEX = 38 // 39 in 1-based array
  const TOP_RIGHT_EYE_INDEX = 43 // 44 in 1-based array
  const TOP_MOUTH_INDEX = 51 // 52 in 1-based array

  onMount(async function() {
    await load_webcam_and_face_models()

    const ctx = canvas.getContext('2d');
    var frame;
    var last_face_detection = new Date();
    // var last_face_distance_mismatch = new Date();

    (async function loop() {
      frame = requestAnimationFrame(loop);

      if (key_pressed) {
        if ((key_pressed === 'left' && lines_pointing_left) || (key_pressed === 'right' && !lines_pointing_left)) {
          status_message = 'correct!'
          requested_distance -= move_face_back_amount;
          lines_pointing_left = Math.random() > 0.5;
        } else {
          status_message = 'oops'
          requested_distance += move_face_forward_amount;
          lines_pointing_left = Math.random() > 0.5;
        }

        key_pressed = null;
        await timer(1000);
      }

      const detection = await faceapi.detectSingleFace(videoElement, faceOptions).withFaceLandmarks()
      if (detection) {
        last_face_detection = new Date();

        const dims = faceapi.matchDimensions(debugCanvasElement, videoElement, true)
        const resizedResult = faceapi.resizeResults(detection, dims)
        faceapi.draw.drawDetections(debugCanvasElement, resizedResult)
        faceapi.draw.drawFaceLandmarks(debugCanvasElement, resizedResult)
        const left_eye = detection.landmarks.positions[TOP_LEFT_EYE_INDEX]
        const right_eye = detection.landmarks.positions[TOP_RIGHT_EYE_INDEX]
        const mid_eye = faceapi.getCenterPoint([left_eye, right_eye]) // right between the eyes
        const mouth = detection.landmarks.positions[TOP_MOUTH_INDEX]
        const face_height_in_camera = mid_eye.sub(mouth).magnitude() // measured on midline from midway between eyes to top lip

        distance = face_height_in_camera;
        if (!requested_distance) {
          requested_distance = distance;
        }

        if (distance > requested_distance + error_in_face_distance_allowed) {
          status_message = `Please move back ${+(distance - (requested_distance + error_in_face_distance_allowed)).toFixed(1)}`
        } else if (distance < requested_distance - error_in_face_distance_allowed) {
          status_message = `Please move forward ${+((requested_distance - error_in_face_distance_allowed) - distance).toFixed(1)}`
        } else {
          status_message = '';
          show_instructions = true;
        }
      } else if ((new Date() - last_face_detection) > face_detection_timeout_threshhold) {
        status_message = "Please make your face visible to the camera and keep still" 
      } else {
        distance = undefined;
      }

      if (!status_message) {
        draw_lines(ctx);
      } else {
        clear_canvas(ctx);
      }


    }());

    return function() {
      cancelAnimationFrame(frame);
    };
  });

  function handleKey(evt) {
    evt.preventDefault();
    if (evt.key == 'ArrowLeft') {
      key_pressed = 'left';
    }
    if (evt.key == 'ArrowRight') {
      key_pressed = 'right';
    }
  }

</script>

<svelte:window on:keydown={handleKey}/>

<div class="container">
  <div>
    <canvas width="500" height="500" bind:this={canvas}></canvas>

    {#if status_message !== ''}
    <status transition:fade={{ duration: 300 }}>{status_message}</status>
    {/if}
    <instructions>
      {#if show_instructions}
        <div transition:fade>
          Press <img src="left-arrow.png" width="40" height="40" alt="left arrow" /> if the lines go up and to the left or <img src="right-arrow.png" alt="right arrow" width="40" height="40" /> if the lines go up and to the right.
        </div>
      {/if}
    </instructions>
    <p style="text-align: center; color: #999;">your webcam content is never recorded</p>
  </div>

  <hr />

  <div class="debug">
    <p>debugging stuff</p>
    <div>
      <video bind:this={videoElement} autoplay muted></video>
      <canvas bind:this={debugCanvasElement} style="position: absolute;"></canvas>
    </div>
    <p>Detected face distance: {distance}</p>
    <p>Requested face distance: {requested_distance}</p>
    <p>Face detection timeout threshhold: <input type="range" bind:value={face_detection_timeout_threshhold} min="0" max="1000" step="50" /> {face_detection_timeout_threshhold}ms </p>
    <p>Move face back amount: <input type="range" min="0.5" max="20" step="0.5" bind:value={move_face_back_amount} /> {move_face_back_amount}</p>
    <p>Move face forward amount: <input type="range" min="0.5" max="20" step="0.5" bind:value={move_face_forward_amount} /> {move_face_forward_amount} (this should be about 3 times more than the backward amount)</p>
    <p>Error in face distance allowed: <input type="range" min="0" max="20" step="0.5" bind:value={error_in_face_distance_allowed} /> {error_in_face_distance_allowed}
    <p>Lines pointing to left: {lines_pointing_left}</p>
  </div>
</div>



<style>
  .container {
    width: 800px;
    margin: 0 auto;
  }
  .container div {
    position: relative;
  }
  instructions {
    text-align: center;
    width: 700px;
    margin: 10px auto 40px auto;
    min-height: 81px;
    font-size: 30px;
    display: block;
  }
  canvas {
    margin: 0 auto;
    border: 1px solid #ccc;
    display: block;
  }
  status {
    display: block;
    font-size: 45px;
    text-align: center;
    margin-left: auto;
    margin-right: auto;
    width: 480px;
    position: absolute;
    top: 175px;
    left: 162px;
    background-color: rgba(222, 213, 213, 0.77);
    padding: 20px 0;
  }

  .debug {
    position: relative;
    margin-top: 40px;
    margin-bottom: 100px;
  }
  .debug div {
    position: relative;
  }
  .debug video {
    width: 100%;    
  }
  .debug canvas {
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
  }
</style>


