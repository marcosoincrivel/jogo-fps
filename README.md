<!DOCTYPE html>
<html lang="pt-br">
<head>
  <meta charset="UTF-8">
  <title>Jogo de Tiro FPS Complexo</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <div id="blocker">
    <div id="instructions">
      Clique para iniciar<br>
      W, A, S, D para mover<br>
      Mouse para olhar<br>
      Clique esquerdo para atirar
    </div>
  </div>
  <div id="hud">Pontos: 0</div>
  <!-- O script principal é importado como módulo -->
  <script type="module" src="js/main.js"></script>
  margin: 0;
  overflow: hidden;
  font-family: Arial, sans-serif;
}

#blocker {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.75);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 10;

#instructions {
  color: #fff;
  text-align: center;
  font-size: 24px;

#hud {
  position: absolute;
  top: 10px;
  left: 10px;
  color: #fff;
  font-size: 18px;
  z-index: 5;// js/main.js
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.152.2/build/three.module.js';
import { PointerLockControls } from 'https://cdn.jsdelivr.net/npm/three@0.152.2/examples/jsm/controls/PointerLockControls.js';
import { Enemy } from './enemy.js';
import { Bullet } from './bullet.js';

let camera, scene, renderer, controls;
let enemies = [];
let bullets = [];
let clock = new THREE.Clock();
let score = 0;
const hud = document.getElementById('hud');

init();
animate();

function init() {
  // Criação da cena e câmera
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x87ceeb);
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 1, 2000);
  camera.position.y = 10;

  // Iluminação
  const hemiLight = new THREE.HemisphereLight(0xffffff, 0x444444);
  hemiLight.position.set(0, 200, 0);
  scene.add(hemiLight);

  const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
  dirLight.position.set(-100, 200, -100);
  dirLight.castShadow = true;
  scene.add(dirLight);

  // Chão
  const floorGeometry = new THREE.PlaneGeometry(2000, 2000);
  const floorMaterial = new THREE.MeshPhongMaterial({ color: 0x999999 });
  const floor = new THREE.Mesh(floorGeometry, floorMaterial);
  floor.rotation.x = -Math.PI / 2;
  floor.receiveShadow = true;
  scene.add(floor);

  // Renderer
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.shadowMap.enabled = true;
  document.body.appendChild(renderer.domElement);

  // Controles com Pointer Lock
  controls = new PointerLockControls(camera, document.body);
  const blocker = document.getElementById('blocker');
  const instructions = document.getElementById('instructions');

  instructions.addEventListener('click', () => {
    controls.lock();
  });

  controls.addEventListener('lock', () => {
    blocker.style.display = 'none';
  });

  controls.addEventListener('unlock', () => {
    blocker.style.display = 'flex';
  });

  scene.add(controls.getObject());

  // Variáveis de movimento do jogador
  const move = { forward: false, backward: false, left: false, right: false };
  const velocity = new THREE.Vector3();
  const direction = new THREE.Vector3();

  document.addEventListener('keydown', (event) => {
    switch (event.code) {
      case 'KeyW': move.forward = true; break;
      case 'KeyA': move.left = true; break;
      case 'KeyS': move.backward = true; break;
      case 'KeyD': move.right = true; break;
    }
  });

  document.addEventListener('keyup', (event) => {
    switch (event.code) {
      case 'KeyW': move.forward = false; break;
      case 'KeyA': move.left = false; break;
      case 'KeyS': move.backward = false; break;
      case 'KeyD': move.right = false; break;
    }
  });

  // Atualiza a posição do jogador
  function updatePlayer(delta) {
    velocity.x -= velocity.x * 10.0 * delta;
    velocity.z -= velocity.z * 10.0 * delta;
    direction.z = Number(move.forward) - Number(move.backward);
    direction.x = Number(move.right) - Number(move.left);
    direction.normalize();
    if (move.forward || move.backward) velocity.z -= direction.z * 400.0 * delta;
    if (move.left || move.right) velocity.x -= direction.x * 400.0 * delta;
    controls.moveRight(-velocity.x * delta);
    controls.moveForward(-velocity.z * delta);
  }

  // Disparo com clique do mouse
  document.addEventListener('mousedown', (event) => {
    if (event.button === 0 && controls.isLocked) {
      const bullet = new Bullet(controls.getObject().position, camera.getWorldDirection(new THREE.Vector3()));
      bullets.push(bullet);
      scene.add(bullet.mesh);
    }
  });

  // Spawn de inimigos
  spawnEnemies(20);

  window.addEventListener('resize', onWindowResize, false);

  // Função de atualização do jogo
  function update() {
    const delta = clock.getDelta();
    updatePlayer(delta);

    // Atualiza as balas
    for (let i = bullets.length - 1; i >= 0; i--) {
      bullets[i].update(delta);
      // Remove a bala se estiver fora de alcance
      if (bullets[i].mesh.position.distanceTo(controls.getObject().position) > 500) {
        scene.remove(bullets[i].mesh);
        bullets.splice(i, 1);
        continue;
      }
      // Verifica colisão com inimigos
      for (let j = enemies.length - 1; j >= 0; j--) {
        if (bullets[i].mesh.position.distanceTo(enemies[j].mesh.position) < 3) {
          // Inimigo atingido
          scene.remove(enemies[j].mesh);
          enemies.splice(j, 1);
          scene.remove(bullets[i].mesh);
          bullets.splice(i, 1);
          score += 10;
          hud.textContent = 'Pontos: ' + score;
          break;
        }
      }
    }

    // Atualiza os inimigos
    enemies.forEach(enemy => {
      enemy.update(delta, controls.getObject().position);
    });
  }

  // Renderização
  function render() {
    update();
    renderer.render(scene, camera);
  }

  // Loop de animação
  function animate() {
    requestAnimationFrame(animate);
    render();
  }
  animate();
}

