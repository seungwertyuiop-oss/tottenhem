const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

const lapValue = document.getElementById('lapValue');
const speedValue = document.getElementById('speedValue');
const rankValue = document.getElementById('rankValue');

const keys = {};
const state = {
  lastTime: 0,
  player: {
    progress: 0.08,
    lap: 1,
    laneOffset: 0,
    speed: 0,
    boost: 0,
    angle: 0,
    color: '#7ef9ff',
  },
  ai: [
    { name: 'Pico', color: '#ff7b7b', progress: 0.2, lap: 1, laneOffset: -40, speed: 130 },
    { name: 'Nova', color: '#ffd166', progress: 0.42, lap: 1, laneOffset: 48, speed: 137 },
    { name: 'Bolt', color: '#c4ff6d', progress: 0.7, lap: 1, laneOffset: -18, speed: 122 },
  ],
  items: [
    { progress: 0.18, laneOffset: 0, radius: 14, active: true },
    { progress: 0.38, laneOffset: 75, radius: 14, active: true },
    { progress: 0.56, laneOffset: -68, radius: 14, active: true },
    { progress: 0.82, laneOffset: 28, radius: 14, active: true },
  ],
};

const track = {
  cx: canvas.width / 2,
  cy: canvas.height / 2,
  a: 230,
  b: 160,
  roadWidth: 110,
  innerWidth: 70,
};

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}

function wrapProgress(value) {
  return ((value % 1) + 1) % 1;
}

function sampleTrack(progress, laneOffset = 0) {
  const t = progress * Math.PI * 2;
  const x = track.cx + Math.cos(t) * track.a + Math.cos(t * 2.2) * 26;
  const y = track.cy + Math.sin(t) * track.b + Math.sin(t * 3.1) * 18;

  const dx = -Math.sin(t) * track.a - 2.2 * Math.sin(t * 2.2) * 26;
  const dy = Math.cos(t) * track.b + 3.1 * Math.cos(t * 3.1) * 18;
  const mag = Math.hypot(dx, dy) || 1;
  const tangent = { x: dx / mag, y: dy / mag };
  const normal = { x: -tangent.y, y: tangent.x };

  return {
    x: x + normal.x * laneOffset,
    y: y + normal.y * laneOffset,
    angle: Math.atan2(tangent.y, tangent.x),
  };
}

function drawBackground() {
  ctx.fillStyle = '#0f2d26';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  for (let i = 0; i < 80; i++) {
    const x = (i * 93) % canvas.width;
    const y = ((i * 57) % canvas.height) + 22;
    ctx.fillStyle = 'rgba(255,255,255,0.03)';
    ctx.fillRect(x, y, 2, 2);
  }
}

function drawTrack() {
  ctx.beginPath();
  for (let i = 0; i <= 600; i++) {
    const p = sampleTrack(i / 600, 0);
    if (i === 0) ctx.moveTo(p.x, p.y);
    else ctx.lineTo(p.x, p.y);
  }
  ctx.closePath();
  ctx.lineWidth = track.roadWidth;
  ctx.strokeStyle = '#2d3f45';
  ctx.stroke();

  ctx.beginPath();
  for (let i = 0; i <= 600; i++) {
    const p = sampleTrack(i / 600, 0);
    if (i === 0) ctx.moveTo(p.x, p.y);
    else ctx.lineTo(p.x, p.y);
  }
  ctx.closePath();
  ctx.lineWidth = track.innerWidth;
  ctx.strokeStyle = '#4a9a53';
  ctx.stroke();

  ctx.beginPath();
  for (let i = 0; i <= 600; i++) {
    const p = sampleTrack(i / 600, 0);
    if (i === 0) ctx.moveTo(p.x, p.y);
    else ctx.lineTo(p.x, p.y);
  }
  ctx.closePath();
  ctx.lineWidth = 12;
  ctx.setLineDash([20, 18]);
  ctx.strokeStyle = 'rgba(250, 250, 250, 0.65)';
  ctx.stroke();
  ctx.setLineDash([]);

  ctx.beginPath();
  for (let i = 0; i <= 600; i++) {
    const p = sampleTrack(i / 600, 0);
    if (i === 0) ctx.moveTo(p.x, p.y);
    else ctx.lineTo(p.x, p.y);
  }
  ctx.closePath();
  ctx.lineWidth = 4;
  ctx.strokeStyle = 'rgba(255,255,255,0.2)';
  ctx.stroke();
}

function drawBoostPad(item) {
  if (!item.active) return;
  const pos = sampleTrack(item.progress, item.laneOffset);
  const glow = 1 + Math.sin(performance.now() / 150) * 0.2;

  ctx.beginPath();
  ctx.arc(pos.x, pos.y, item.radius * glow, 0, Math.PI * 2);
  ctx.fillStyle = 'rgba(255, 211, 92, 0.28)';
  ctx.fill();

  ctx.beginPath();
  ctx.arc(pos.x, pos.y, item.radius * 0.72, 0, Math.PI * 2);
  ctx.fillStyle = '#ffe38a';
  ctx.fill();

  ctx.strokeStyle = '#f3a700';
  ctx.lineWidth = 2;
  ctx.stroke();

  ctx.strokeStyle = '#fff7c4';
  ctx.beginPath();
  ctx.moveTo(pos.x - 7, pos.y);
  ctx.lineTo(pos.x + 7, pos.y);
  ctx.moveTo(pos.x, pos.y - 7);
  ctx.lineTo(pos.x, pos.y + 7);
  ctx.stroke();
}

