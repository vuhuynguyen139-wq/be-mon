<!DOCTYPE html>
<html lang="vi">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>NVH 3D Camera Studio</title>

<style>
* {
    box-sizing: border-box;
}

body {
    margin: 0;
    background:
        radial-gradient(circle at 50% 20%, #243b55, #05070b 65%);
    color: white;
    font-family: Arial, sans-serif;
    overflow-x: hidden;
}

header {
    height: 70px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0 25px;
    background: rgba(0,0,0,.45);
    backdrop-filter: blur(15px);
    border-bottom: 1px solid rgba(255,255,255,.1);
}

.logo {
    font-size: 22px;
    font-weight: bold;
}

.status {
    color: #00ffae;
    font-size: 14px;
}

.main {
    width: 95%;
    max-width: 1400px;
    margin: 25px auto;
}

.camera-box {
    position: relative;
    width: 100%;
    aspect-ratio: 16 / 9;
    background: #111;
    border-radius: 25px;
    overflow: hidden;
    border: 1px solid rgba(255,255,255,.15);
    box-shadow: 0 30px 80px rgba(0,0,0,.5);
}

video,
#photoCanvas {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
}

video {
    transform: scaleX(-1);
}

#photoCanvas {
    display: none;
}

/* khung AR */
.ar-object {
    position: absolute;
    left: 50%;
    top: 50%;
    width: 180px;
    height: 180px;
    transform:
        translate(-50%, -50%)
        rotateX(15deg)
        rotateY(20deg);

    transform-style: preserve-3d;
    pointer-events: none;

    transition:
        width .12s,
        height .12s;
}

.cube {
    width: 100%;
    height: 100%;
    position: relative;
    transform-style: preserve-3d;
    animation: floating 3s ease-in-out infinite;
}

.face {
    position: absolute;
    width: 100%;
    height: 100%;
    border: 2px solid #00ffd0;
    background: rgba(0,255,208,.08);
    box-shadow:
        0 0 25px rgba(0,255,208,.5),
        inset 0 0 25px rgba(0,255,208,.1);
}

.front  { transform: translateZ(90px); }
.back   { transform: rotateY(180deg) translateZ(90px); }
.right  { transform: rotateY(90deg) translateZ(90px); }
.left   { transform: rotateY(-90deg) translateZ(90px); }
.top    { transform: rotateX(90deg) translateZ(90px); }
.bottom { transform: rotateX(-90deg) translateZ(90px); }

@keyframes floating {
    0%,100% {
        transform: translateY(0);
    }

    50% {
        transform: translateY(-10px);
    }
}

/* điểm tay */
.hand-point {
    position: absolute;
    width: 18px;
    height: 18px;
    border-radius: 50%;
    background: #00ffd0;
    box-shadow: 0 0 20px #00ffd0;
    display: none;
    pointer-events: none;
}

/* giao diện */
.controls {
    margin-top: 20px;
    display: flex;
    gap: 12px;
    flex-wrap: wrap;
}

button {
    border: 0;
    padding: 13px 20px;
    border-radius: 12px;
    background: rgba(255,255,255,.1);
    color: white;
    cursor: pointer;
    font-weight: bold;
    border: 1px solid rgba(255,255,255,.15);
}

button:hover {
    background: rgba(255,255,255,.2);
}

.capture {
    background: #00c896;
    color: #001b15;
}

.gallery {
    margin-top: 25px;
    display: grid;
    grid-template-columns: repeat(auto-fill,minmax(180px,1fr));
    gap: 15px;
}

.gallery img {
    width: 100%;
    border-radius: 15px;
    border: 1px solid rgba(255,255,255,.15);
}

.help {
    margin-top: 20px;
    padding: 20px;
    border-radius: 15px;
    background: rgba(255,255,255,.06);
    line-height: 1.7;
}

#gesture {
    color: #00ffd0;
    font-weight: bold;
}
</style>
</head>

<body>

<header>
    <div class="logo">NVH 3D CAMERA</div>
    <div class="status" id="status">● CAMERA OFF</div>
</header>

