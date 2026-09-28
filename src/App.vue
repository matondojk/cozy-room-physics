<template>
  <div class="app-container" :class="[currentTheme, isLampOn ? 'lamp-on' : 'lamp-off']"
       @mousedown="handleDown" @mousemove="handleMove" @mouseup="handleUp"
       @touchstart.prevent="handleDown" @touchmove.prevent="handleMove" @touchend="handleUp">

    <div id="sky">
        <div class="layer sky-night"></div>
        <div class="layer sky-afternoon"></div>
        <div class="layer sky-day"></div>
        <div id="stars-container">
            <div v-for="star in stars" :key="star.id" class="star" :style="star.style"></div>
        </div>
        <div id="celestial-body" @mousedown.stop="toggleTheme" @touchstart.stop="toggleTheme"></div>
    </div>

    <div id="ground">
        <div class="layer ground-night"></div>
        <div class="layer ground-afternoon"></div>
        <div class="layer ground-day"></div>
    </div>

    <div id="scene">
        <div class="house" ref="houseEl">
            <div class="roof-inner"></div>
            <div class="roof"></div>
            <div class="room" ref="roomEl">
                <div class="walls"></div>
                
                <div class="window">
                    <div class="layer window-night"></div>
                    <div class="layer window-afternoon"></div>
                    <div class="layer window-day"></div>
                </div>
                
                <div id="light-switch" @mousedown.stop="toggleLamp" @touchstart.stop="toggleLamp">
                    <div class="switch-button"></div>
                </div>

                <div class="picture"><div class="picture-art"></div></div>
                <div class="rug"></div>
                
                <div class="sofa">
                    <div class="sofa-base"></div>
                    <div class="sofa-back"></div>
                    <div class="sofa-cushion left"></div>
                    <div class="sofa-cushion right"></div>
                    <div class="sofa-arm left"></div>
                    <div class="sofa-arm right"></div>
                </div>

                <div class="plant">
                    <div class="leaf leaf1"></div>
                    <div class="leaf leaf2"></div>
                    <div class="leaf leaf3"></div>
                    <div class="pot"></div>
                </div>

                <div class="table">
                    <div class="table-leg left"></div>
                    <div class="table-leg right"></div>
                    <div class="table-top"></div>
                </div>

                <div class="floor"></div>
                
                <div class="lighting-overlay" ref="lightingEl"></div>
            </div>
        </div>
    </div>
    
    <div class="hint">Arraste a lâmpada, clique no céu ou no interruptor!</div>

    <canvas ref="lampCanvas"></canvas>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

const currentTheme = ref('night');
const isLampOn = ref(true);
const stars = ref([]);

// Generate stars
for (let i = 0; i < 150; i++) {
    const size = Math.random() * 2.5 + 0.5;
    stars.value.push({
        id: i,
        style: {
            left: Math.random() * 100 + '%',
            top: Math.random() * 100 + '%',
            width: size + 'px',
            height: size + 'px',
            animationDuration: (Math.random() * 3 + 2) + 's',
            animationDelay: (Math.random() * 5) + 's'
        }
    });
}

function toggleTheme() {
    if (currentTheme.value === 'night') currentTheme.value = 'day';
    else if (currentTheme.value === 'day') currentTheme.value = 'afternoon';
    else currentTheme.value = 'night';
}

function toggleLamp() {
    isLampOn.value = !isLampOn.value;
}

// Physics & Canvas Setup
const lampCanvas = ref(null);
const houseEl = ref(null);
const roomEl = ref(null);
const lightingEl = ref(null);

let ctx = null;
let animationFrameId = null;

let pivot = { x: 0, y: 0 };
let cableLength = 220;
let angle = 0.4;
let aVelocity = 0;
let aAcceleration = 0;
const gravity = 0.6;
const damping = 0.992; 

let isDragging = false;
let lampPos = { x: 0, y: 0 };
const lampHitbox = 60;

