const canvas = document.getElementById('orbCanvas');
const ctx = canvas.getContext('2d');

const state = {
  width: 0,
  height: 0,
  dpr: Math.min(window.devicePixelRatio || 1, 2),
  radius: 170,
  targetRadius: 170,
  volume: 0,
  smoothVolume: 0,
  listening: false,
  processing: false,
  speaking: false,
  phase: 0,
  particles: [],
  noise: 0,
  processTime: 0,
  recognition: null,
  audioContext: null,
  analyser: null,
  source: null,
};

let microphoneStream = null;
let processTimer = null;

function createParticles() {
  state.particles = [];
  const total = 220;

  for (let i = 0; i < total; i += 1) {
    state.particles.push({
      angle: Math.random() * Math.PI * 2,
      radialOffset: 20 + Math.random() * 120,
      size: 1 + Math.random() * 3.2,
      speed: 0.2 + Math.random() * 1.2,
      alpha: 0.2 + Math.random() * 0.9,
      colorIndex: i % 3,
    });
  }
}

function resizeCanvas() {
  state.width = window.innerWidth;
  state.height = window.innerHeight;
  state.dpr = Math.min(window.devicePixelRatio || 1, 2);

  canvas.width = state.width * state.dpr;
  canvas.height = state.height * state.dpr;
  canvas.style.width = `${state.width}px`;
  canvas.style.height = `${state.height}px`;

  ctx.setTransform(state.dpr, 0, 0, state.dpr, 0, 0);
}

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}

function setupAudio() {
  const AudioCtor = window.AudioContext || window.webkitAudioContext;
  if (!AudioCtor) return;

  state.audioContext = new AudioCtor();
  state.analyser = state.audioContext.createAnalyser();
  state.analyser.fftSize = 2048;
  state.analyser.smoothingTimeConstant = 0.82;

  if (microphoneStream) {
    state.source = state.audioContext.createMediaStreamSource(microphoneStream);
    state.source.connect(state.analyser);
  }
}

async function ensureMicrophone() {
  if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
    return false;
  }

  try {
    microphoneStream = await navigator.mediaDevices.getUserMedia({
      audio: {
        echoCancellation: true,
        noiseSuppression: true,
        autoGainControl: true,
      },
    });

    if (!state.audioContext) {
      setupAudio();
    } else if (state.source) {
      state.source.disconnect();
      state.source = null;
    }

    if (state.audioContext && microphoneStream) {
      state.source = state.audioContext.createMediaStreamSource(microphoneStream);
      state.source.connect(state.analyser);
    }

    return true;
  } catch (error) {
    return false;
  }
}

function getVolume() {
  if (!state.analyser) return 0;

  const data = new Uint8Array(state.analyser.fftSize);
  state.analyser.getByteFrequencyData(data);

  let sum = 0;
  for (let i = 0; i < data.length; i += 1) {
    sum += data[i];
  }

  const average = sum / data.length;
  return clamp(average / 255, 0, 1);
}

function buildResponse(input) {
  const lower = input.toLowerCase();

  if (lower.includes('hallo') || lower.includes('hello')) {
    return 'Hallo! Ich bin hier und höre zu. Wie kann ich dir helfen?';
  }

  if (lower.includes('check') || lower.includes('prüf') || lower.includes('kontrollier')) {
    return 'Alles klar, ich prüfe das jetzt für dich.';
  }

  if (lower.includes('start') || lower.includes('starten') || lower.includes('los')) {
    return 'Starten wir direkt. Ich setze die Aktion jetzt in Gang.';
  }

  if (lower.includes('wie geht') || lower.includes('wie geht es') || lower.includes('status')) {
    return 'Der Status ist gut. Die Kugel läuft stabil und wartet auf deinen nächsten Befehl.';
  }

  if (lower.includes('schließen') || lower.includes('beenden') || lower.includes('stop')) {
    return 'Okay, ich beende den aktuellen Prozess und bin wieder bereit.';
  }

  if (lower.includes('ja') || lower.includes('okay') || lower.includes('verstanden')) {
    return 'Verstanden. Ich mache es so, wie du es willst.';
  }

  if (lower.includes('musik') || lower.includes('song') || lower.includes('ton')) {
    return 'Ich stelle die Audio-Reaktion auf einen stärkeren Effekt und lasse die Kugel mitlaufen.';
  }

  return 'Verstanden. Ich starte jetzt die passende Aktion und halte die Kugel aktiv.';
}