function spawnEnemies(count) {
  for (let i = 0; i < count; i++) {
    let pos = new THREE.Vector3(
      (Math.random() - 0.5) * 800,
      2,
      (Math.random() - 0.5) * 800
    );
    let enemy = new Enemy(pos);
    enemies.push(enemy);
    scene.add(enemy.mesh);
  }
}

function onWindowResize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.152.2/build/three.module.js';

export class Enemy {
  constructor(position) {
    this.speed = 20; // Velocidade de perseguição
    this.mesh = this.createMesh();
    this.mesh.position.copy(position);
  }

  createMesh() {
    const geometry = new THREE.BoxGeometry(4, 4, 4);
    const material = new THREE.MeshPhongMaterial({ color: 0xff0000 });
    const mesh = new THREE.Mesh(geometry, material);
    mesh.castShadow = true;
    mesh.receiveShadow = true;
    return mesh;
  }

  update(delta, playerPosition) {
    // Move o inimigo em direção ao jogador
    const direction = new THREE.Vector3().subVectors(playerPosition, this.mesh.position);
    direction.y = 0; // Ignora a diferença vertical
    if (direction.length() > 1) {
      direction.normalize();
      this.mesh.position.add(direction.multiplyScalar(this.speed * delta));
    }
  }
}
5. js/bullet.js
js
Copiar
Editar
// js/bullet.js
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.152.2/build/three.module.js';

export class Bullet {
  constructor(startPosition, direction) {
    this.speed = 600; // Velocidade da bala
    this.direction = direction.clone().normalize();
    this.mesh = this.createMesh();
    this.mesh.position.copy(startPosition);
  }

  createMesh() {
    const geometry = new THREE.SphereGeometry(0.5, 8, 8);
    const material = new THREE.MeshBasicMaterial({ color: 0xffff00 });
    return new THREE.Mesh(geometry, material);
  }

  update(delta) {
    const moveDistance = this.speed * delta;
    this.mesh.position.add(this.direction.clone().multiplyScalar(moveDistance));
  }
}
