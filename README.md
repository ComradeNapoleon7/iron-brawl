<!DOCTYPE html>
<html>
<head>
  <title>WW2-themed PvP Air and Tank game</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.4.2/p5.min.js"></script>
  <style>
    body {
      margin: 0;
      overflow: hidden;
      display: flex; /* Use flexbox to center content */
      justify-content: center; /* Center horizontally */
      align-items: center; /* Center vertically */
      min-height: 100vh; /* Ensure body takes full viewport height */
      background-color: #333; /* Dark background for outside canvas */
    }
    canvas {
      display: block; /* Remove canvas margin */
      border: 1px solid #555; /* Optional: add a subtle border to the canvas */
    }
    #fullscreenButton {
      position: absolute; /* Position the button over the canvas */
      bottom: 20px; /* Adjust as needed */
      right: 20px; /* Adjust as needed */
      padding: 10px 15px;
      font-size: 16px;
      background-color: #4CAF50; /* Green */
      color: white;
      border: none;
      border-radius: 5px;
      cursor: pointer;
      z-index: 100; /* Ensure button is above canvas */
    }
    #fullscreenButton:hover {
      background-color: #45a049;
    }
  </style>
</head>
<body>

<script>
// --- Your existing P5.js sketch code goes here ---
// (All the code you provided from 'let tank;' down to 'return tNear <= tFar && tNear >= 0 && tFar <= 1;')
// --- End of your existing P5.js sketch code ---

let tank;
let bullets = [];
let obstacles = [];
let cameraOffset;
let cameraTargetOffset;
let worldWidth = 8000;
let worldHeight = 8000;
let zoomLevel = 1;
let targetZoom = 1;
let minZoom = 1;
let maxZoom = 0.1;
const FIRING_RANGE_RADIUS = 2250; // 4.5 km diameter = 4500 pixels

function setup() {
  createCanvas(800, 600);
  noCursor(); // Hide the mouse cursor
  tank = new Tank(worldWidth / 2, worldHeight / 2);
  cameraOffset = createVector(0, 0);
  cameraTargetOffset = createVector(0, 0);
  for (let i = 0; i < 30; i++) {
    let w = random(100, 200);
    let h = random(100, 200);
    let x = random(w / 2, worldWidth - w / 2);
    let y = random(h / 2, worldHeight - h / 2);
    if (abs(x - tank.pos.x) > w / 2 + 35 && abs(y - tank.pos.y) > h / 2 + 18) {
      obstacles.push(new Obstacle(x, y, w, h));
    } else {
      i--;
    }
  }
  let wallThickness = 100;
  obstacles.push(new Obstacle(worldWidth / 2, wallThickness / 2, worldWidth, wallThickness));
  obstacles.push(new Obstacle(worldWidth / 2, worldHeight - wallThickness / 2, worldWidth, wallThickness));
  obstacles.push(new Obstacle(wallThickness / 2, worldHeight / 2, wallThickness, worldHeight));
  obstacles.push(new Obstacle(worldWidth - wallThickness / 2, worldHeight / 2, wallThickness, worldHeight));

  // Create the fullscreen button in setup
  let fsButton = createButton('Fullscreen');
  fsButton.id('fullscreenButton'); // Assign an ID for styling
  fsButton.mousePressed(toggleFullscreen);
}