const themesColors = {
    night: {
        center: 'rgba(255, 245, 230, 1)', mid1: 'rgba(220, 195, 160, 1)', mid2: 'rgba(100, 80, 65, 1)',
        edge: 'rgba(30, 20, 20, 1)', ambient: 'rgba(5, 5, 5, 1)', ambientOff: 'rgba(2, 2, 2, 1)'
    },
    afternoon: {
        center: 'rgba(255, 245, 230, 1)', mid1: 'rgba(230, 210, 180, 1)', mid2: 'rgba(180, 130, 90, 1)',
        edge: 'rgba(120, 80, 60, 1)', ambient: 'rgba(90, 50, 40, 1)', ambientOff: 'rgba(70, 40, 30, 1)'
    },
    day: {
        center: 'rgba(255, 255, 255, 1)', mid1: 'rgba(255, 255, 255, 1)', mid2: 'rgba(230, 230, 230, 1)',
        edge: 'rgba(200, 200, 200, 1)', ambient: 'rgba(180, 180, 180, 1)', ambientOff: 'rgba(180, 180, 180, 1)'
    }
};

function updatePivot() {
    if (!houseEl.value) return;
    const rect = houseEl.value.getBoundingClientRect();
    pivot.x = rect.left + rect.width / 2;
    pivot.y = rect.top + 30;
    cableLength = rect.height * 0.40; 
}

function resize() {
    if (!lampCanvas.value) return;
    lampCanvas.value.width = window.innerWidth;
    lampCanvas.value.height = window.innerHeight;
    updatePivot();
}

function getMousePos(e) {
    if (e.touches && e.touches.length > 0) return { x: e.touches[0].clientX, y: e.touches[0].clientY };
    return { x: e.clientX, y: e.clientY };
}

function handleDown(e) {
    const pos = getMousePos(e);
    const dist = Math.sqrt((pos.x - lampPos.x)**2 + (pos.y - lampPos.y)**2);
    if (dist < lampHitbox) {
        isDragging = true; 
        aVelocity = 0;
        document.body.style.cursor = 'grabbing';
    }
}

function handleMove(e) {
    const pos = getMousePos(e);
    if (isDragging) {
        const dx = pos.x - pivot.x; 
        const dy = pos.y - pivot.y;
        angle = Math.atan2(dx, dy); 
        if (angle > Math.PI/2.5) angle = Math.PI/2.5;
        if (angle < -Math.PI/2.5) angle = -Math.PI/2.5;
    } else {
        const dist = Math.sqrt((pos.x - lampPos.x)**2 + (pos.y - lampPos.y)**2);
        document.body.style.cursor = dist < lampHitbox ? 'grab' : 'default';
    }
}

function handleUp() {
    if (isDragging) {
        isDragging = false;
        document.body.style.cursor = 'default';
    }
}

function drawLamp() {
    ctx.save();
    ctx.translate(lampPos.x, lampPos.y);
    ctx.rotate(-angle);

    if (isLampOn.value) {
        ctx.beginPath();
        ctx.arc(0, 22, 25, 0, Math.PI * 2);
        ctx.fillStyle = 'rgba(255, 230, 160, 0.4)';
        ctx.shadowColor = '#ffeaa8';
        ctx.shadowBlur = 45;
        ctx.fill(); ctx.fill();
    }

    ctx.shadowBlur = 0;
    ctx.beginPath(); ctx.moveTo(-18, -4); ctx.lineTo(18, -4); ctx.lineTo(38, 28); ctx.lineTo(-38, 28); ctx.closePath();
    ctx.fillStyle = '#1a1a1a'; ctx.fill();
    
    if (isLampOn.value) {
        ctx.beginPath(); ctx.moveTo(-33, 26); ctx.lineTo(33, 26); ctx.lineTo(13, -2); ctx.lineTo(-13, -2); ctx.closePath();
        ctx.fillStyle = 'rgba(255, 220, 120, 0.15)'; ctx.fill();
    }

    ctx.beginPath(); ctx.rect(-7, -12, 14, 8);
    ctx.fillStyle = '#0d0d0d'; ctx.fill();

    ctx.beginPath(); ctx.arc(0, 24, 11, 0, Math.PI * 2);
    ctx.fillStyle = isLampOn.value ? '#ffffff' : '#4a4a4a';
    if (isLampOn.value) { ctx.shadowColor = '#fff'; ctx.shadowBlur = 10; }
    ctx.fill();

    ctx.restore();
}