function speakText(text) {
  if (!('speechSynthesis' in window)) {
    return;
  }

  window.speechSynthesis.cancel();

  const utterance = new SpeechSynthesisUtterance(text);
  utterance.lang = 'de-DE';
  utterance.rate = 0.96;
  utterance.pitch = 1.0;
  utterance.volume = 1;

  utterance.onstart = () => {
    state.speaking = true;
    state.processing = false;
    state.listening = false;
  };

  utterance.onend = () => {
    state.speaking = false;
    state.listening = true;
    if (state.recognition) {
      state.recognition.start();
    }
  };

  window.speechSynthesis.speak(utterance);
}

function beginProcessing(prompt) {
  state.listening = false;
  state.processing = true;
  state.speaking = false;
  state.processTime = 0;

  if (processTimer) {
    clearTimeout(processTimer);
  }

  processTimer = setTimeout(() => {
    const reply = buildResponse(prompt);
    state.processing = false;
    speakText(reply);
  }, 1500);
}

function initRecognition() {
  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
  if (!SpeechRecognition) {
    return;
  }

  state.recognition = new SpeechRecognition();
  state.recognition.lang = 'de-DE';
  state.recognition.continuous = true;
  state.recognition.interimResults = false;

  state.recognition.onresult = (event) => {
    const transcript = Array.from(event.results)
      .map((result) => result[0].transcript)
      .join(' ')
      .trim();

    if (!transcript) {
      return;
    }

    beginProcessing(transcript);
  };

  state.recognition.onerror = () => {
    state.listening = false;
  };

  state.recognition.onend = () => {
    if (state.listening && state.recognition) {
      state.recognition.start();
    }
  };

  state.listening = true;
  state.recognition.start();
}

async function activate() {
  await ensureMicrophone();
  if (!state.audioContext) {
    setupAudio();
  }
  initRecognition();
}

function drawBackground() {
  const gradient = ctx.createRadialGradient(
    state.width / 2,
    state.height / 2,
    10,
    state.width / 2,
    state.height / 2,
    Math.max(state.width, state.height) * 0.85,
  );
  gradient.addColorStop(0, 'rgba(15, 26, 48, 0.65)');
  gradient.addColorStop(0.4, 'rgba(6, 14, 24, 0.35)');
  gradient.addColorStop(1, 'rgba(0, 0, 0, 0)');

  ctx.fillStyle = gradient;
  ctx.fillRect(0, 0, state.width, state.height);
}