function draw() {
  background(200, 220, 170);

  zoomLevel = lerp(zoomLevel, targetZoom, 0.1); // Smooth zoom

  let moveDistX = width / 4;
  let moveDistY = height / 4;
  if (keyIsDown(LEFT_ARROW)) {
    cameraTargetOffset.x -= moveDistX;
  }
  if (keyIsDown(RIGHT_ARROW)) {
    cameraTargetOffset.x += moveDistX;
  }
  if (keyIsDown(UP_ARROW)) {
    cameraTargetOffset.y -= moveDistY;
  }
  if (keyIsDown(DOWN_ARROW)) {
    cameraTargetOffset.y += moveDistY;
  }

  cameraOffset.lerp(cameraTargetOffset, 0.1);

  let cameraPos = createVector(tank.pos.x + cameraOffset.x, tank.pos.y + cameraOffset.y);
  let viewWidth = width / zoomLevel;
  let viewHeight = height / zoomLevel;
  cameraPos.x = constrain(cameraPos.x, viewWidth / 2, worldWidth - viewWidth / 2);
  cameraPos.y = constrain(cameraPos.y, viewHeight / 2, worldHeight - viewHeight / 2);

  push();
  translate(width / 2, height / 2);
  scale(zoomLevel);
  translate(-cameraPos.x, -cameraPos.y);

  push();
  translate(tank.pos.x, tank.pos.y);
  stroke(255, 0, 0);
  strokeWeight(2 / zoomLevel);
  noFill();
  ellipse(0, 0, FIRING_RANGE_RADIUS * 2, FIRING_RANGE_RADIUS * 2);
  noStroke();
  pop();

  tank.update();
  tank.display();

  for (let obstacle of obstacles) {
    obstacle.display();
  }

  for (let i = bullets.length - 1; i >= 0; i--) {
    bullets[i].update();
    if (bullets[i].hitsObstacle() || bullets[i].hitsFiringRange()) {
      bullets.splice(i, 1);
      continue;
    }
    bullets[i].display();
    if (bullets[i].offscreen()) {
      bullets.splice(i, 1);
    }
  }

  pop();

  let mousePos = createVector(mouseX, mouseY);
  let worldMousePos = mousePos.copy();
  worldMousePos.sub(createVector(width / 2, height / 2));
  worldMousePos.div(zoomLevel);
  worldMousePos.add(cameraPos);

  // Draw mouse crosshair
  push();
  translate(mouseX, mouseY);
  let now = millis();
  let progress = (now - tank.lastShot) / tank.shotCooldown;
  let crosshairColor = (now - tank.lastShot < tank.shotCooldown)
    ? (progress >= 0.95 && progress <= 0.9944 ? color(255, 255, 255) : color(255, 165, 0)) // White flash at 95%-99.44% reload, else orange
    : (tank.shotBlocked(worldMousePos) ? color(255, 0, 0) : color(0, 255, 0)); // Red if blocked, green if clear
  stroke(crosshairColor);
  strokeWeight(2);
  noFill();
  ellipse(0, 0, 20, 20);
  line(-12, 0, 12, 0);
  line(0, -12, 0, 12);
  // Draw reload animation if reloading
  if (now - tank.lastShot < tank.shotCooldown) {
    let angle = progress * TWO_PI;
    stroke(progress >= 0.95 && progress <= 0.9944 ? color(255, 255, 255) : color(255, 165, 0)); // White flash at 95%-99.44% reload, else orange
    strokeWeight(4); // Original thickness
    arc(0, 0, 20, 20, -HALF_PI, -HALF_PI + angle);
  }
  noStroke();
  pop();

  // Draw turret crosshair
  let mouseDist = dist(tank.pos.x, tank.pos.y, worldMousePos.x, worldMousePos.y);
  let turretDir = p5.Vector.fromAngle(tank.angle + tank.turretAngle);
  let turretPos = tank.pos.copy().add(turretDir.mult(mouseDist));
  let turretScreenPos = turretPos.copy();
  turretScreenPos.sub(cameraPos);
  turretScreenPos.mult(zoomLevel);
  turretScreenPos.add(createVector(width / 2, height / 2));
  push();
  translate(turretScreenPos.x, turretScreenPos.y);
  stroke(255, 255, 255); // White turret crosshair
  strokeWeight(1); // Smaller and thinner
  noFill();
  ellipse(0, 0, 10, 10); // Smaller than mouse crosshair (20)
  line(-6, 0, 6, 0);
  line(0, -6, 0, 6);
  noStroke();
  pop();

  push();
  translate(width - 110, 10);
  fill(200, 220, 170, 200);
  rect(0, 0, 100, 100);
  let mapScale = 100 / worldWidth;
  push();
  scale(mapScale);
  // Draw firing range on minimap
  stroke(255, 0, 0);
  strokeWeight(1 / mapScale);
  noFill();
  ellipse(tank.pos.x, tank.pos.y, FIRING_RANGE_RADIUS * 2, FIRING_RANGE_RADIUS * 2);
  noStroke();
  for (let obstacle of obstacles) {
    fill(150, 100, 50);
    rectMode(CENTER);
    rect(obstacle.pos.x, obstacle.pos.y, obstacle.w, obstacle.h);
  }
  fill(255);
  ellipse(tank.pos.x, tank.pos.y, 5 / mapScale, 5 / mapScale);
  noFill();
  stroke(255);
  strokeWeight(1 / mapScale);
  rect(cameraPos.x, cameraPos.y, viewWidth, viewHeight);
  noStroke();
  pop();
  pop();
}

function keyPressed() {
  if (keyCode === 67) { // 'C' key
    cameraTargetOffset.set(0, 0);
  }
}