<div class="main">

    <div class="camera-box">

        <video id="video" autoplay playsinline></video>

        <canvas id="photoCanvas"></canvas>

        <!-- vật thể 3D -->
        <div class="ar-object" id="arObject">

            <div class="cube">

                <div class="face front"></div>
                <div class="face back"></div>
                <div class="face right"></div>
                <div class="face left"></div>
                <div class="face top"></div>
                <div class="face bottom"></div>

            </div>

        </div>

        <!-- điểm ngón tay -->
        <div class="hand-point" id="handPoint"></div>

    </div>

    <div class="controls">

        <button onclick="startCamera()">
            📷 Bật camera
        </button>

        <button class="capture" onclick="takePhoto()">
            📸 Chụp ảnh
        </button>

        <button onclick="resetObject()">
            🔄 Reset mẫu 3D
        </button>

    </div>

    <div class="help">

        <b>Điều khiển bằng tay</b><br>

        ☝️ <b>Ngón trỏ:</b>
        kéo mẫu 3D<br>

        ✌️ <b>Hai ngón:</b>
        chuẩn bị thao tác<br>

        🖐️ <b>Bàn tay mở:</b>
        giữ mẫu<br>

        👍 <b>Ngón cái:</b>
        có thể dùng làm lệnh chụp ở bản nâng cấp<br>

        <br>

        Gesture:
        <span id="gesture">Chưa nhận diện</span>

    </div>

    <h2>Ảnh đã chụp</h2>

    <div class="gallery" id="gallery"></div>

</div>

<script>

const video = document.getElementById("video");
const canvas = document.getElementById("photoCanvas");
const ctx = canvas.getContext("2d");

const arObject = document.getElementById("arObject");
const handPoint = document.getElementById("handPoint");

const statusText = document.getElementById("status");
const gestureText = document.getElementById("gesture");

let cameraStarted = false;

let objectX = 50;
let objectY = 50;
let objectScale = 1;

let dragging = false;


/* =========================
   CAMERA
========================= */

async function startCamera() {

    try {

        const stream = await navigator.mediaDevices.getUserMedia({
            video: {
                width: { ideal: 1280 },
                height: { ideal: 720 },
                facingMode: "user"
            },
            audio: false
        });

        video.srcObject = stream;

        cameraStarted = true;

        statusText.textContent = "● CAMERA ON";

    } catch(error) {

        alert(
            "Không mở được camera.\n\n" +
            "Hãy cho phép trình duyệt sử dụng webcam."
        );

        console.error(error);

    }

}


/* =========================
   CHỤP ẢNH
========================= */

function takePhoto() {

    if (!cameraStarted) {

        alert("Hãy bật camera trước.");

        return;
    }

    canvas.width = video.videoWidth;
    canvas.height = video.videoHeight;

    /*
       Do camera đang mirror nên
       ta lật canvas lại.
    */

    ctx.save();

    ctx.translate(canvas.width, 0);
    ctx.scale(-1, 1);

    ctx.drawImage(
        video,
        0,
        0,
        canvas.width,
        canvas.height
    );

    ctx.restore();

    const image = canvas.toDataURL("image/png");

    addPhoto(image);

}


/* =========================
   THÊM ẢNH VÀO GALLERY
========================= */

function addPhoto(image) {

    const img = document.createElement("img");

    img.src = image;

    img.title = "Ảnh NVH";

    document.getElementById("gallery")
        .prepend(img);

}


/* =========================
   RESET 3D
========================= */

function resetObject() {

    objectX = 50;
    objectY = 50;
    objectScale = 1;

    updateObject();

}


/* =========================
   UPDATE 3D
========================= */

function updateObject() {

    arObject.style.left = objectX + "%";
    arObject.style.top = objectY + "%";

    arObject.style.transform =
        `translate(-50%, -50%)
         rotateX(15deg)
         rotateY(20deg)
         scale(${objectScale})`;

}


/* =========================
   TEST KÉO BẰNG CHUỘT
   trước khi dùng tay
========================= */

const cameraBox =
    document.querySelector(".camera-box");

cameraBox.addEventListener("pointerdown", e => {

    dragging = true;

});

cameraBox.addEventListener("pointerup", e => {

    dragging = false;

});

cameraBox.addEventListener("pointermove", e => {

    if (!dragging) return;

    const rect =
        cameraBox.getBoundingClientRect();

    objectX =
        ((e.clientX - rect.left) /
        rect.width) * 100;

    objectY =
        ((e.clientY - rect.top) /
        rect.height) * 100;

    updateObject();

});


/* =========================
   PHÍM SPACE = CHỤP
========================= */

document.addEventListener("keydown", e => {

    if (e.code === "Space") {

        e.preventDefault();

        takePhoto();

    }

});


/* =========================
   DEMO GESTURE
========================= */

function simulateGesture(name) {

    gestureText.textContent = name;

}


/*
   Sau này phần này sẽ kết nối
   MediaPipe Hand Landmarker.

   Ví dụ:

   Pointing_Up
       -> di chuyển mẫu

   Open_Palm
       -> giữ mẫu

   Victory
       -> zoom

   Thumb_Up
       -> chụp ảnh tự động
*/

updateObject();

</script>

</body>
</html>