function drawOrb(time) {
  const centerX = state.width / 2;
  const centerY = state.height / 2;

  const amplitude = getVolume();
  state.smoothVolume += (amplitude - state.smoothVolume) * 0.12;

  if (state.processing) {
    state.targetRadius = 90 + Math.sin(time * 8) * 12;
  } else if (state.speaking) {
    state.targetRadius = 150 + Math.sin(time * 12) * 23 + state.smoothVolume * 75;
  } else if (state.listening) {
    state.targetRadius = 120 + state.smoothVolume * 180;
  } else {
    state.targetRadius = 130 + Math.sin(time * 3) * 10;
  }

  state.radius += (state.targetRadius - state.radius) * 0.12;

  const glowGradient = ctx.createRadialGradient(
    centerX,
    centerY,
    state.radius * 0.2,
    centerX,
    centerY,
    state.radius * 2.1,
  );

  glowGradient.addColorStop(0, 'rgba(120, 220, 255, 0.9)');
  glowGradient.addColorStop(0.28, 'rgba(96, 145, 255, 0.8)');
  glowGradient.addColorStop(0.58, 'rgba(160, 120, 255, 0.45)');
  glowGradient.addColorStop(1, 'rgba(0, 0, 0, 0)');

  ctx.fillStyle = glowGradient;
  ctx.beginPath();
  ctx.arc(centerX, centerY, state.radius * 2.15, 0, Math.PI * 2);
  ctx.fill();

  const coreGradient = ctx.createRadialGradient(
    centerX - state.radius * 0.28,
    centerY - state.radius * 0.26,
    0,
    centerX,
    centerY,
    state.radius,
  );
  coreGradient.addColorStop(0, 'rgba(255,255,255,0.98)');
  coreGradient.addColorStop(0.18, 'rgba(170, 240, 255, 0.9)');
  coreGradient.addColorStop(0.48, 'rgba(110, 200, 255, 0.9)');
  coreGradient.addColorStop(0.72, 'rgba(98, 120, 255, 0.7)');
  coreGradient.addColorStop(1, 'rgba(32, 25, 70, 0.16)');

  ctx.fillStyle = coreGradient;
  ctx.beginPath();
  ctx.arc(centerX, centerY, state.radius, 0, Math.PI * 2);
  ctx.fill();

  const orbitCount = 170;
  for (let i = 0; i < orbitCount; i += 1) {
    const p = state.particles[i];
    const orbit = state.radius * 0.95 + p.radialOffset + state.smoothVolume * 120;
    const angle = p.angle + time * (0.6 + p.speed * 0.8) + i * 0.03;

    const x = centerX + Math.cos(angle) * orbit;
    const y = centerY + Math.sin(angle) * orbit * 0.9;

    const fluctuation = 1 + Math.sin(time * 3 + i) * 0.6;
    const size = p.size * fluctuation * (1 + state.smoothVolume * 2.2);

    if (state.processing) {
      const loaderSpin = time * 2.2 + i * 0.15;
      const pulseRadius = state.radius * 0.5 + Math.sin(loaderSpin) * 15;
      const px = centerX + Math.cos(loaderSpin + i) * pulseRadius;
      const py = centerY + Math.sin(loaderSpin * 1.3 + i) * pulseRadius;
      drawParticle(px, py, size, p.alpha, i);
    } else {
      drawParticle(x, y, size, p.alpha, i);
    }
  }

  if (state.processing) {
    const ringRadius = state.radius * 1.3 + Math.sin(time * 10) * 10;
    ctx.beginPath();
    ctx.strokeStyle = 'rgba(125, 220, 255, 0.8)';
    ctx.lineWidth = 2;
    ctx.arc(centerX, centerY, ringRadius, -Math.PI / 2, -Math.PI / 2 + Math.PI * 2 * (0.35 + Math.sin(time * 2) * 0.2));
    ctx.stroke();

    ctx.beginPath();
    ctx.strokeStyle = 'rgba(255, 255, 255, 0.3)';
    ctx.lineWidth = 1.2;
    ctx.arc(centerX, centerY, ringRadius + 15, 0, Math.PI * 2);
    ctx.stroke();
  }
}

function drawParticle(x, y, size, alpha, index) {
  const colors = [
    'rgba(138, 233, 255, 1)',
    'rgba(153, 169, 255, 1)',
    'rgba(210, 150, 255, 1)',
  ];

  ctx.beginPath();
  ctx.fillStyle = colors[index % colors.length];
  ctx.globalAlpha = alpha;
  ctx.arc(x, y, size, 0, Math.PI * 2);
  ctx.fill();
  ctx.globalAlpha = 1;
}

function animate(time) {
  drawBackground();
  state.phase += 0.016;
  drawOrb(time * 0.001);
  requestAnimationFrame(animate);
}

window.addEventListener('resize', resizeCanvas);
window.addEventListener('pointerdown', async () => {
  if (!state.audioContext) {
    await ensureMicrophone();
  }

  if (state.recognition) {
    state.listening = true;
    try {
      state.recognition.start();
    } catch (error) {
      // Ignore repeated starts.
    }
    return;
  }

  activate();
}, { once: true });

resizeCanvas();
createParticles();
requestAnimationFrame(animate);
