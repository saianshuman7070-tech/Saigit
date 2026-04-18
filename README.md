<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<title>Gesture Controlled 3D Particle System</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no"/>

<style>
html, body {
  margin: 0;
  padding: 0;
  overflow: hidden;
  background: #000;
}
canvas {
  display: block;
}
</style>

<!-- THREE.JS -->
<script src="https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.min.js"></script>

<!-- MEDIAPIPE -->
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>
</head>

<body>

<video id="video" autoplay playsinline muted style="display:none"></video>
<canvas id="three-canvas"></canvas>

<script>
/* =========================
   BASIC SETUP
========================= */
const isMobile = /Android|iPhone|iPad/i.test(navigator.userAgent);
const COUNT = isMobile ? 12000 : 25000;

const scene = new THREE.Scene();

const camera = new THREE.PerspectiveCamera(
  70,
  window.innerWidth / window.innerHeight,
  0.1,
  1000
);
camera.position.z = 6;

const renderer = new THREE.WebGLRenderer({
  canvas: document.getElementById("three-canvas"),
  alpha: true,
  antialias: true
});
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

/* =========================
   PARTICLES
========================= */
const geometry = new THREE.BufferGeometry();
const positions = new Float32Array(COUNT * 3);
const colors = new Float32Array(COUNT * 3);
const velocities = new Float32Array(COUNT * 3);

for (let i = 0; i < COUNT; i++) {
  positions.set([
    (Math.random() - 0.5) * 5,
    (Math.random() - 0.5) * 5,
    (Math.random() - 0.5) * 5
  ], i * 3);

  colors.set([
    Math.random(),
    Math.random(),
    Math.random()
  ], i * 3);
}

geometry.setAttribute("position", new THREE.BufferAttribute(positions, 3));
geometry.setAttribute("color", new THREE.BufferAttribute(colors, 3));

const material = new THREE.PointsMaterial({
  size: 0.05,
  vertexColors: true,
  transparent: true,
  blending: THREE.AdditiveBlending
});

const particles = new THREE.Points(geometry, material);
scene.add(particles);

/* =========================
   SHAPES
========================= */
function heart(i,s){
  const t=i/COUNT*Math.PI*2;
  return [
    s*0.16*Math.pow(Math.sin(t),3),
    s*(0.13*Math.cos(t)-0.05*Math.cos(2*t)),
    0
  ];
}

function flower(i,s){
  const t=i/COUNT*Math.PI*2;
  const r=s*Math.sin(6*t);
  return [r*Math.cos(t),r*Math.sin(t),0];
}

function saturn(i,s){
  const t=i/COUNT*Math.PI*2;
  return [s*Math.cos(t),s*Math.sin(t),Math.sin(t*8)*0.25];
}

const shapes=[heart,flower,saturn];
let shapeIndex=0;
let lastSwitch=0;

function applyShape(fn,scale){
  const p=geometry.attributes.position.array;
  for(let i=0;i<COUNT;i++){
    const [x,y,z]=fn(i,scale);
    p[i*3]=x;
    p[i*3+1]=y;
    p[i*3+2]=z;
  }
  geometry.attributes.position.needsUpdate=true;
}

/* =========================
   MEDIAPIPE
========================= */
const hands = new Hands({
  locateFile: f => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${f}`
});

hands.setOptions({
  maxNumHands:1,
  modelComplexity:0,
  minDetectionConfidence:0.7,
  minTrackingConfidence:0.7
});

let lastX=0,lastTime=performance.now();

hands.onResults(r=>{
  if(!r.multiHandLandmarks.length) return;
  const lm=r.multiHandLandmarks[0];

  const thumb=lm[4], index=lm[8], palm=lm[0];
  const pinch=Math.hypot(thumb.x-index.x,thumb.y-index.y);
  const open=Math.hypot(lm[5].x-lm[17].x,lm[5].y-lm[17].y);

  // Shape switch (debounced)
  if(pinch<0.035 && performance.now()-lastSwitch>700){
    shapeIndex=(shapeIndex+1)%shapes.length;
    applyShape(shapes[shapeIndex],3);
    lastSwitch=performance.now();
  }

  // Scale
  particles.scale.setScalar(THREE.MathUtils.clamp(open*4,0.6,3));

  // Size
  material.size=THREE.MathUtils.clamp(0.18-pinch,0.02,0.12);

  // Fireworks
  const now=performance.now();
  const speed=Math.abs(palm.x-lastX)/(now-lastTime);
  if(speed>0.002){
    for(let i=0;i<COUNT;i++){
      velocities[i*3]=(Math.random()-0.5)*0.4;
      velocities[i*3+1]=(Math.random()-0.5)*0.4;
      velocities[i*3+2]=(Math.random()-0.5)*0.4;
    }
  }
  lastX=palm.x; lastTime=now;

  // Color flow
  const c=geometry.attributes.color.array;
  for(let i=0;i<COUNT;i++){
    c[i*3]=1-pinch;
    c[i*3+1]=open;
    c[i*3+2]=0.5+Math.random()*0.5;
  }
  geometry.attributes.color.needsUpdate=true;
});

/* =========================
   CAMERA
========================= */
const video=document.getElementById("video");
const cam=new Camera(video,{
  onFrame:async()=>hands.send({image:video}),
  width:640,
  height:480
});
cam.start();

/* =========================
   ANIMATE
========================= */
function animate(){
  requestAnimationFrame(animate);
  const p=geometry.attributes.position.array;
  for(let i=0;i<COUNT;i++){
    p[i*3]+=velocities[i*3]*0.94;
    p[i*3+1]+=velocities[i*3+1]*0.94;
    p[i*3+2]+=velocities[i*3+2]*0.94;
  }
  geometry.attributes.position.needsUpdate=true;
  particles.rotation.y+=0.001;
  renderer.render(scene,camera);
}
animate();

/* =========================
   RESIZE
========================= */
window.addEventListener("resize",()=>{
  camera.aspect=innerWidth/innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth,innerHeight);
});
</script>

</body>
</html>