function mouseWheel(event) {
  let zoomChange = -event.delta / 1000;
  targetZoom += zoomChange;
  targetZoom = constrain(targetZoom, maxZoom, minZoom);
  return false;
}

function mousePressed() {
  tank.shoot();
}

// Function to handle fullscreen
function toggleFullscreen() {
  if (!document.fullscreenElement) {
    document.documentElement.requestFullscreen().catch(err => {
      console.log(`Error attempting to enable fullscreen: ${err.message} (${err.name})`);
    });
  } else {
    document.exitFullscreen();
  }
}

class Tank {
  constructor(x, y) {
    this.pos = createVector(x, y);
    this.angle = 0;
    this.turretAngle = 0;
    this.vel = createVector(0, 0);
    this.angularVel = 0;
    this.lastShot = 0;
    this.shotCooldown = 4500;
  }

  update() {
    let accForward = 0.1; // Halved from 0.2 to make acceleration slightly harder
    let accBackward = 0.0375; // Halved from 0.075 to maintain proportion
    let maxSpeedForward = 5; // Unchanged: ~40 mph
    let maxSpeedBackward = 1.875; // Unchanged: ~15 mph
    let drag = 0.1;
    let maxAngularVel = 0.5236;

    // W moves forward, S moves backward
    if (keyIsDown(87)) { // W: forward
      let force = p5.Vector.fromAngle(this.angle).mult(accForward);
      this.vel.add(force);
    }
    if (keyIsDown(83)) { // S: backward
      let force = p5.Vector.fromAngle(this.angle).mult(-accBackward);
      this.vel.add(force);
    }

    // A and D turning: no inversion
    if (keyIsDown(65)) { // A: turn left
      this.angularVel -= 0.01;
    }
    if (keyIsDown(68)) { // D: turn right
      this.angularVel += 0.01;
    }

    this.angularVel = constrain(this.angularVel, -maxAngularVel / frameRate(), maxAngularVel / frameRate());

    // Apply appropriate max speed based on key
    let velMag = this.vel.mag();
    let maxSpeed = keyIsDown(83) ? maxSpeedBackward : maxSpeedForward;
    if (velMag > maxSpeed) {
      this.vel.setMag(maxSpeed);
    }

    this.vel.mult(1 - drag);
    this.angularVel *= 0.9;

    let oldPos = this.pos.copy();
    this.pos.add(this.vel);
    this.angle += this.angularVel;

    for (let obstacle of obstacles) {
      if (this.collidesWith(obstacle)) {
        this.pos = oldPos;
        this.vel.set(0, 0);
        break;
      }
    }

    this.pos.x = constrain(this.pos.x, 0, worldWidth);
    this.pos.y = constrain(this.pos.y, 0, worldHeight);

    let cameraPos = createVector(tank.pos.x + cameraOffset.x, tank.pos.y + cameraOffset.y);
    let mousePos = createVector(mouseX, mouseY);
    mousePos.sub(createVector(width / 2, height / 2));
    mousePos.div(zoomLevel);
    mousePos.add(cameraPos);
    let toMouse = mousePos.sub(this.pos);
    let desiredAngle = toMouse.heading();
    let currentAngle = this.angle + this.turretAngle;
    let angleDiff = (desiredAngle - currentAngle + PI) % TWO_PI - PI;
    let maxTurretSpeed = 0.6109;
    this.turretAngle += constrain(angleDiff, -maxTurretSpeed / frameRate(), maxTurretSpeed / frameRate());
  }

  shoot() {
    let now = millis();
    if (now - this.lastShot > this.shotCooldown) {
      let turretDir = p5.Vector.fromAngle(this.angle + this.turretAngle);
      let turretPos = this.pos.copy().add(p5.Vector.fromAngle(this.angle).mult(10)).add(turretDir.mult(16));
      bullets.push(new Bullet(turretPos, turretDir));
      this.lastShot = now;
    }
  }

  shotBlocked(target) {
    let turretPos = this.pos.copy().add(p5.Vector.fromAngle(this.angle).mult(10)).add(p5.Vector.fromAngle(this.angle + this.turretAngle).mult(16));
    for (let obstacle of obstacles) {
      // Check if line of sight intersects obstacle
      if (lineIntersectsRect(turretPos, target, obstacle)) {
        return true;
      }
      // Check if target is over obstacle
      let obsHalfW = obstacle.w / 2;
      let obsHalfH = obstacle.h / 2;
      if (abs(target.x - obstacle.pos.x) < obsHalfW && abs(target.y - obstacle.pos.y) < obsHalfH) {
        return true;
      }
    }
    return false;
  }

