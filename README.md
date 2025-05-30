<!DOCTYPE html>
<html>
<head>
  <title>WW2 PvP Game</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.4.2/p5.min.js"></script>
  <style>
    body { margin: 0; overflow: hidden; display: flex; justify-content: center; align-items: center; min-height: 100vh; background: #333; }
    canvas { border: 1px solid #555; }
    button { position: absolute; padding: 10px 15px; font-size: 16px; background: #4CAF50; color: white; border: none; border-radius: 5px; cursor: pointer; z-index: 100; }
    button:hover { background: #45a049; }
    #tankButton { top: 40%; left: 40%; }
    #airplaneButton { top: 40%; left: 50%; }
    #playButton { top: 50%; left: 45%; display: none; }
    #fullscreenButton { bottom: 20px; right: 20px; }
  </style>
</head>
<body>
<script>
  let tank_turret_img, tank_hull_img, player, vType, teammates = [], enemies = [], bullets = [], bombs = [], obstacles = [], messages = [];
  let camOff, camTgt, camPos, cycleIdx = 0, cycleList = [], worldW = 8000, worldH = 8000;
  let zoom = 1, tgtZoom = 1, minZoom = 1, maxZoom = 0.05, gameState = 'start', range = 2250;
  let tankBtn, planeBtn, playBtn;

  function preload() {
    try {
      tank_turret_img = loadImage('assets/tank_turret.png');
      tank_hull_img = loadImage('assets/tank_hull.png');
    } catch (e) {
      console.warn('Image load failed');
    }
  }

  function setup() {
    createCanvas(800, 500);
    resetGame();
    tankBtn = createButton('Tank').attribute('id', 'tankButton').mousePressed(() => select('tank'));
    planeBtn = createButton('Airplane').attribute('id', 'airplaneButton').mousePressed(() => select('airplane'));
    playBtn = createButton('Play').attribute('id', 'playButton').mousePressed(startGame).hide();
    createButton('Fullscreen').attribute('id', 'fullscreenButton').mousePressed(() => fullscreen(!fullscreen()));
    camOff = createVector(0, 0);
    camPos = createVector(0, 0);
  }

  function resetGame() {
    obstacles = [];
    bullets = [];
    bombs = [];
    teammates = [];
    enemies = [];
    messages = [];
    camOff = createVector(0, 0);
    camPos = createVector(0, 0);
    zoom = 1;
    tgtZoom = 1;
    for (let i = 0; i < 30; i++) {
      let w = random(100, 200), h = random(100, 200), x = random(w/2, worldW - w/2), y = random(h/2, worldH - h/2);
      if (x >= 1000 && x <= 7000) obstacles.push({ pos: createVector(x, y), w, h });
      else i--;
    }
    let wt = 100;
    obstacles.push(
      { pos: createVector(worldW/2, wt/2), w: worldW, h: wt },
      { pos: createVector(worldW/2, worldH-wt/2), w: worldW, h: wt },
      { pos: createVector(wt/2, worldH/2), w: wt, h: worldH },
      { pos: createVector(worldW-wt/2, worldH/2), w: wt, h: worldH }
    );
    enemies.push(
      new Tank(7400, 3900, 'enemy'),
      new Tank(7400, 4000, 'enemy'),
      new Airplane(7400, 4100, 'enemy'),
      new Airplane(7400, 4200, 'enemy')
    );
  }

  function select(type) {
    vType = type;
    tankBtn.hide();
    planeBtn.hide();
    playBtn.show();
  }

  function startGame() {
    gameState = 'playing';
    tankBtn.hide();
    planeBtn.hide();
    playBtn.hide();
    resetGame();
    player = vType === 'tank' ? new Tank(600, 4000, 'player') : new Airplane(600, 4000, 'player');
    teammates = vType === 'tank' ?
      [new Tank(600, 3900, 'player'), new Airplane(600, 4100, 'player'), new Airplane(600, 4200, 'player')] :
      [new Airplane(600, 4100, 'player'), new Tank(600, 3900, 'player'), new Tank(600, 4200, 'player')];
    cycleList = [player, ...teammates].filter(v => vType === 'tank' ? v instanceof Tank : v instanceof Airplane);
    cycleIdx = 0;
    camTgt = player;
    camPos.set(player.pos);
  }

  function wrapGameText(txt, maxW) {
    let words = txt.split(' '), lines = [], ln = '';
    for (let word of words) {
      let tLn = ln ? ln + ' ' + word : word;
      if (textWidth(tLn) <= maxW) ln = tLn;
      else { lines.push(ln); ln = word; }
    }
    if (ln) lines.push(ln);
    return lines;
  }

  class Tank {
    constructor(x, y, team) {
      this.pos = createVector(x, y);
      this.angle = team === 'player' ? 0 : PI;
      this.turretAngle = 0;
      this.vel = createVector(0, 0);
      this.angVel = 0;
      this.hp = 500;
      this.maxHP = 500;
      this.team = team;
      this.state = 'alive';
      this.lastShot = 0;
      this.shotCD = 4500;
      this.expTime = 0;
      this.shell = 'HE';
      this.fire = 'none';
      this.fireStart = 0;
      this.extUsed = -30000;
      this.noFire = false;
      this.noFireEnd = 0;
      this.noEng = false;
      this.noEngEnd = 0;
      this.noTur = false;
      this.noTurEnd = 0;
      this.killed = false;
    }

    update() {
      if (this.state === 'exploding') { this.expTime += 1000 / frameRate(); return; }
      if (this.hp <= 0) { this.state = 'exploding'; this.expTime = 0; return; }

      let accF = 0.0715, accB = 0.0268125, maxF = this.noEng ? 1.7 : 3.4, maxB = this.noEng ? 0.64 : 1.28;
      let turnMax = 3.08, drag = 0.05, maxAng = 0.5236;

      if (this.team === 'player' && this === player) {
        if (keyIsDown(87)) this.vel.add(p5.Vector.fromAngle(this.angle).mult(accF));
        if (keyIsDown(83)) this.vel.add(p5.Vector.fromAngle(this.angle).mult(-accB));
        if (keyIsDown(65)) this.angVel -= 0.01;
        if (keyIsDown(68)) this.angVel += 0.01;
      } else {
        let tgt = this.team === 'player' ? player.pos : teammates.reduce((a, v) => a.add(v.pos), player.pos.copy()).div(teammates.length + 1);
        let toTgt = p5.Vector.sub(tgt, this.pos);
        let desAng = toTgt.heading();
        this.angVel += constrain(atan2(sin(desAng - this.angle), cos(desAng - this.angle)), -maxAng / frameRate(), maxAng / frameRate());
        if (toTgt.mag() > 100) this.vel.add(p5.Vector.fromAngle(this.angle).mult(accF));
        if (this.team === 'enemy' && !this.noFire && millis() - this.lastShot > this.shotCD) {
          let dir = p5.Vector.fromAngle(this.angle + this.turretAngle);
          bullets.push(new Bullet(
            this.pos.copy().add(p5.Vector.fromAngle(this.angle).mult(10)).add(dir.mult(16)),
            dir,
            'tank',
            this.team,
            this.shell
          ));
          this.lastShot = millis();
        }
      }

      this.angVel = constrain(this.angVel, -maxAng / frameRate(), maxAng / frameRate());
      let velMag = this.vel.mag();
      let maxS = keyIsDown(83) && this.team === 'player' && this === player ?
        lerp(turnMax * (maxB / maxF), maxB, 1 - abs(this.angVel) / (maxAng / frameRate())) :
        lerp(turnMax, maxF, 1 - abs(this.angVel) / (maxAng / frameRate()));
      if (velMag > maxS) this.vel.setMag(maxS);
      this.vel.mult(1 - drag);
      this.angVel *= 0.9;

      let oldPos = this.pos.copy();
      this.pos.add(this.vel);
      this.angle += this.angVel;
      for (let o of obstacles) if (this.collides(o)) { this.pos = oldPos; this.vel.set(0, 0); break; }
      this.pos.x = constrain(this.pos.x, 0, worldW);
      this.pos.y = constrain(this.pos.y, 0, worldH);

      if (!this.noTur || millis() > this.noTurEnd) {
        let mPos = createVector(mouseX, mouseY).sub(width / 2, height / 2).div(zoom).add(camPos);
        let toM = p5.Vector.sub(mPos, this.pos);
        let desAng = toM.heading();
        let curAng = this.angle + this.turretAngle;
        let diff = atan2(sin(desAng - curAng), cos(desAng - curAng));
        this.turretAngle += Math.sign(diff) * Math.min(abs(diff), 0.6109 / frameRate());
      }

      if (this.fire === 'burning' && millis() - this.fireStart <= 20000) {
        this.hp -= 10 / frameRate();
        if (this.hp <= 0) { this.state = 'exploding'; this.expTime = 0; }
      } else this.fire = 'none';
      if (this.noFire && millis() > this.noFireEnd) this.noFire = false;
      if (this.noEng && millis() > this.noEngEnd) this.noEng = false;
      if (this.noTur && millis() > this.noTurEnd) this.noTur = false;
    }

    display() {
      push();
      translate(this.pos.x, this.pos.y);
      rotate(this.angle);
      if (this.state === 'exploding') {
        let a = map(this.expTime, 0, 1000, 255, 0);
        fill(255, 165, 0, a);
        noStroke();
        randomSeed(this.pos.x * 1000 + this.pos.y);
        for (let i = 0; i < 10; i++) ellipse(random(-100, 100), random(-100, 100), random(50, 100));
        pop();
        return;
      }
      if (tank_hull_img) {
        imageMode(CENTER);
        image(tank_hull_img, 0, 0, 70, 36);
      } else {
        fill(this === player ? [0, 255, 0] : this.team === 'player' ? [0, 0, 255] : [255, 0, 0]);
        stroke(100);
        strokeWeight(1 / zoom);
        rectMode(CENTER);
        rect(0, 0, 70, 36, 10);
        noStroke();
        ellipse(41, 0, 2, 2);
      }
      translate(10, 0);
      rotate(this.turretAngle);
      if (tank_turret_img) {
        imageMode(CENTER);
        image(tank_turret_img, 0, 0, 50, 20);
      } else {
        fill(this === player ? [0, 255, 0] : this.team === 'player' ? [0, 0, 255] : [255, 0, 0]);
        stroke(100);
        strokeWeight(1 / zoom);
        ellipse(0, 0, 20, 20);
        rect(15, 0, 30, 5);
      }
      pop();
      if (this.fire === 'burning') {
        push();
        translate(this.pos.x, this.pos.y);
        rotate(this.angle);
        fill(255, 165, 0);
        noStroke();
        for (let i = 0; i < 5; i++) ellipse(random(-30, -20), random(-10, 10), random(10, 20));
        pop();
      }
      if (zoom <= 0.1 && this === player && vType === 'tank') {
        push();
        translate(this.pos.x, this.pos.y);
        fill(0, 255, 0);
        noStroke();
        triangle(-5, -18, 5, -18, 0, -28);
        pop();
      }
    }

    collides(o) {
      let t = { l: this.pos.x - 35, r: this.pos.x + 35, t: this.pos.y - 18, b: this.pos.y + 18 };
      let obs = { l: o.pos.x - o.w / 2, r: o.pos.x + o.w / 2, t: o.pos.y - o.h / 2, b: o.pos.y + o.h / 2 };
      return !(t.r < obs.l || t.l > obs.r || t.b < obs.t || t.t > obs.b);
    }

    shotBlocked(tgt) {
      let tPos = this.pos.copy().add(p5.Vector.fromAngle(this.angle).mult(10)).add(p5.Vector.fromAngle(this.angle + this.turretAngle).mult(16));
      let toTgt = p5.Vector.sub(tgt, tPos);
      let dist = toTgt.mag();
      let dir = p5.Vector.fromAngle(this.angle + this.turretAngle);
      let test = tPos.copy();
      for (let i = 0; i < floor(dist / 10) + 1; i++) {
        for (let o of obstacles) if (test.x >= o.pos.x - o.w / 2 && test.x <= o.pos.x + o.w / 2 && test.y >= o.pos.y - o.h / 2 && test.y <= o.pos.y + o.h / 2) return true;
        test.add(dir.copy().setMag(10));
      }
      return false;
    }

    getHitZone(pt, bVel) {
      let loc = p5.Vector.sub(pt, this.pos).rotate(-this.angle);
      if (abs(loc.x) > 35 || abs(loc.y) > 18) return null;
      if (keyIsDown(84) && dist(loc.x, loc.y, 10, 0) <= 10) return { zone: 'turret', norm: createVector(loc.x - 10, loc.y).normalize() };
      let impDir = bVel.copy().normalize();
      let tankFwd = createVector(cos(this.angle), sin(this.angle));
      let cosTh = impDir.dot(tankFwd);
      let impAng = acos(cosTh) * 180 / PI;
      if (loc.x >= 0 && loc.x <= 35 && impAng < 45) return { zone: 'front', norm: createVector(1, 0) };
      if (loc.x >= -35 && loc.x < 0 && impAng > 135) return { zone: 'rear', norm: createVector(-1, 0) };
      return { zone: 'side', norm: loc.y < 0 ? createVector(0, -1) : createVector(0, 1) };
    }
  }

  class Airplane {
    constructor(x, y, team) {
      this.pos = createVector(x, y);
      this.angle = team === 'player' ? 0 : PI;
      this.vel = p5.Vector.fromAngle(this.angle).mult(5.274);
      this.angVel = 0;
      this.hp = 400;
      this.maxHP = 400;
      this.team = team;
      this.state = 'flying';
      this.lastShot = 0;
      this.shotCD = 500;
      this.lastBomb = 0;
      this.bombCD = 1000;
      this.bombs = 2;
      this.expTime = 0;
      this.lastMPos = createVector(mouseX, mouseY);
      this.mMoved = false;
      this.edgeWarn = team === 'player' ? -1 : null;
      this.eng = 'normal';
      this.engEnd = 0;
      this.fireStart = 0;
      this.jam = false;
      this.jamEnd = 0;
      this.killed = false;
    }

    update() {
      if (this.state === 'exploding') { this.expTime += 1000 / frameRate(); return; }
      if (this.hp <= 0) { this.state = 'exploding'; this.expTime = 0; }

      let acc = 0.3, maxS = this.eng === 'normal' ? 6.153 : 4.5, minS = 3.516, maxAng = 0.99525888;
      if (this.team === 'player' && this === player) {
        if (keyIsDown(87)) this.vel.setMag(this.vel.mag() + acc);
        if (keyIsDown(83)) this.vel.setMag(max(0, this.vel.mag() - acc));
        let mPos = createVector(mouseX, mouseY).sub(width / 2, height / 2).div(zoom).add(camPos);
        let toM = p5.Vector.sub(mPos, this.pos);
        let desAng = toM.heading();
        let diff = atan2(sin(desAng - this.angle), cos(desAng - this.angle));
        if (dist(mouseX, mouseY, this.lastMPos.x, this.lastMPos.y) > 0.1) this.mMoved = true;
        this.lastMPos.set(mouseX, mouseY);
        if (abs(diff) <= 0.01745 && !this.mMoved) {
          this.angle = desAng;
          this.angVel = 0;
        } else {
          this.angVel += diff * 0.99525888 / frameRate();
          this.angVel = constrain(this.angVel, -maxAng / frameRate(), maxAng / frameRate());
          this.mMoved = false;
        }
      } else {
        let tgt = this.team === 'player' ? player.pos : teammates.reduce((a, v) => a.add(v.pos), player.pos.copy()).div(teammates.length + 1);
        let toTgt = p5.Vector.sub(tgt, this.pos);
        let desAng = toTgt.heading();
        this.angVel += atan2(sin(desAng - this.angle), cos(desAng - this.angle)) * 0.99525888 / frameRate();
        this.angVel = constrain(this.angVel, -maxAng / frameRate(), maxAng / frameRate());
        this.vel.setMag(5.274);
        if (this.team === 'enemy' && !this.jam && millis() - this.lastShot > this.shotCD) {
          [-40, -25, 25, 40].forEach(o => bullets.push(new Bullet(
            this.pos.copy().add(p5.Vector.fromAngle(this.angle + HALF_PI).mult(o)),
            p5.Vector.fromAngle(this.angle + random(-0.05, 0.05)),
            'airplane',
            this.team,
            'bullet'
          )));
          this.lastShot = millis();
        }
        if (this.team === 'enemy' && millis() - this.lastBomb > this.bombCD && this.bombs > 0) {
          for (let i = 0; i < 2; i++) bombs.push(new Bomb(
            this.pos.copy().add(createVector(random(-50, 50), random(-150, 150)).rotate(this.angle)),
            this.angle,
            this.team
          ));
          this.bombs--;
          this.lastBomb = millis();
        }
      }

      this.vel.setMag(constrain(this.vel.mag(), minS, maxS));
      this.pos.add(this.vel);
      this.angle += this.angVel;
      this.angVel *= 0.95;
      if (this.team === 'player' && this === player) {
        let m = 50;
        if (this.pos.x < -m || this.pos.x > worldW + m || this.pos.y < -m || this.pos.y > worldH + m) {
          if (this.edgeWarn === -1) this.edgeWarn = millis();
          else if (millis() - this.edgeWarn >= 20000) {
            this.hp = 0;
            this.state = 'exploding';
            gameState = 'start';
            tankBtn.show();
            planeBtn.show();
          }
        } else this.edgeWarn = -1;
      }
      if (this.jam && millis() > this.jamEnd) this.jam = false;
      if (this.eng === 'disabled' && millis() > this.engEnd) this.eng = 'normal';
    }

    display() {
      push();
      translate(this.pos.x, this.pos.y);
      rotate(this.angle);
      if (this.state === 'flying') {
        fill(this === player ? [0, 255, 0] : this.team === 'player' ? [0, 0, 255] : [255, 0, 0]);
        noStroke();
        ellipse(0, 0, 24, 72);
        translate(-15, 0);
        ellipse(0, 0, 96, 21.6);
      } else {
        let a = map(this.expTime, 0, 1000, 255, 0);
        fill(255, 165, 0, a);
        noStroke();
        randomSeed(this.pos.x * 1000 + this.pos.y);
        for (let i = 0; i < 10; i++) ellipse(random(-100, 100), random(-100, 100), random(50, 100));
      }
      pop();
    }

    getHitZone(pt) {
      let loc = p5.Vector.sub(pt, this.pos).rotate(-this.angle);
      let inFus = (loc.x / 12) ** 2 + (loc.y / 36) ** 2 <= 1;
      let inWing = ((loc.x + 15) / 48) ** 2 + (loc.y / 10.8) ** 2 <= 1;
      if (inFus) return loc.x >= 24 ? 'propeller' : loc.x <= -12 ? 'tail' : 'fuselage';
      if (inWing) return 'wings';
      return null;
    }
  }

  class Bullet {
    constructor(pos, dir, type, team, shell) {
      this.pos = pos.copy();
      this.vel = dir.copy().mult(type === 'tank' ? 30 : 30);
      this.type = type;
      this.team = team;
      this.shell = shell || 'bullet';
    }

    update() { this.pos.add(this.vel); }

    hitsObstacle() {
      if (this.type === 'airplane') return false;
      return obstacles.some(o => this.pos.x >= o.pos.x - o.w / 2 && this.pos.x <= o.pos.x + o.w / 2 && this.pos.y >= o.pos.y - o.h / 2 && this.pos.y <= o.pos.y + o.h / 2);
    }

    hitsRange() { return p5.Vector.dist(this.pos, player.pos) > range; }

    hitsTarget() {
      let tgts = this.team === 'player' ? enemies : [player, ...teammates];
      for (let t of tgts) {
        if (t instanceof Tank) {
          let hit = t.getHitZone(this.pos, this.vel);
          if (!hit) continue;
          let { zone, norm } = hit;
          let cosTh = this.vel.dot(norm.mult(-1)) / (this.vel.mag() * norm.mag() + 0.0001);
          let th = acos(cosTh) * 180 / PI;
          let bounce = this.shell === 'HE' ? { front: 35, side: 40, rear: 45, turret: 30 } : { front: 45, side: 50, rear: 45, turret: 40 };
          if (th <= bounce[zone]) {
            if (t === player) messages.push({ text: 'Failed to penetrate!', time: millis(), dur: 3000 });
            return true;
          }
          let dmg = 0, fireC = 0, disC = 0;
          if (this.shell === 'HE') {
            if (zone === 'front' || zone === 'turret') {
              if (t === player) messages.push({ text: 'Failed to penetrate!', time: millis(), dur: 3000 });
              return true;
            }
            dmg = zone === 'side' ? 120 : 150;
            fireC = zone === 'rear' ? 0.3 : 0;
          } else {
            dmg = zone === 'front' ? 60 : zone === 'side' ? 120 : zone === 'rear' ? 130 : 80;
            fireC = zone === 'rear' ? 0.1 : 0;
            disC = zone === 'turret' ? 0.03 : 0;
            if (zone === 'turret' && random() < 0.2) {
              dmg *= 2;
              if (this.team === 'player') messages.push({ text: 'Critical hit: turret damage', time: millis(), dur: 3000, col: [255, 165, 0] });
            }
            if (zone === 'turret' && random() < 0.05) {
              t.noFire = true;
              t.noFireEnd = millis() + 10000;
              if (this.team === 'player') messages.push({ text: 'Enemy firing disabled', time: millis(), dur: 5000, col: [255, 165, 0] });
            }
            if (zone === 'turret' && random() < 0.05) {
              t.noTur = true;
              t.noTurEnd = millis() + 10000;
              if (this.team === 'player') messages.push({ text: 'Enemy turret disabled', time: millis(), dur: 5000, col: [255, 165, 0] });
            }
          }
          if (dmg > 0) {
            t.hp -= dmg;
            let m = messages.find(m => m.type === (this.team === 'player' ? 'damage' : 'hit') && millis() - m.time <= m.cWin);
            let dtl = { dmg, zone, weapon: this.shell };
            if (m) { m.dtls.push(dtl); m.dmg += dmg; m.time = millis(); }
            else messages.push({ type: this.team === 'player' ? 'damage' : 'hit', dmg, dtls: [dtl], time: millis(), dur: 3000, cWin: 2000 });
          }
          if (fireC > 0 && random() < fireC && t.fire === 'none') {
            t.fire = 'burning';
            t.fireStart = millis();
            messages.push({ text: this.team === 'player' ? 'Enemy engine fire' : 'Engine fire!', time: millis(), dur: 5000, col: this.team === 'player' ? [255, 165, 0] : null });
          }
          if (disC > 0 && random() < disC) {
            t.noFire = true;
            t.noFireEnd = millis() + 10000;
            messages.push({ text: this.team === 'player' ? 'Enemy firing disabled' : 'Firing disabled!', time: millis(), dur: 5000, col: this.team === 'player' ? [255, 165, 0] : null });
          }
          if (t.hp <= 0) {
            t.state = 'exploding';
            t.expTime = 0;
            if (this.team === 'player' && !t.killed) {
              t.killed = true;
              messages.push({ text: `Enemy kill: tank`, time: millis(), dur: 5000 });
            }
            if (t === player) {
              gameState = 'start';
              tankBtn.show();
              planeBtn.show();
            }
          }
          return true;
        } else if (t instanceof Airplane) {
          let zone = t.getHitZone(this.pos);
          if (!zone) continue;
          let dmg = zone === 'wings' || zone === 'tail' ? 20 : zone === 'fuselage' ? 15 : 40;
          if (zone === 'propeller') {
            if (t.eng === 'normal' && random() < 0.05) {
              t.eng = 'damage';
              messages.push({ text: this.team === 'player' ? 'Enemy engine damage' : 'Engine damaged!', time: millis(), dur: 5000, col: this.team === 'player' ? [255, 165, 0] : null });
            } else if (t.eng === 'damage' && random() < 0.1) {
              t.eng = 'disabled';
              t.engEnd = millis() + 30000;
              messages.push({ text: this.team === 'player' ? 'Enemy engine disabled' : 'Engine disabled!', time: millis(), dur: 5000, col: this.team === 'player' ? [255, 165, 0] : null });
            }
          }
          if (dmg > 0) {
            t.hp -= dmg;
            let m = messages.find(m => m.type === (this.team === 'player' ? 'damage' : 'hit') && millis() - m.time <= m.cWin);
            let dtl = { dmg, zone, weapon: 'Bullet' };
            if (m) { m.dtls.push(dtl); m.dmg += dmg; m.time = millis(); }
            else messages.push({ type: this.team === 'player' ? 'damage' : 'hit', dmg, dtls: [dtl], time: millis(), dur: 3000, cWin: 2000 });
          }
          if (t.hp <= 0) {
            t.state = 'exploding';
            t.expTime = 0;
            if (this.team === 'player' && !t.killed) {
              t.killed = true;
              messages.push({ text: `Enemy kill: aircraft`, time: millis(), dur: 5000 });
            }
            if (t === player) {
              gameState = 'start';
              tankBtn.show();
              planeBtn.show();
            }
          }
          return true;
        }
      }
      return false;
    }

    offscreen() { return this.pos.x < 0 || this.pos.x > worldW || this.pos.y < 0 || this.pos.y > worldH; }

    display() {
      push();
      translate(this.pos.x, this.pos.y);
      fill(this.type === 'tank' ? [0, 0, 0] : [200, 130, 130]);
      noStroke();
      ellipse(0, 0, this.type === 'tank' ? 10 : 7);
      pop();
    }
  }

  class Bomb {
    constructor(pos, angle, team) {
      this.pos = pos.copy();
      this.angle = angle;
      this.vel = p5.Vector.fromAngle(angle).mult(0.5);
      this.z = 100;
      this.velZ = -0.5;
      this.state = 'falling';
      this.hitTime = 0;
      this.expTime = 0;
      this.team = team;
    }

    update() {
      if (this.state === 'falling') {
        this.pos.add(this.vel);
        this.z += this.velZ;
        if (this.z <= 0) {
          this.state = 'hit';
          this.hitTime = millis();
          this.z = 0;
          this.hitsTarget();
        }
      } else if (this.state === 'hit') {
        this.expTime = millis() - this.hitTime;
        if (this.expTime >= 1000) this.state = 'done';
      }
    }

    hitsTarget() {
      let tgts = this.team === 'player' ? enemies : [player, ...teammates];
      for (let t of tgts) if (t instanceof Tank) {
        let d = p5.Vector.dist(this.pos, t.pos);
        let dmg = d < 35 ? 150 : d <= 150 ? 100 : d <= 350 ? 50 : 0;
        let fireD = d < 35 ? 10000 : d <= 150 ? 5000 : 0;
        let engD = fireD;
        if (dmg > 0) {
          t.hp -= dmg;
          let m = messages.find(m => m.type === (this.team === 'player' ? 'damage' : 'hit') && millis() - m.time <= m.cWin);
          let dtl = { dmg, zone: 'tank', weapon: 'Bomb' };
          if (m) { m.dtls.push(dtl); m.dmg += dmg; m.time = millis(); }
          else messages.push({ type: this.team === 'player' ? 'damage' : 'hit', dmg, dtls: [dtl], time: millis(), dur: 3000, cWin: 2000 });
        }
        if (fireD > 0) {
          t.noFire = true;
          t.noFireEnd = millis() + fireD;
          messages.push({ text: this.team === 'player' ? 'Enemy firing disabled' : 'Firing disabled', time: millis(), dur: 5000, col: this.team === 'player' ? [255, 165, 0] : null });
        }
        if (engD > 0) {
          t.noEng = true;
          t.noEngEnd = millis() + engD;
          messages.push({ text: this.team === 'player' ? 'Enemy engine disabled' : 'Engine disabled', time: millis(), dur: 5000, col: this.team === 'player' ? [255, 165, 0] : null });
        }
        if (t.hp <= 0) {
          t.state = 'exploding';
          t.expTime = 0;
          if (this.team === 'player' && !t.killed) {
            t.killed = true;
            messages.push({ text: `Enemy kill: tank`, time: millis(), dur: 5000 });
          }
          if (t === player) {
            gameState = 'start';
            tankBtn.show();
            planeBtn.show();
          }
        }
      }
    }

    display() {
      push();
      translate(this.pos.x, this.pos.y);
      rotate(this.angle);
      if (this.state === 'falling') {
        fill(100);
        noStroke();
        ellipse(0, this.z / 10, 10, 10);
      } else {
        let a = map(this.expTime, 0, 1000, 255, 0);
        fill(255, 165, 0, a);
        noStroke();
        randomSeed(this.pos.x * 1000 + this.pos.y);
        for (let i = 0; i < 10; i++) ellipse(random(-100, 100), random(-100, 50), random(50, 100));
      }
      pop();
    }
  }

  function draw() {
    if (gameState === 'start') {
      background(100);
      noStroke();
      cursor();
      textAlign(CENTER, CENTER);
      textSize(32);
      fill(255);
      text('Select Vehicle', width / 2, height / 2);
      return;
    }
    if (mouseX >= 0 && mouseX <= width && mouseY >= 0 && mouseY <= height) noCursor();
    else cursor();
    background(200, 220, 170);
    zoom = lerp(zoom, tgtZoom, 0.1);
    let moveX = keyIsDown(LEFT_ARROW) ? -width / 8 : keyIsDown(RIGHT_ARROW) ? width / 8 : 0;
    let moveY = keyIsDown(UP_ARROW) ? -height / 8 : keyIsDown(DOWN_ARROW) ? height / 8 : 0;
    camOff.lerp(moveX, moveY, 0.1);
    camPos.lerp(camTgt.pos.copy().add(camOff), 0.5);
    push();
    translate(width / 2, height / 2);
    scale(zoom);
    translate(-camPos.x, -camPos.y);
    push();
    translate(player.pos.x, player.pos.y);
    stroke(255, 0, 0);
    strokeWeight(2 / zoom);
    noFill();
    ellipse(0, 0, range * 2, range * 2);
    pop();
    obstacles.forEach(o => {
      push();
      translate(o.pos.x, o.pos.y);
      fill(150, 100, 50);
      noStroke();
      rectMode(CENTER);
      rect(0, 0, o.w, o.h);
      pop();
    });
    [enemies, teammates, [player]].forEach((list, i) => list.forEach(v => {
      if (v instanceof Airplane && (!i || v === player)) {
        v.update();
        if (v.state !== 'exploding' || v.expTime < 1000) v.display();
      }
    }));
    teammates.forEach(t => {
      if (t instanceof Airplane) {
        push();
        translate(t.pos.x, t.pos.y - 30);
        fill(255, 0, 0);
        rectMode(CENTER);
        rect(0, 0, 60, 5);
        fill(0, 255, 0);
        rectMode(CORNER);
        rect(-30, -2.5, (t.hp / t.maxHP) * 60, 5);
        pop();
      }
    });
    [enemies, teammates, [player]].forEach((list, i) => list.forEach(v => {
      if (v instanceof Tank && (!i || v === player)) {
        v.update();
        if (v.state !== 'exploding' || v.expTime < 1000) v.display();
      }
    }));
    teammates.forEach(t => {
      if (t instanceof Tank) {
        push();
        translate(t.pos.x, t.pos.y - 25);
        fill(255, 255, 0);
        rectMode(CENTER);
        rect(0, 0, 50, 5);
        fill(0, 255, 0);
        rect(-25, -2.5, (t.hp / t.maxHP) * 50, 5);
        pop();
      }
    });
    bombs = bombs.filter(b => { b.update(); if (b.state === 'done') return false; b.display(); return true; });
    bullets = bullets.filter(b => { b.update(); if (b.hitsObstacle() || b.hitsRange() || b.offscreen() || b.hitsTarget()) return false; b.display(); return true; });
    pop();
    let mPos = createVector(mouseX, mouseY);
    let wPos = mPos.copy().sub(width / 2, height / 2).div(zoom).add(camPos);
    cycleList = [player, ...teammates].filter(v => v.state !== 'exploding' && (vType === 'tank' ? v instanceof Tank : v instanceof Airplane));
    if (cycleIdx >= cycleList.length) cycleIdx = 0;
    if (vType === 'airplane' && mouseIsPressed && !player.jam) {
      let now = millis();
      if (now - player.lastShot >= player.shotCD) {
        [-40, -25, 25, 40].forEach(o => bullets.push(new Bullet(
          player.pos.copy().add(p5.Vector.fromAngle(player.angle + HALF_PI).mult(o)),
          p5.Vector.fromAngle(player.angle + random(-0.05, 0.05)),
          'airplane',
          'player',
          'bullet'
        )));
        player.lastShot = now;
        if (!player.fireStart) player.fireStart = now;
        if (now - player.fireStart > 4000 && random() < 0.2) {
          player.jam = true;
          player.jamEnd = now + 2000;
          messages.push({ text: 'Machine gun jammed!', time: now, dur: 5000 });
        }
      }
    } else if (vType === 'airplane') player.fireStart = 0;
    if (vType === 'tank') {
      let now = millis();
      let prog = (now - player.lastShot) / player.shotCD;
      let cCol = now - player.lastShot >= player.shotCD && !player.noFire ? [0, 255, 0] : prog >= 0.95 && prog <= 0.9944 && !player.noFire ? [255, 255, 0] : player.noFire ? [255, 0, 0] : [255, 165, 0];
      push();
      translate(mouseX, mouseY);
      stroke(cCol);
      strokeWeight(2);
      noFill();
      ellipse(0, 0, 20, 20);
      line(-12, 0, -6, 0);
      line(6, 0, 12, 0);
      line(0, -12, 0, -6);
      line(0, 6, 0, 12);
      if (now - player.lastShot < player.shotCD && !player.noFire) {
        stroke(prog >= 0.95 ? [255, 255, 0] : [0, 255, 0]);
        strokeWeight(4);
        arc(0, 0, 20, 20, -HALF_PI, -HALF_PI + prog * TWO_PI);
      }
      pop();
      let tDir = p5.Vector.fromAngle(player.angle + player.turretAngle);
      let tPos = player.pos.copy().add(p5.Vector.fromAngle(player.angle).mult(10)).add(tDir.mult(min(dist(player.pos.x, player.pos.y, wPos.x, wPos.y), range)));
      let tsPos = tPos.sub(camPos).mult(zoom).add(width / 2, height / 2);
      push();
      translate(tsPos.x, tsPos.y);
      stroke(player.shotBlocked(wPos) ? [255, 0, 0] : [255, 255, 255]);
      strokeWeight(1);
      noFill();
      ellipse(0, 0, 10, 10);
      line(-6, 0, 6, 0);
      line(0, -6, 0, 6);
      pop();
      push();
      resetMatrix();
      textAlign(LEFT);
      textSize(16);
      fill(255);
      text(`Shell: ${player.shell}`, 10, 50);
      pop();
    } else {
      let now = millis();
      let prog = (now - player.lastShot) / player.shotCD;
      let cCol = now - player.lastShot >= player.shotCD && !player.jam ? [0, 255, 0] : prog >= 0.95 && prog <= 0.9944 && !player.jam ? [255, 255, 0] : player.jam ? [255, 0, 0] : [255, 165, 0];
      push();
      translate(mouseX, mouseY);
      stroke(cCol);
      strokeWeight(2);
      noFill();
      ellipse(0, 0, 20, 20);
      line(-12, 0, -6, 0);
      line(6, 0, 12, 0);
      line(0, -12, 0, -6);
      line(0, 6, 0, 12);
      if (now - player.lastShot < player.shotCD && !player.jam) {
        stroke(prog >= 0.95 ? [255, 255, 0] : [0, 255, 0]);
        strokeWeight(4);
        arc(0, 0, 20, 20, -HALF_PI, -HALF_PI + prog * TWO_PI);
      }
      pop();
      let cPos = player.pos.copy().add(p5.Vector.fromAngle(player.angle + random(-0.1, 0.1)).mult(dist(player.pos.x, player.pos.y, wPos.x, wPos.y)));
      let csPos = cPos.sub(camPos).mult(zoom).add(width / 2, height / 2);
      push();
      translate(csPos.x, csPos.y);
      stroke(255);
      strokeWeight(1);
      noFill();
      ellipse(0, 0, 10, 10);
      line(-6, 0, 6, 0);
      line(0, -6, 0, 6);
      pop();
    }
    push();
    translate(width - 110, 10);
    fill(200, 220, 170, 200);
    rectMode(CORNER);
    rect(0, 0, 100, 100);
    let mapS = 100 / worldW;
    push();
    scale(mapS);
    stroke(255, 0, 0);
    strokeWeight(1 / mapS);
    noFill();
    ellipse(player.pos.x, player.pos.y, range * 2, range * 2);
    noStroke();
    obstacles.forEach(o => {
      fill(150);
      rectMode(CENTER);
      rect(o.pos.x, o.pos.y, o.w, o.h);
    });
    stroke(255);
    strokeWeight(1 / mapS);
    rect(camPos.x, camPos.y, width / zoom, height / zoom);
    pop();
    noStroke();
    [player, ...teammates].forEach(v => {
      push();
      translate(v.pos.x, v.pos.y);
      rotate(v.angle);
      fill(v === player ? [0, 255, 0] : [0, 0, 255]);
      if (v instanceof Tank) {
        ellipse(0, 0, 4 / mapS, 4 / mapS);
        rect(2 / mapS, 0, 2.25 / mapS, 1 / mapS);
      } else {
        triangle(-2 / mapS, 0, 5 / mapS, 0, -2 / mapS, 2 / mapS);
      }
      pop();
    });
    enemies.forEach(e => {
      push();
      translate(e.pos.x, e.pos.y);
      rotate(e.angle);
      fill(255, 0, 0);
      if (e instanceof Tank) {
        ellipse(0, 0, 4 / mapS, 4 / mapS);
        rect(2 / mapS, 0, 2.25 / mapS, 1 / mapS);
      } else {
        triangle(-2 / mapS, 0, 5 / mapS, 0, -2 / mapS, 2 / mapS);
      }
      pop();
    });
    pop();
    if (vType === 'airplane' && player.edgeWarn !== -1) {
      push();
      resetMatrix();
      textAlign(CENTER);
      textSize(24);
      fill(255, 0, 0);
      text('Warning: Return to battlefield', width / 2, 30);
      textSize(20);
      text(`${ceil(20 - (millis() - player.edgeWarn) / 1000)} seconds`, width / 2, 60);
      pop();
    }
    push();
    resetMatrix();
    textAlign(LEFT);
    textSize(16);
    fill(255);
    if (vType === 'tank') {
      let spd = player.vel.mag() * 34.1297;
      let dot = player.vel.dot(p5.Vector.fromAngle(player.angle)) / (player.vel.mag() + 0.0001);
      let scl = lerp(35 / 117, 20 / 43.875, (dot + 1) / 2);
      text(`Speed: ${(spd * scl).toFixed(1)} mph`, 10, 30);
      text(`Health: ${floor(player.hp)}/${player.maxHP}`, 10, 70);
    } else {
      text(`Speed: ${(player.vel.mag() * 60.351).toFixed(1)} mph`, 10, 30);
      text(`Health: ${floor(player.hp)}/${player.maxHP}`, 10, 70);
      text(`Bombs: ${player.bombs}`, 10, 90);
    }
    let yOff = 110;
    messages = messages.filter(m => {
      if (millis() - m.time >= m.dur) return false;
      let a = map(millis() - m.time, m.dur - 1000, m.dur, 255, 0);
      let col = m.col ? [m.col[0], m.col[1], m.col[2], a] : m.type === 'damage' ? [0, 255, 0, a] : m.type === 'hit' ? [255, 0, 0, a] : [255, a];
      fill(col);
      let txt = m.type === 'damage' || m.type === 'hit' ? `${m.type.charAt(0).toUpperCase() + m.type.slice(1)}: ${m.dmg} ${m.dtls.map(d => `${d.dmg} to ${d.zone} (${d.weapon})`).join(', ')}` : m.text;
      wrapGameText(txt, width / 2).forEach(l => { text(l, 10, yOff); yOff += 40; });
      return true;
    });
    pop();
  }

  function keyPressed() {
    if (keyCode === 67) {
      cycleIdx = (cycleIdx + 1) % cycleList.length;
      camTgt = cycleList[cycleIdx];
      camOff.set(0, 0);
      camPos.set(camTgt.pos);
    }
    if (vType === 'tank' && player instanceof Tank) {
      if (keyCode === 69 && millis() - player.extUsed >= 30000 && player.fire === 'burning') {
        player.fire = 'none';
        player.extUsed = millis();
        messages.push({ text: 'Fire extinguished', time: millis(), dur: 5000 });
      }
      if (keyCode === 72) player.shell = 'HE';
      if (keyCode === 65) player.shell = 'AP';
    }
  }

  function mousePressed() {
    if (vType === 'tank' && player instanceof Tank && !player.noFire) {
      let now = millis();
      if (now - player.lastShot >= player.shotCD) {
        let dir = p5.Vector.fromAngle(player.angle + player.turretAngle);
        bullets.push(new Bullet(
          player.pos.copy().add(p5.Vector.fromAngle(player.angle).mult(10)).add(dir.mult(16)),
          dir,
          'tank',
          'player',
          player.shell
        ));
        player.lastShot = now;
      }
    } else if (vType === 'airplane' && player instanceof Airplane && player.bombs > 0) {
      let now = millis();
      if (now - player.lastBomb > player.bombCD) {
        bombs.push(new Bomb(player.pos.copy(), player.angle, 'player'));
        player.bombs--;
        player.lastBomb = now;
      }
    }
  }

  function mouseWheel(e) {
    tgtZoom = constrain(tgtZoom * (e.delta > 0 ? 0.95 : 1.05), maxZoom, minZoom);
    return false;
  }

  function windowResized() {
    resizeCanvas(windowWidth, windowHeight);
  }
</script>
</body>
</html>
