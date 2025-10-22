<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <title>Logo Water Reflection Loop</title>
  <style>
    html,body { height:100%; margin:0; background: transparent; display:flex; align-items:center; justify-content:center; }
    canvas { background: transparent; display:block; }
  </style>
</head>
<body>
  <canvas id="c"></canvas>
  <script>
  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d', { alpha: true });

  // Settings
  const WIDTH = 1200;
  const HEIGHT = 1200;
  canvas.width = WIDTH;
  canvas.height = HEIGHT;

  // Visual layout
  const logoY = HEIGHT * 0.32;      // top position for logo
  const maxReflectionHeight = HEIGHT * 0.45;

  const logo = new Image();
  logo.src = 'logo-purple.png'; // place this file in same folder

  // Create a repeatable noise/ripple pattern for displacement (offscreen)
  const noiseCanvas = document.createElement('canvas');
  noiseCanvas.width = 512;
  noiseCanvas.height = 512;
  const nctx = noiseCanvas.getContext('2d');
  // generate subtle grayscale noise to use with sin waves
  function generateNoise() {
    const img = nctx.createImageData(noiseCanvas.width, noiseCanvas.height);
    for (let i = 0; i < img.data.length; i += 4) {
      const v = 128 + Math.floor(48 * Math.sin((i/4)%37) * Math.cos((i/4)%53)); // deterministic but textured
      img.data[i] = img.data[i+1] = img.data[i+2] = v;
      img.data[i+3] = 255;
    }
    nctx.putImageData(img, 0, 0);
  }
  generateNoise();

  let t0 = performance.now();

  function drawFrame(now) {
    const t = (now - t0) / 1000; // seconds
    ctx.clearRect(0,0,WIDTH,HEIGHT);

    // subtle background transparent canvas (so exported with alpha)
    // Draw original logo centered horizontally
    const scale = Math.min(WIDTH * 0.45 / logo.width, (HEIGHT*0.45) / logo.height);
    const lw = logo.width * scale;
    const lh = logo.height * scale;
    const lx = (WIDTH - lw)/2;
    const ly = logoY;

    // soft shadow under logo
    ctx.save();
    ctx.fillStyle = 'rgba(0,0,0,0.12)';
    ctx.filter = 'blur(28px)';
    ctx.fillRect(lx + lw*0.05, ly + lh - 12, lw*0.9, 12);
    ctx.restore();

    // Draw the logo (solid)
    ctx.drawImage(logo, lx, ly, lw, lh);

    // --- Reflection with ripple displacement ---
    // Create an offscreen buffer of the logo flipped vertically
    const buffer = document.createElement('canvas');
    buffer.width = lw;
    buffer.height = Math.min(lh, maxReflectionHeight);
    const bctx = buffer.getContext('2d', { alpha: true });
    bctx.save();
    // Draw flipped logo
    bctx.translate(0, buffer.height);
    bctx.scale(1, -1);
    // clip so reflected height can be shorter than the original
    bctx.drawImage(logo, 0, 0, lw, lh, 0, 0, lw, lh);
    bctx.restore();

    // Create displacement for ripple using composite trick
    // We'll create a displacement map on another canvas
    const disp = document.createElement('canvas');
    disp.width = buffer.width;
    disp.height = buffer.height;
    const dctx = disp.getContext('2d');

    // Fill with moving sine-based waves + subtle noise
    dctx.clearRect(0,0,disp.width,disp.height);
    dctx.drawImage(noiseCanvas, 0, 0, disp.width, disp.height);

    // overlay animated sine waves
    dctx.globalCompositeOperation = 'lighter';
    const waveCount = 3;
    for (let i=0;i<waveCount;i++){
      const amp = 6 + i*3;
      const freq = 0.8 + i*0.5;
      const phase = t * (0.6 + i*0.4);
      dctx.strokeStyle = `rgba(220,220,255,${0.04 + i*0.02})`;
      dctx.lineWidth = 2;
      dctx.beginPath();
      for (let x = 0; x < disp.width; x += 2) {
        const y = Math.floor(disp.height/2 + Math.sin((x / disp.width * Math.PI * freq) + phase) * amp);
        if (x === 0) dctx.moveTo(x,y); else dctx.lineTo(x,y);
      }
      dctx.stroke();
    }
    dctx.globalCompositeOperation = 'source-over';

    // Now build the actual reflected image by sampling lines with offset from disp
    // We'll copy horizontal strips from the buffer, offset horizontally by disp brightness
    const src = bctx.getImageData(0,0,buffer.width,buffer.height);
    const map = dctx.getImageData(0,0,disp.width,disp.height);
    const out = ctx;
    // Draw each horizontal strip with offset
    for (let y = 0; y < buffer.height; y++) {
      // sample displacement from map: use red channel
      const idx = (y * disp.width + Math.floor((Math.sin(t*0.6 + y*0.02) * 0.5 + 0.5) * (disp.width-1))) * 4;
      const brightness = map.data[idx]; // 0..255
      const shift = (brightness - 128)/128 * 12; // shift range -12..12 px
      // create image strip
      const strip = bctx.getImageData(0, y, buffer.width, 1);
      const pxX = Math.round(lx + shift);
      out.putImageData(strip, pxX, ly + lh + y);
    }

    // Apply an overall fading mask to reflection (fade with distance)
    ctx.save();
    const grad = ctx.createLinearGradient(0, ly+lh, 0, ly+lh + buffer.height);
    grad.addColorStop(0, 'rgba(255,255,255,0.35)'); // preserve some clarity near logo
    grad.addColorStop(1, 'rgba(255,255,255,0.02)'); // fade out
    ctx.fillStyle = grad;
    ctx.globalCompositeOperation = 'destination-out';
    ctx.fillRect(lx, ly + lh, lw, buffer.height);
    ctx.restore();

    // subtle moving highlight (gloss) across water
    ctx.save();
    ctx.globalCompositeOperation = 'screen';
    const glossW = lw * 0.8;
    const glossX = lx + (Math.sin(t*0.9)*0.5 + 0.5) * (lw - glossW);
    const glossY = ly + lh + buffer.height*0.25;
    const g = ctx.createLinearGradient(glossX, glossY, glossX + glossW, glossY);
    g.addColorStop(0, 'rgba(255,255,255,0.06)');
    g.addColorStop(0.5, 'rgba(255,255,255,0.14)');
    g.addColorStop(1, 'rgba(255,255,255,0.06)');
    ctx.fillStyle = g;
    ctx.fillRect(glossX, glossY, glossW, buffer.height * 0.7);
    ctx.restore();

    // small gentle wave-ripples overlay near bottom for texture
    ctx.save();
    ctx.globalAlpha = 0.28;
    ctx.strokeStyle = 'rgba(255,255,255,0.06)';
    for (let r = 0; r < 6; r++){
      ctx.beginPath();
      const offset = (t*30 + r*20) % (WIDTH+200) - 100;
      for (let x = -50; x < WIDTH+50; x += 8) {
        const y = ly + lh + buffer.height*0.32 + Math.sin((x+offset) * 0.02 + r)* (2 + r*0.3);
        if (x === -50) ctx.moveTo(x, y); else ctx.lineTo(x,y);
      }
      ctx.stroke();
    }
    ctx.restore();

    // Optional: Draw a subtle vignette or darken edges to focus on logo (keeps transparency center)
    // Not applied because export should keep clean transparency.

    requestAnimationFrame(drawFrame);
  }

  logo.onload = () => {
    t0 = performance.now();
    requestAnimationFrame(drawFrame);
  };
  </script>
</body>
</html>