  collidesWith(obstacle) {
    let tankHalfW = 35;
    let tankHalfH = 18;
    let obsHalfW = obstacle.w / 2;
    let obsHalfH = obstacle.h / 2;

    return (abs(this.pos.x - obstacle.pos.x) < tankHalfW + obsHalfW &&
            abs(this.pos.y - obstacle.pos.y) < tankHalfH + obsHalfH);
  }

  display() {
    push();
    translate(this.pos.x, this.pos.y);
    rotate(this.angle);
    // Hull with thinner green border
    fill(100, 120, 100);
    stroke(0, 255, 0);
    strokeWeight(1); // 2x thinner (was 2)
    rectMode(CENTER);
    rect(0, 0, 70, 36, 10);
    noStroke();
    // Front marker (flatter triangle with rounded tip)
    fill(0, 255, 0);
    triangle(35, -3, 35, 3, 41, 0); // Flatter (6px base), less protrusive (tip at x=41)
    ellipse(41, 0, 2, 2); // Smaller rounded tip
    translate(10, 0); // Turret position unchanged
    rotate(this.turretAngle);
    fill(80, 100, 80);
    ellipse(0, 0, 20, 20);
    rect(15, 0, 20, 5); // Longer barrel (length 20, centered at x=15)
    pop();
  }
}

class Bullet {
  constructor(pos, dir) {
    this.pos = pos.copy();
    this.vel = dir.copy().mult(3);
  }

  update() {
    this.pos.add(this.vel);
  }

  hitsObstacle() {
    for (let obstacle of obstacles) {
      let obsHalfW = obstacle.w / 2;
      let obsHalfH = obstacle.h / 2;
      if (abs(this.pos.x - obstacle.pos.x) < obsHalfW &&
          abs(this.pos.y - obstacle.pos.y) < obsHalfH) {
        return true;
      }
    }
    return false;
  }

  hitsFiringRange() {
    let distToTank = dist(this.pos.x, this.pos.y, tank.pos.x, tank.pos.y);
    return distToTank >= FIRING_RANGE_RADIUS;
  }

  display() {
    push();
    translate(this.pos.x, this.pos.y);
    fill(255, 0, 0);
    ellipse(0, 0, 5, 5);
    pop();
  }

  offscreen() {
    return this.pos.x < 0 || this.pos.x > worldWidth || this.pos.y < 0 || this.pos.y > worldHeight;
  }
}

class Obstacle {
  constructor(x, y, w, h) {
    this.pos = createVector(x, y);
    this.w = w;
    this.h = h;
  }

  display() {
    push();
    translate(this.pos.x, this.pos.y);
    fill(150, 100, 50);
    rectMode(CENTER);
    rect(0, 0, this.w, this.h, 5);
    pop();
  }
}

function lineIntersectsRect(p1, p2, rect) {
  let rectLeft = rect.pos.x - rect.w / 2;
  let rectRight = rect.pos.x + rect.w / 2;
  let rectTop = rect.pos.y - rect.h / 2;
  let rectBottom = rect.pos.y + rect.h / 2;

  let minX = min(p1.x, p2.x);
  let maxX = max(p1.x, p2.x);
  let minY = min(p1.y, p2.y);
  let maxY = max(p1.y, p2.y);

  if (maxX < rectLeft || minX > rectRight || maxY < rectTop || minY > rectBottom) {
    return false;
  }

  let lineDir = p2.copy().sub(p1);
  let tNear = -Infinity;
  let tFar = Infinity;

  if (lineDir.x !== 0) {
    let tx1 = (rectLeft - p1.x) / lineDir.x;
    let tx2 = (rectRight - p1.x) / lineDir.x;
    tNear = max(tNear, min(tx1, tx2));
    tFar = min(tFar, max(tx1, tx2));
  } else if (p1.x < rectLeft || p1.x > rectRight) {
    return false;
  }

  if (lineDir.y !== 0) {
    let ty1 = (rectTop - p1.y) / lineDir.y;
    let ty2 = (rectBottom - p1.y) / lineDir.y;
    tNear = max(tNear, min(ty1, ty2));
    tFar = min(tFar, max(ty1, ty2));
  } else if (p1.y < rectTop || p1.y > rectBottom) {
    return false;
  }

  return tNear <= tFar && tNear >= 0 && tFar <= 1;
}
</script>
</body>
</html>