function animate() {
    if (!ctx) return;

    if (!isDragging) {
        aAcceleration = (-gravity / cableLength) * Math.sin(angle);
        aVelocity += aAcceleration; 
        aVelocity *= damping; 
        angle += aVelocity;
    }

    lampPos.x = pivot.x + cableLength * Math.sin(angle);
    lampPos.y = pivot.y + cableLength * Math.cos(angle);

    ctx.clearRect(0, 0, lampCanvas.value.width, lampCanvas.value.height);

    ctx.beginPath(); 
    ctx.moveTo(pivot.x, pivot.y); 
    ctx.lineTo(lampPos.x, lampPos.y);
    ctx.strokeStyle = '#050505'; 
    ctx.lineWidth = 3; 
    ctx.stroke();

    drawLamp();

    const t = themesColors[currentTheme.value];
    if (lightingEl.value) {
        if (isLampOn.value) {
            const roomRect = roomEl.value.getBoundingClientRect();
            const relX = lampPos.x - roomRect.left;
            const relY = lampPos.y - roomRect.top + 20; 
            
            lightingEl.value.style.background = `
                radial-gradient(circle at ${relX}px ${relY}px, 
                    ${t.center} 0%, ${t.mid1} 22%, ${t.mid2} 45%, ${t.edge} 80%, ${t.ambient} 100%
                )
            `;
        } else {
            lightingEl.value.style.background = t.ambientOff;
        }
    }

    animationFrameId = requestAnimationFrame(animate);
}

onMounted(() => {
    ctx = lampCanvas.value.getContext('2d');
    resize();
    window.addEventListener('resize', resize);
    animate();
});

onUnmounted(() => {
    window.removeEventListener('resize', resize);
    if (animationFrameId) cancelAnimationFrame(animationFrameId);
});

</script>

<style scoped>
.app-container {
    width: 100vw;
    height: 100vh;
    overflow: hidden;
    position: relative;
    background-color: #020111;
    transition: background 2s ease;
}

.layer { position: absolute; top: 0; left: 0; width: 100%; height: 100%; transition: opacity 2s ease; opacity: 0; z-index: 0; }