function drawCar(x, y, angle, color, isPlayer = false) {
  ctx.save();
  ctx.translate(x, y);
  ctx.rotate(angle + Math.PI / 2);

  ctx.fillStyle = isPlayer ? '#7ef9ff' : color;
  ctx.strokeStyle = isPlayer ? '#dffcff' : '#f9f9f9';
  ctx.lineWidth = 2;

  ctx.beginPath();
  ctx.moveTo(18, 0);
  ctx.lineTo(8, 18);
  ctx.lineTo(-10, 18);
  ctx.lineTo(-22, 0);
  ctx.lineTo(-10, -18);
  ctx.lineTo(8, -18);
  ctx.closePath();
  ctx.fill();
  ctx.stroke();

  ctx.fillStyle = '#132330';
  ctx.fillRect(-15, -10, 10, 20);
  ctx.fillRect(4, -10, 10, 20);

  if (isPlayer) {
    ctx.fillStyle = '#ffd166';
    ctx.fillRect(-3, -15, 6, 8);
  }

  ctx.restore();
}

function updatePlayer(dt) {
  const player = state.player;
  const accelerate = (keys['ArrowUp'] || keys['w']) ? 1 : 0;
  const brake = (keys['ArrowDown'] || keys['s']) ? 1 : 0;
  const left = (keys['ArrowLeft'] || keys['a']) ? 1 : 0;
  const right = (keys['ArrowRight'] || keys['d']) ? 1 : 0;
  const boost = keys[' '] ? 1 : 0;

  const desiredSpeed = accelerate ? 185 : 0;
  const brakeForce = brake ? 210 : 0;

  player.speed += (desiredSpeed - player.speed) * 0.05;
  player.speed -= brakeForce * dt * 0.8;
  player.speed *= 0.992;
  player.speed = clamp(player.speed, 0, 240);

  if (boost && player.boost > 0) {
    player.speed += 220 * dt;
    player.boost = Math.max(0, player.boost - dt * 0.7);
  } else {
    player.boost = Math.min(1, player.boost + dt * 0.16);
  }

  const steeringInput = (right - left);
  player.laneOffset += steeringInput * player.speed * dt * 0.22;
  player.laneOffset *= 0.92;
  player.laneOffset = clamp(player.laneOffset, -72, 72);

  const speedScale = player.speed / 180;
  player.progress += (player.speed * dt * 0.00092) * (1 + (boost && player.boost > 0 ? 0.3 : 0));
  player.progress = wrapProgress(player.progress);

  const pos = sampleTrack(player.progress, player.laneOffset);
  player.angle = pos.angle;

  const prevProgress = player.progress;
  if (prevProgress < 0.12 && player.lapHold > 0.8) {
    player.lap += 1;
    player.lapHold = prevProgress;
  }
  player.lapHold = prevProgress;

  if (player.progress < 0.12 && player.lastProgress > 0.88) {
    player.lap += 1;
  }
  player.lastProgress = player.progress;

  for (const item of state.items) {
    if (!item.active) continue;
    const posItem = sampleTrack(item.progress, item.laneOffset);
    const dx = posItem.x - pos.x;
    const dy = posItem.y - pos.y;
    const dist = Math.hypot(dx, dy);
    if (dist < 24) {
      item.active = false;
      player.boost = 1;
      player.speed = Math.min(player.speed + 55, 260);
    }
  }

  const rank = getRank();
  speedValue.textContent = `${Math.round(player.speed)}`;
  lapValue.textContent = `${Math.min(player.lap, 3)}`;
  rankValue.textContent = `${rank}/${state.ai.length + 1}`;
}

function updateAi(dt) {
  for (const car of state.ai) {
    car.speed += (160 + Math.sin((car.progress + car.laneOffset) * 20) * 18 - car.speed) * 0.06;
    car.progress += car.speed * dt * 0.00095;
    car.progress = wrapProgress(car.progress);
    car.laneOffset += Math.sin((performance.now() / 1000) + car.progress * 10) * 0.7;
    car.laneOffset = clamp(car.laneOffset, -58, 58);
  }
}

function getRank() {
  const playerDistance = state.player.lap * 1 + state.player.progress;
  const ranks = state.ai.map((car) => ({
    name: car.name,
    score: car.lap + car.progress,
  }));

  let rank = 1;
  for (const car of ranks) {
    if (car.score > playerDistance) rank += 1;
  }
  return rank;
}

function render() {
  drawBackground();
  drawTrack();

  for (const item of state.items) drawBoostPad(item);

  for (const car of state.ai) {
    const pos = sampleTrack(car.progress, car.laneOffset);
    drawCar(pos.x, pos.y, pos.angle, car.color, false);
  }

  const p = sampleTrack(state.player.progress, state.player.laneOffset);
  drawCar(p.x, p.y, p.angle, state.player.color, true);

  if (state.player.boost > 0) {
    ctx.beginPath();
    ctx.arc(p.x - Math.cos(p.angle + Math.PI / 2) * 18, p.y - Math.sin(p.angle + Math.PI / 2) * 18, 16, 0, Math.PI * 2);
    ctx.fillStyle = 'rgba(126, 249, 255, 0.65)';
    ctx.fill();
  }
}

function loop(timestamp) {
  const dt = Math.min((timestamp - state.lastTime) / 1000 || 0.016, 0.033);
  state.lastTime = timestamp;

  updatePlayer(dt);
  updateAi(dt);
  render();
  requestAnimationFrame(loop);
}

window.addEventListener('keydown', (event) => {
  const key = event.key.length === 1 ? event.key.toLowerCase() : event.key;
  keys[key] = true;
  if (['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight', ' '].includes(event.key)) {
    event.preventDefault();
  }
});

window.addEventListener('keyup', (event) => {
  const key = event.key.length === 1 ? event.key.toLowerCase() : event.key;
  keys[key] = false;
});

state.player.lapHold = state.player.progress;
state.player.lastProgress = state.player.progress;
requestAnimationFrame(loop);
