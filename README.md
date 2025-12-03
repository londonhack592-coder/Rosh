# Rosh
Lyallpur WARZONE 
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>FPS Prototype — Lyallpur WARZONE (prototype)</title>
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <style>
    html,body { height:100%; margin:0; overflow:hidden; font-family: Arial, sans-serif; }
    #overlay {
      position: absolute; left: 10px; top: 10px; z-index: 10;
      color: #fff; text-shadow: 0 1px 3px rgba(0,0,0,0.7);
    }
    #hud {
      position: absolute; right: 10px; top: 10px; z-index: 10; color:#fff;
      text-align: right; text-shadow: 0 1px 3px rgba(0,0,0,0.7);
    }
    #crosshair {
      position: absolute; left:50%; top:50%; width:20px; height:20px; margin:-10px 0 0 -10px; z-index:9;
      pointer-events:none;
    }
    #instructions {
      position: absolute; left:50%; top:50%; transform:translate(-50%,-50%); color:white; z-index:20; text-align:center;
      background: rgba(0,0,0,0.6); padding:20px; border-radius:8px;
    }
    button { padding:8px 12px; font-size:14px; }
    canvas { display:block; }
  </style>
</head>
<body>
  <div id="overlay">
    <strong>Prototype FPS</strong><br>
    Move: WASD • Jump: Space • Shoot: Left click • Lock pointer to play
  </div>
  <div id="hud">
    <div id="score">Score: 0</div>
    <div id="health">Health: 100</div>
  </div>
  <div id="crosshair">+</div>

  <div id="instructions">
    <h2>Click to start — Pointer Lock required</h2>
    <p>Use mouse to look, WASD to move, left click to shoot.</p>
    <button id="startBtn">Start Game</button>
  </div>

  <script type="module">
  // == MODULES from UNPKG (Three.js + PointerLockControls)
  import * as THREE from 'https://unpkg.com/three@0.152.2/build/three.module.js';
  import { PointerLockControls } from 'https://unpkg.com/three@0.152.2/examples/jsm/controls/PointerLockControls.js';

  // Basic scene setup
  const scene = new THREE.Scene();
  scene.background = new THREE.Color(0x88ccee);

  const camera = new THREE.PerspectiveCamera(75, innerWidth/innerHeight, 0.1, 1000);
  camera.position.set(0, 1.6, 0); // eye height

  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(innerWidth, innerHeight);
  document.body.appendChild(renderer.domElement);

  window.addEventListener('resize', () => {
    camera.aspect = innerWidth/innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(innerWidth, innerHeight);
  });

  // Lights
  const hemi = new THREE.HemisphereLight(0xffffff, 0x444444, 1.1);
  hemi.position.set(0, 50, 0);
  scene.add(hemi);

  const dir = new THREE.DirectionalLight(0xffffff, 0.6);
  dir.position.set(5,10,7);
  scene.add(dir);

  // Ground
  const floorGeo = new THREE.PlaneGeometry(200,200);
  const floorMat = new THREE.MeshStandardMaterial({ color: 0x556644 });
  const floor = new THREE.Mesh(floorGeo, floorMat);
  floor.rotation.x = -Math.PI/2;
  scene.add(floor);

  // Simple environment blocks
  function makeBlock(x,z,w=4,d=4,h=3,color=0x8b6f5a){
    const m = new THREE.Mesh(new THREE.BoxGeometry(w,h,d), new THREE.MeshStandardMaterial({ color }));
    m.position.set(x, h/2, z);
    scene.add(m);
    return m;
  }
  makeBlock( -12, -8, 6,6,4, 0x7f5533 );
  makeBlock( 10, 12, 8,6,4, 0x3e6f4a );
  makeBlock( -20, 25, 10,3,6, 0x33447a );

  // Player controls (pointer lock)
  const controls = new PointerLockControls(camera, document.body);
  const startBtn = document.getElementById('startBtn');
  const instructions = document.getElementById('instructions');

  startBtn.addEventListener('click', () => {
    controls.lock();
  });

  controls.addEventListener('lock', () => {
    instructions.style.display = 'none';
  });
  controls.addEventListener('unlock', () => {
    instructions.style.display = '';
  });

  // Movement state
  const move = { forward:false, backward:false, left:false, right:false, jump:false };
  let velocity = new THREE.Vector3();
  let canJump = true;
  const speed = 6.0;

  document.addEventListener('keydown', (e) => {
    switch(e.code){
      case 'KeyW': move.forward = true; break;
      case 'KeyS': move.backward = true; break;
      case 'KeyA': move.left = true; break;
      case 'KeyD': move.right = true; break;
      case 'Space':
        if(canJump){ velocity.y = 6; canJump = false; }
        break;
    }
  });
  document.addEventListener('keyup', (e) => {
    switch(e.code){
      case 'KeyW': move.forward = false; break;
      case 'KeyS': move.backward = false; break;
      case 'KeyA': move.left = false; break;
      case 'KeyD': move.right = false; break;
    }
  });

  // Shooting: raycast for instant-hit bullets
  const raycaster = new THREE.Raycaster();
  const shootCooldownMs = 150;
  let lastShot = 0;
  const bullets = []; // for muzzle flash visuals

  window.addEventListener('mousedown', (e) => {
    if(!controls.isLocked) return;
    if(e.button === 0) shoot();
  });

  function shoot(){
    const now = performance.now();
    if(now - lastShot < shootCooldownMs) return;
    lastShot = now;

    // Ray from camera
    raycaster.setFromCamera(new THREE.Vector2(0,0), camera);
    const intersects = raycaster.intersectObjects(enemies.map(en => en.mesh), false);
    if(intersects.length > 0){
      const hit = intersects[0].object.userData.enemyRef;
      if(hit){
        hit.takeDamage(34); // damage per shot
      }
    }

    // muzzle flash (small sphere that fades)
    const s = new THREE.Mesh(new THREE.SphereGeometry(0.04,8,8), new THREE.MeshBasicMaterial({ color: 0xffe08a }));
    s.position.copy(camera.position).add(new THREE.Vector3(0, -0.05, -0.2).applyQuaternion(camera.quaternion));
    scene.add(s);
    bullets.push({ mesh: s, born: now });
  }

  // enemies management
  const enemies = [];
  let score = 0;
  let playerHealth = 100;
  const scoreEl = document.getElementById('score');
  const healthEl = document.getElementById('health');

  class Enemy {
    constructor(pos){
      const geo = new THREE.BoxGeometry(1,1.6,1);
      const mat = new THREE.MeshStandardMaterial({ color: 0x882222 });
      this.mesh = new THREE.Mesh(geo, mat);
      this.mesh.position.copy(pos);
      this.mesh.userData.enemyRef = this;
      this.hp = 100;
      this.speed = 1.0 + Math.random()*0.6;
      scene.add(this.mesh);
      enemies.push(this);
    }
    update(dt){
      const dir = new THREE.Vector3().subVectors(new THREE.Vector3().copy(camera.position).setY(this.mesh.position.y), this.mesh.position).normalize();
      this.mesh.position.addScaledVector(dir, this.speed * dt);
      // simple attack range
      const dist = this.mesh.position.distanceTo(camera.position);
      if(dist < 1.8){
        // damage player
        this._attackCooldown = (this._attackCooldown || 0) - dt;
        if(this._attackCooldown <= 0){
          playerHealth -= 6;
          this._attackCooldown = 0.7;
          if(playerHealth <= 0){ playerHealth = 0; onPlayerDead(); }
        }
      }
    }
    takeDamage(d){
      this.hp -= d;
      // flash
      const old = this.mesh.material.color.getHex();
      this.mesh.material.color.set(0xffffff);
      setTimeout(()=> this.mesh.material.color.set(old), 80);
      if(this.hp <= 0) this.die();
    }
    die(){
      scene.remove(this.mesh);
      const idx = enemies.indexOf(this);
      if(idx>=0) enemies.splice(idx,1);
      score += 10;
      scoreEl.textContent = "Score: " + score;
    }
  }

  // spawn wave
  function spawnEnemyWave(n=3){
    for(let i=0;i<n;i++){
      const angle = Math.random()*Math.PI*2;
      const r = 10 + Math.random()*20;
      const x = Math.cos(angle)*r;
      const z = Math.sin(angle)*r;
      new Enemy(new THREE.Vector3(x, 0.8, z));
    }
  }
  // initial spawn
  spawnEnemyWave(4);

  function onPlayerDead(){
    controls.unlock();
    instructions.style.display = '';
    instructions.innerHTML = `<h2>You died</h2><p>Score: ${score}</p><button id="restart">Restart</button>`;
    document.getElementById('restart').addEventListener('click', ()=> {
      // reset
      playerHealth = 100; score = 0;
      scoreEl.textContent = "Score: 0";
      healthEl.textContent = "Health: 100";
      // remove existing enemies
      enemies.slice().forEach(e => { scene.remove(e.mesh); enemies.splice(enemies.indexOf(e),1); });
      spawnEnemyWave(4);
      instructions.style.display = 'none';
      controls.lock();
    });
  }

  // basic head bob / camera smoothing
  let prevTime = performance.now();
  function animate(){
    requestAnimationFrame(animate);
    const time = performance.now();
    const dt = Math.min(0.05, (time - prevTime)/1000 );
    prevTime = time;

    // movement
    const direction = new THREE.Vector3();
    if(move.forward) direction.z -= 1;
    if(move.backward) direction.z += 1;
    if(move.left) direction.x -= 1;
    if(move.right) direction.x += 1;
    direction.normalize();

    // transform direction by camera yaw (controls.getObject())
    if(controls.isLocked){
      const quat = controls.getObject().quaternion;
      const moveDir = direction.clone().applyQuaternion(new THREE.Quaternion().setFromEuler(new THREE.Euler(0, getCameraYaw(), 0)));
      velocity.x += moveDir.x * speed * dt * 10;
      velocity.z += moveDir.z * speed * dt * 10;
      // gravity
      velocity.y -= 9.8 * dt;
      // apply
      controls.getObject().position.addScaledVector(new THREE.Vector3(velocity.x, 0, velocity.z), dt);
      controls.getObject().position.y += velocity.y * dt;

      // ground collision
      if(controls.getObject().position.y < 1.6){
        velocity.y = 0;
        controls.getObject().position.y = 1.6;
        canJump = true;
      }

      // damping
      velocity.x -= velocity.x * 10.0 * dt;
      velocity.z -= velocity.z * 10.0 * dt;
    }

    // enemies update
    enemies.forEach(e => e.update(dt));

    // bullets life
    for(let i = bullets.length-1; i>=0; i--){
      const b = bullets[i];
      const age = time - b.born;
      b.mesh.scale.setScalar(1 + age*0.005);
      b.mesh.material.opacity = Math.max(0, 1 - age*0.008);
      b.mesh.material.transparent = true;
      if(age > 300) { scene.remove(b.mesh); bullets.splice(i,1); }
    }

    // spawn additional enemies over time
    if(Math.random() < 0.006){ spawnEnemyWave(1); }

    // update HUD
    healthEl.textContent = "Health: " + Math.max(0, Math.round(playerHealth));
    // simple ambient camera bob
    if(controls.isLocked){
      const bob = Math.sin(time*0.005) * 0.02;
      camera.position.y = 1.6 + bob;
    }

    renderer.render(scene, camera);
  }
  animate();

  function getCameraYaw(){
    // derive yaw from controls' quaternion
    const q = controls.getObject().quaternion;
    const e = new THREE.Euler().setFromQuaternion(q, 'YXZ');
    return e.y;
  }

  // Small helper: show pointer lock when clicking canvas as well
  renderer.domElement.addEventListener('click', ()=> {
    if(!controls.isLocked) controls.lock();
  });

  // Nice: spawn more enemies on interval (keeps game active)
  setInterval(()=> {
    if(controls.isLocked && enemies.length < 10) spawnEnemyWave(1 + Math.floor(Math.random()*2));
  }, 4000);

  // prevent context menu on right click
  window.addEventListener('contextmenu', (e) => e.preventDefault());
  </script>
</body>
</html>