#sky { position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index: 0; }
.sky-night { background: linear-gradient(180deg, #010a15 0%, #07152b 50%, #0c2040 100%); }
.sky-afternoon { background: linear-gradient(180deg, #384273 0%, #a44f5c 40%, #d85c35 70%, #f2a65a 100%); }
.sky-day { background: linear-gradient(180deg, #5ebdf2 0%, #89cff0 50%, #b3e6ff 100%); }

.app-container.night .sky-night, .app-container.afternoon .sky-afternoon, .app-container.day .sky-day { opacity: 1; }

#stars-container { position: absolute; top: 0; left: 0; width: 100%; height: 100%; transition: opacity 2s ease; z-index: 1; pointer-events: none; }
.app-container.day #stars-container, .app-container.afternoon #stars-container { opacity: 0; }
.app-container.night #stars-container { opacity: 1; }

.star { position: absolute; background: #ffffff; border-radius: 50%; box-shadow: 0 0 5px #ffffff; animation: twinkle infinite ease-in-out; }
@keyframes twinkle { 0%, 100% { opacity: 0.2; transform: scale(0.8); } 50% { opacity: 1; transform: scale(1.2); } }

#celestial-body { position: absolute; border-radius: 50%; transition: all 2s ease; cursor: pointer; z-index: 10; pointer-events: auto; }
#celestial-body:hover { transform: scale(1.05); }

.app-container.night #celestial-body { top: 8%; right: 12%; width: 90px; height: 90px; background: #fffef2; box-shadow: 0 0 60px rgba(255, 254, 242, 0.6), 0 0 120px rgba(255, 254, 242, 0.3), inset -15px -15px 20px rgba(0,0,0,0.15); }
.app-container.afternoon #celestial-body { top: 50%; right: 25%; width: 100px; height: 100px; background: #ffcc66; box-shadow: 0 0 80px rgba(255, 170, 50, 0.8), 0 0 150px rgba(255, 170, 50, 0.4), inset -10px -10px 20px rgba(200,50,0,0.2); }
.app-container.day #celestial-body { top: 15%; right: 20%; width: 110px; height: 110px; background: #fffbdf; box-shadow: 0 0 80px rgba(255, 255, 200, 0.9), 0 0 150px rgba(255, 255, 200, 0.6), inset -5px -5px 10px rgba(200,100,0,0.1); }

#ground { position: absolute; bottom: 0; left: 0; width: 100%; height: 18vh; z-index: 1; border-top: 3px solid #111f11; overflow: hidden; }
.ground-night { background: linear-gradient(180deg, #071207 0%, #030503 100%); }
.ground-afternoon { background: linear-gradient(180deg, #1c1410 0%, #0d0a08 100%); border-top-color: #2b1812;}
.ground-day { background: linear-gradient(180deg, #478c3b 0%, #26521e 100%); border-top-color: #316327; }

.app-container.night .ground-night, .app-container.afternoon .ground-afternoon, .app-container.day .ground-day { opacity: 1; }

#scene { position: absolute; top: 0; left: 0; width: 100%; height: 100%; display: flex; justify-content: center; align-items: flex-end; z-index: 2; pointer-events: none; }
.house { position: relative; width: 80vw; max-width: 900px; height: 65vh; max-height: 550px; bottom: 18vh; background-color: #24222b; border: 24px solid #17161a; border-bottom: none; display: flex; justify-content: center; align-items: flex-end; box-shadow: 0 40px 80px rgba(0,0,0,0.9); }
.roof { position: absolute; top: -160px; left: -5%; width: 110%; height: 160px; background: #111; clip-path: polygon(50% 0%, 0% 100%, 100% 100%); border-bottom: 24px solid #0a0a0c; }
.roof-inner { position: absolute; top: -136px; left: 0%; width: 100%; height: 136px; background: #1a1a1f; clip-path: polygon(50% 0%, 5% 100%, 95% 100%); z-index: -1; }
.room { position: relative; width: 100%; height: 100%; background: #39323c; overflow: hidden; }

.walls { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background: linear-gradient(90deg, rgba(0,0,0,0.5) 0%, transparent 15%, transparent 85%, rgba(0,0,0,0.5) 100%), linear-gradient(180deg, rgba(0,0,0,0.7) 0%, transparent 40%, transparent 100%), #413946; }
.floor { position: absolute; bottom: 0; left: 0; width: 100%; height: 16%; background: repeating-linear-gradient(90deg, #422d1e, #422d1e 45px, #362417 45px, #362417 48px); border-top: 6px solid #23160c; box-shadow: inset 0 15px 30px rgba(0,0,0,0.6); }

#light-switch { position: absolute; top: 45%; left: 30px; width: 22px; height: 38px; background: #c0b8b5; border-radius: 3px; box-shadow: 2px 2px 5px rgba(0,0,0,0.5), inset -1px -1px 3px rgba(255,255,255,0.2); z-index: 25; display: flex; justify-content: center; align-items: center; pointer-events: auto; cursor: pointer; }
.switch-button { width: 8px; height: 16px; background: #e8e8e8; border-radius: 2px; box-shadow: 0 3px 2px rgba(0,0,0,0.4); transition: transform 0.15s ease, box-shadow 0.15s ease; pointer-events: none; }
.app-container.lamp-off .switch-button { transform: translateY(6px); box-shadow: 0 -3px 2px rgba(0,0,0,0.4); }
.app-container.lamp-on .switch-button { transform: translateY(-6px); box-shadow: 0 3px 2px rgba(0,0,0,0.4); }

.window { position: absolute; top: 15%; left: 8%; width: 22%; height: 45%; border: 8px solid #282830; box-shadow: inset 0 0 30px rgba(0,0,0,0.9), 8px 8px 20px rgba(0,0,0,0.5); overflow: hidden; }
.window-night { background: linear-gradient(to bottom, #07152b, #0c2040); }
.window-afternoon { background: linear-gradient(to bottom, #a44f5c, #f2a65a); }
.window-day { background: linear-gradient(to bottom, #5ebdf2, #89cff0); }
.app-container.night .window-night, .app-container.afternoon .window-afternoon, .app-container.day .window-day { opacity: 1; }
.window::before { content: ''; position: absolute; left: 50%; top: 0; width: 6px; height: 100%; background: #282830; transform: translateX(-50%); z-index: 5; }
.window::after { content: ''; position: absolute; top: 50%; left: 0; width: 100%; height: 6px; background: #282830; transform: translateY(-50%); z-index: 5; }

.picture { position: absolute; top: 18%; right: 12%; width: 16%; height: 26%; background: #111; border: 6px solid #4a3320; box-shadow: 6px 6px 15px rgba(0,0,0,0.6); }
.picture-art { width: 100%; height: 100%; background: linear-gradient(135deg, #d36b44, #36486b); }
.rug { position: absolute; bottom: 15%; left: 22%; width: 56%; height: 25%; background: repeating-radial-gradient(circle at 50% 50%, #7d3737, #7d3737 12px, #692d2d 12px, #692d2d 24px); border-radius: 50%; transform: scaleY(0.35); transform-origin: bottom center; box-shadow: 0 15px 25px rgba(0,0,0,0.8); z-index: 5; }
.sofa { position: absolute; bottom: 16%; right: 10%; width: 35%; height: 32%; z-index: 10; }
.sofa-base { position: absolute; bottom: 0; left: 5%; width: 90%; height: 42%; background: #2a3c48; border-radius: 8px; box-shadow: inset 0 10px 10px rgba(255,255,255,0.05), 0 10px 20px rgba(0,0,0,0.7); }
.sofa-back { position: absolute; bottom: 40%; left: 5%; width: 90%; height: 50%; background: #344857; border-radius: 12px 12px 0 0; box-shadow: inset 0 15px 15px rgba(255,255,255,0.05); }
.sofa-cushion { position: absolute; bottom: 40%; width: 44%; height: 26%; background: #3f5566; border-radius: 8px; box-shadow: inset 0 5px 5px rgba(255,255,255,0.1), 0 -2px 6px rgba(0,0,0,0.4); }
.sofa-cushion.left { left: 5%; } .sofa-cushion.right { right: 5%; }
.sofa-arm { position: absolute; bottom: 0; width: 15%; height: 65%; background: #1f2b33; border-radius: 8px; box-shadow: inset 0 5px 10px rgba(255,255,255,0.05); }
.sofa-arm.left { left: 0; } .sofa-arm.right { right: 0; }
.table { position: absolute; bottom: 16%; left: 42%; width: 22%; height: 18%; z-index: 15; }
.table-top { position: absolute; top: 0; left: 0; width: 100%; height: 18%; background: #94684a; border-radius: 50%; box-shadow: inset 0 2px 5px rgba(255,255,255,0.15), 0 8px 12px rgba(0,0,0,0.6); transform: scaleY(0.45); }
.table-leg { position: absolute; top: 10%; width: 8%; height: 90%; background: #523420; box-shadow: inset 3px 0 6px rgba(0,0,0,0.6); border-radius: 0 0 5px 5px; }
.table-leg.left { left: 20%; } .table-leg.right { right: 20%; }
.plant { position: absolute; bottom: 16%; left: 32%; width: 12%; height: 38%; z-index: 20; }
.pot { position: absolute; bottom: 0; left: 20%; width: 60%; height: 32%; background: linear-gradient(90deg, #783e25, #ad5f3a, #783e25); border-radius: 4px 4px 16px 16px; box-shadow: 6px 6px 15px rgba(0,0,0,0.6); }
.leaf { position: absolute; background: #2a5c24; border-radius: 50% 0 50% 0; box-shadow: inset 2px 2px 5px rgba(255,255,255,0.15); }
.leaf1 { bottom: 25%; left: 30%; width: 55%; height: 50%; transform: rotate(-25deg); }
.leaf2 { bottom: 30%; left: 10%; width: 65%; height: 45%; transform: rotate(-50deg); background: #32692b;}
.leaf3 { bottom: 35%; left: 45%; width: 50%; height: 55%; transform: rotate(15deg); }

.lighting-overlay { position: absolute; top: 0; left: 0; width: 100%; height: 100%; mix-blend-mode: multiply; z-index: 30; pointer-events: none; transition: background 0.1s; }
#lampCanvas { position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index: 100; pointer-events: none; }

.hint { position: absolute; top: 20px; width: 100%; text-align: center; color: rgba(255,255,255,0.8); font-size: 15px; letter-spacing: 3px; font-weight: 300; pointer-events: none; z-index: 200; animation: pulseHint 2s infinite alternate, fadeHint 6s forwards; animation-delay: 0s, 3s; opacity: 1; text-transform: uppercase; }
@keyframes pulseHint { 0% { transform: scale(1); } 100% { transform: scale(1.05); } }
@keyframes fadeHint { 80% { opacity: 1; } 100% { opacity: 0; display: none; } }
</style>
