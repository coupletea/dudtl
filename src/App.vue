<!-- TwentyTenEnvelope.vue -->
<template>
  <main class="stage" :class="{ bright: isBright }" :style="{ '--bg-img': bgUrl ? `url('${bgUrl}')` : 'none' }">
    <div class="stars" ref="starsRef"></div>
    <div class="petals" ref="petalsRef"></div>
    <div class="balloon-layer" ref="balloonLayer"></div>
    <canvas id="fx" ref="canvasRef"></canvas>

    <section class="card3d tilt" ref="cardRef">
      <div class="envelope base" :class="{ open: isOpen }" ref="envRef">
        <div class="glow"></div>
        <div class="lid" ref="lidRef"></div>

        <div class="paper" ref="paperRef">
          <h1 class="title">Chúc mừng 20-10 <span class="heart"></span></h1>
          <p class="subtitle">
            <span class="ribbon">Gửi đến em, cô gái duy nhất</span>
          </p>

          <div class="inner" :class="{ show: isOpen }" ref="innerRef" aria-hidden="!isOpen">
            <p class="msg">
              <span class="type" :class="{ typing: typingCaret }">{{ typedText }}</span>
            </p>

            <p class="msg">
              Chúc em 20-10 vui vẻ, chúc em luôn xinh đẹp, mạnh mẽ, và bình an
              <br />
              <span class="signature">Thân gửi, người Cần Thơ</span>
            </p>

            <div class="row">
              <button class="btn violet" @click="clickConfetti">Pháo</button>
              <button class="btn mint" @click="shootHearts">tym</button>
            </div>
          </div>

          <div class="row">
            <button class="btn pink" @click="toggleOpen">{{ isOpen ? 'Đóng thiệp' : 'Mở thiệp' }}</button>
          </div>
        </div>
      </div>
    </section>

    <div class="spacer" :class="{ show: isOpen }"></div>
  </main>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, watch, nextTick } from "vue";

/* phần tử */
const starsRef = ref(null);
const petalsRef = ref(null);
const canvasRef = ref(null);
const cardRef = ref(null);
const envRef = ref(null);
const paperRef = ref(null);
const innerRef = ref(null);

/* trạng thái */
const isOpen = ref(false);
const isBright = ref(true);
const typedText = ref("");
const typingCaret = ref(false);
const bgUrl = ref("https://reviewcongnghe.net/wp-content/uploads/2024/04/hinh-nen-may-tinh-dep-thumb.jpg");
const balloonLayer = ref(null);

/* helpers */
const rand = (min, max) => Math.random() * (max - min) + min;

/* sao nền */
function buildStars() {
  const el = starsRef.value;
  if (!el) return;
  el.innerHTML = "";
  for (let i = 0; i < 120; i++) {
    const s = document.createElement("span");
    s.style.left = Math.random() * 100 + "vw";
    s.style.top = Math.random() * 100 + "vh";
    s.style.animationDelay = (Math.random() * 2.4).toFixed(2) + "s";
    el.appendChild(s);
  }
}

/* cánh hoa */
let petalTimer;
function spawnPetal() {
  const wrap = petalsRef.value;
  if (!wrap) return;
  const p = document.createElement("div");
  p.className = "petal";
  p.style.left = rand(0, 100) + "vw";
  const dur = rand(8, 16);
  p.style.animationDuration = dur + "s";
  p.style.setProperty("--tx", rand(-12, 12) + "vw");
  p.style.setProperty("--rot", rand(-360, 360) + "deg");
  p.style.opacity = rand(0.6, 1);
  wrap.appendChild(p);
  setTimeout(() => p.remove(), dur * 1000);
}
function startPetals() { petalTimer = setInterval(spawnPetal, 220); }
function stopPetals() { if (petalTimer) clearInterval(petalTimer); }

/* bóng bay */
function spawnBalloon() {
  const b = document.createElement("div");
  b.className = "balloon";
  b.style.left = rand(4, 96) + "vw";
  b.style.filter = `hue-rotate(${rand(0, 360)}deg)`;
  b.style.transform = `scale(${rand(0.8, 1.4)})`;
  b.style.animationDuration = rand(7, 13) + "s";
  // append vào layer trong component, không append ra document.body nữa
  balloonLayer.value?.appendChild(b);
  setTimeout(() => b.remove(), 13000);
}

/* confetti */
let ctx, W, H, lastTs, rafId;
const pieces = [];
function resizeCanvas() {
  const c = canvasRef.value;
  if (!c) return;
  W = c.width = window.innerWidth;
  H = c.height = window.innerHeight;
  ctx = c.getContext("2d");
}
function confettiBurst(x = W / 2, y = H / 3) {
  const n = 200;
  for (let i = 0; i < n; i++) {
    pieces.push({
      x, y,
      vx: Math.cos((i / n) * 2 * Math.PI) * rand(1, 6) + rand(-2, 2),
      vy: Math.sin((i / n) * 2 * Math.PI) * rand(1, 6) + rand(-2, 0),
      r: rand(2, 4.2),
      life: rand(1.2, 2.4),
      hue: rand(0, 360),
    });
  }
}
function loop(ts) {
  if (!ctx) { rafId = requestAnimationFrame(loop); return; }
  if (!lastTs) lastTs = ts;
  const dt = Math.min(0.033, (ts - lastTs) / 1000);
  lastTs = ts;
  ctx.clearRect(0, 0, W, H);
  for (let i = pieces.length - 1; i >= 0; i--) {
    const p = pieces[i];
    p.life -= dt;
    if (p.life <= 0) { pieces.splice(i, 1); continue; }
    p.vy += 9.8 * dt * 0.6;
    p.vx *= 0.995;
    p.x += p.vx * 60 * dt;
    p.y += p.vy * 60 * dt;
    ctx.save();
    ctx.translate(p.x, p.y);
    ctx.rotate((1 - p.life) * 8);
    ctx.fillStyle = `hsl(${p.hue} 90% 60%)`;
    ctx.fillRect(-p.r, -p.r, p.r * 2, p.r * 2);
    ctx.restore();
  }
  rafId = requestAnimationFrame(loop);
}

/* nghiêng theo chuột */
function bindTilt() {
  const card = cardRef.value;
  if (!card) return;
  const onMove = e => {
    const r = card.getBoundingClientRect();
    const rx = ((e.clientX - r.left) / r.width - 0.5) * 10;
    const ry = ((e.clientY - r.top) / r.height - 0.5) * -10;
    card.style.transform = `rotateY(${rx}deg) rotateX(${ry}deg)`;
  };
  const onLeave = () => { card.style.transform = "rotateY(0deg) rotateX(0deg)"; };
  card.addEventListener("mousemove", onMove);
  card.addEventListener("mouseleave", onLeave);
  return () => {
    card.removeEventListener("mousemove", onMove);
    card.removeEventListener("mouseleave", onLeave);
  };
}

/* gõ chữ */
const lines = [
  "Nhân ngày 20-10 😇",
  "Thì phải có quà cho em chứ nhỉ? 😊",
  "Hi vọng tấm thiệp này sẽ khiến em cười cuối ngày 😉",
];
let li = 0, ci = 0, typing = true;
let typeTimer = null;
let looping = false;

function startTyping(opts = {}) {
  const {
    loop = true,
    speed = 120,          // tốc độ gõ
    deleteSpeed = 50,    // tốc độ xóa
    pauseLine = 800,     // dừng ở cuối dòng, mili giây
    pauseCycle = 1200    // dừng khi hết các dòng, mili giây
  } = opts;

  stopTyping();                  // dọn lịch cũ
  looping = loop;
  typedText.value = "";
  li = 0; ci = 0; typing = true;
  typingCaret.value = true;

  const step = () => {
    if (li >= lines.length) {
      if (looping) {
        // nghỉ một nhịp rồi quay lại từ đầu
        typeTimer = setTimeout(() => {
          li = 0; ci = 0; typing = true;
          step();
        }, pauseCycle);
      } else {
        typingCaret.value = false;
      }
      return;
    }

    const line = lines[li];
    typedText.value = line.slice(0, ci);

    if (typing) {
      ci++;
      // tới cuối dòng, nghỉ rồi chuyển sang xóa
      if (ci > line.length) {
        typeTimer = setTimeout(() => {
          typing = false;
          step();
        }, pauseLine);
        return;
      }
      typeTimer = setTimeout(step, speed);
    } else {
      ci--;
      // xóa hết, sang dòng kế tiếp
      if (ci <= 0) {
        typing = true;
        li++;
      }
      typeTimer = setTimeout(step, deleteSpeed);
    }
  };

  step();
}

function stopTyping() {
  if (typeTimer) clearTimeout(typeTimer);
  typeTimer = null;
  looping = false;
  typingCaret.value = false;
}

/* hearts */
function shootHearts() {
  for (let i = 0; i < 100; i++) {
    setTimeout(() => {
      const h = document.createElement("div");
      h.className = "heartTiny";
      h.style.position = "fixed";
      h.style.left = rand(40, 60) + "vw";
      h.style.top = rand(44, 56) + "vh";
      const s = rand(10, 22);
      h.style.width = h.style.height = s + "px";
      h.style.filter = "drop-shadow(0 4px 10px rgba(255,111,179,.45))";
      h.style.zIndex = 5;
      h.style.pointerEvents = "none";
      h.style.transform = `translate(-50%,-50%) rotate(${rand(0, 360)}deg)`;
      h.style.background = "transparent";
      h.innerHTML =
          '<svg viewBox="0 0 32 29" width="100%" height="100%"><path fill="#ff6fb3" d="M23.6,0c-2.7,0-5.1,1.5-7.6,4.6C13.5,1.5,11.1,0,8.4,0C3.7,0,0,3.7,0,8.4c0,10.1,14.7,18.1,16,19 c1.3-0.9,16-8.9,16-19C32,3.7,28.3,0,23.6,0z"/></svg>';
      document.body.appendChild(h);
      const dx = rand(-32, 32), dy = rand(-24, -60);
      const anim = h.animate(
          [
            { transform: "translate(-50%,-50%) scale(0.4)", opacity: 1 },
            { transform: `translate(calc(-50% + ${dx}vw), calc(-50% + ${dy}vh)) scale(1.4)`, opacity: 0 },
          ],
          { duration: 900 + Math.random() * 600, easing: "cubic-bezier(.2,.7,0,1)" }
      );
      anim.onfinish = () => h.remove();
    }, i * 10);
  }
}

/* nút bấm */
function toggleOpen() { isOpen.value = !isOpen.value; }
function toggleBright(){ isBright.value = !isBright.value; }
function clickConfetti() {
  confettiBurst();
  for (let i = 0; i < 10; i++) setTimeout(spawnBalloon, i * 20);
}

/* khi mở, cuộn xuống và hiện nội dung, đồng thời bắt đầu gõ chữ, bắn confetti nhẹ */
watch(isOpen, async (val) => {
  if (val) {
    await nextTick();
    document.documentElement.style.scrollBehavior = "smooth";
    innerRef.value?.scrollIntoView({ behavior: "smooth", block: "center" });
    startTyping();
    setTimeout(() => confettiBurst(), 300);
  } else {
    stopTyping();
    typedText.value = "";
    document.documentElement.style.scrollBehavior = "";
  }
});

/* lifecycle */
let removeTilt, resizeHandler, clickHandler;
onMounted(() => {
  buildStars();
  startPetals();
  resizeCanvas();
  lastTs = performance.now();
  rafId = requestAnimationFrame(loop);

  const c = canvasRef.value;
  clickHandler = e => confettiBurst(e.clientX, e.clientY);
  c.addEventListener("click", clickHandler);

  removeTilt = bindTilt();

  resizeHandler = () => resizeCanvas();
  window.addEventListener("resize", resizeHandler);
});
onBeforeUnmount(() => {
  stopPetals();
  stopTyping();
  if (rafId) cancelAnimationFrame(rafId);
  const c = canvasRef.value;
  if (c && clickHandler) c.removeEventListener("click", clickHandler);
  if (removeTilt) removeTilt();
  if (resizeHandler) window.removeEventListener("resize", resizeHandler);
});
</script>

<style scoped>
:root{
  --bg:#0b1220;
  --pink:#ff6fb3;
  --pink-2:#ff95c8;
  --violet:#8a7dff;
  --mint:#7cf7c6;
  --white:#ffffff;
  --glass:rgba(255,255,255,.08);
  --shadow:0 10px 30px rgba(0,0,0,.45);
}

*{box-sizing:border-box}
html,body{height:100%}

/* nền có ảnh google, phủ gradient để dễ đọc chữ */
.stage{
  min-height:100dvh;
  display:grid;
  place-items:center;
  color:var(--white);
  background:
      linear-gradient(180deg, rgba(11,18,32,.8), rgba(11,18,32,.85)),
      var(--bg-img),
      radial-gradient(1200px 800px at 80% -10%, #162647 0%, rgba(22,38,71,0) 60%),
      radial-gradient(1200px 800px at 0% 110%, #261642 0%, rgba(38,22,66,0) 60%),
      linear-gradient(180deg, #0b1220 0%, #0b1220 100%);
  background-size: cover, cover, auto, auto, auto;
  background-position: center, center, center, center, center;
  overflow:visible;
  position:relative;
}

/* chế độ rực sáng, overlay và tinh chỉnh */
.stage.bright::before{
  content:"";
  position:fixed;
  inset:-10%;
  pointer-events:none;
  z-index:0;
  background:
      radial-gradient(60vmax 35vmax at 50% 15%, rgba(255,111,179,.28), transparent 60%),
      radial-gradient(55vmax 40vmax at 20% 80%, rgba(138,125,255,.22), transparent 65%),
      radial-gradient(65vmax 45vmax at 85% 70%, rgba(124,247,198,.22), transparent 70%);
  mix-blend-mode: screen;
  filter: blur(18px) saturate(115%);
  animation: glowPulse 4.6s ease-in-out infinite;
}
.stage.bright{ filter: brightness(1.12) contrast(1.05) saturate(1.08); }
.stage.bright .glow{ opacity:1; filter: blur(18px) saturate(120%); }
.stage.bright .paper{
  box-shadow: 0 12px 32px rgba(255,255,255,.14), 0 14px 36px rgba(0,0,0,.35);
  backdrop-filter: blur(10px);
}
.stage.bright .lid{ filter: brightness(1.08) saturate(1.12); }
.stage.bright .btn{ box-shadow: 0 6px 18px rgba(0,0,0,.28), inset 0 0 0 1px rgba(255,255,255,.9); }
.stage.bright .title{ text-shadow: 0 2px 0 rgba(255,255,255,.28), 0 0 36px rgba(255,111,179,.45); }

@keyframes glowPulse{
  0%{ opacity:.7; transform:scale(1); }
  50%{ opacity:1; transform:scale(1.03); }
  100%{ opacity:.7; transform:scale(1); }
}

/* stars */
.stars{position:fixed;inset:0;pointer-events:none;z-index:1}
.stars span{position:absolute;width:2px;height:2px;background:rgba(255,255,255,.9);border-radius:50%;box-shadow:0 0 6px 2px rgba(255,255,255,.6);animation: twinkle 2.4s ease-in-out infinite}
@keyframes twinkle{0%{opacity:.3;transform:scale(.7)}50%{opacity:1;transform:scale(1)}100%{opacity:.3;transform:scale(.7)}}

/* petals */
.petals{position:fixed;inset:0;pointer-events:none;z-index:2}
.petal{position:absolute;width:14px;height:18px;background:radial-gradient( circle at 30% 20%, #fff 0%, #ffd2e8 35%, #ff6fb3 70%, #e23b8b 100% );border-radius: 60% 60% 60% 60% / 80% 80% 30% 30%; filter:blur(.2px); opacity:.9; animation: fall linear infinite}
@keyframes fall{to{transform:translate3d(var(--tx), 110vh, 0) rotate(var(--rot)); opacity:.95}}

/* card */
.card3d{position:relative;perspective:1200px;width:min(92vw, 980px)}
.envelope{position:relative;height:420px}
.lid, .paper, .base{position:absolute;inset:0;border-radius:24px}
.base{background:linear-gradient(145deg, rgba(255,255,255,.1), rgba(255,255,255,.04)); backdrop-filter:blur(8px); border:1px solid rgba(255,255,255,.16); box-shadow:var(--shadow); transform-style:preserve-3d}
.lid{
  background:conic-gradient(from 230deg, var(--pink), var(--violet), var(--mint), var(--pink));
  -webkit-mask:radial-gradient(160% 100% at 50% 0, #000 60%, transparent 61%) top / 100% 60% no-repeat, linear-gradient(#000 0 0) bottom/100% 42% no-repeat;
  mask:radial-gradient(160% 100% at 50% 0, #000 60%, transparent 61%) top / 100% 60% no-repeat, linear-gradient(#000 0 0) bottom/100% 42% no-repeat;
  transform-origin:50% 30%; transform:rotateX(0deg); transition:transform 800ms cubic-bezier(.16,.8,.16,1)
}
.paper{padding:26px 26px 30px; transform:translateZ(2px)}
.glow{position:absolute;inset:-2px;border-radius:24px;filter:blur(16px);background: radial-gradient(600px 220px at 50% 10%, rgba(255,111,179,.55), rgba(138,125,255,.35), rgba(124,247,198,.25), transparent 70%); pointer-events:none;opacity:.85}

.title{margin:8px 0 10px;font-size: clamp(28px, 6vw, 52px); font-weight:800; line-height:1.05; letter-spacing:.4px; text-align:center; text-shadow:0 2px 0 rgba(255,255,255,.2), 0 0 30px rgba(255,111,179,.35)}
.subtitle{margin:-6px 0 18px; text-align:center; font-size: clamp(16px, 2.6vw, 20px); opacity:.9}
.ribbon{display:inline-block;padding:8px 14px;border-radius:999px; background:linear-gradient(90deg, var(--pink), var(--violet)); box-shadow:0 8px 22px rgba(255,111,179,.35); font-weight:700; letter-spacing:.4px}

/* nội dung bên trong */
.inner{
  max-height:0;
  opacity:0;
  overflow:hidden;
  pointer-events:none;
  transform: translateY(8px);
  transition: max-height .6s ease, opacity .4s ease, transform .4s ease;
}
.inner.show{
  max-height:1000px;
  opacity:1;
  pointer-events:auto;
  transform: translateY(0);
}

.msg{margin:14px auto 16px;max-width:680px; line-height:1.7; font-size:clamp(15px, 2.6vw, 18px); background:var(--glass); border:1px solid rgba(255,255,255,.14); padding:16px 18px;border-radius:16px}
.signature{margin-top:8px; font-style:italic; text-align:right; opacity:.9}

/* buttons */
.row{display:flex;gap:12px;flex-wrap:wrap;justify-content:center;margin:16px 0 6px}
.btn{cursor:pointer; border:none; padding:12px 16px; font-weight:700; letter-spacing:.3px;border-radius:14px; background:linear-gradient(180deg,#ffffff,#e9e9ff); color:#2a2a57; box-shadow:0 4px 14px rgba(0,0,0,.22), inset 0 0 0 1px rgba(255,255,255,.8); transition: transform .15s ease, box-shadow .15s ease}
.btn:hover{transform:translateY(-2px)}
.btn:active{transform:translateY(0); box-shadow:0 2px 10px rgba(0,0,0,.26)}
.btn.pink{background:linear-gradient(180deg, #ffe5f2, #ffd4ec); color:#8b2d5a}
.btn.violet{background:linear-gradient(180deg, #efeaff, #e4ddff); color:#37307e}
.btn.mint{background:linear-gradient(180deg, #e8fff7, #d6ffef); color:#16694f}

/* tim lớn */
.heart{
  display:inline-block;
  width:22px;height:22px;
  position:relative;
  transform:translateY(3px);
  filter:drop-shadow(0 4px 10px rgba(255,111,179,.4));
  animation: thump 1.2s ease-in-out infinite;
}
.heart:before,
.heart:after{
  content:"";
  position:absolute;
  width:12px;height:18px;
  background:var(--pink);
  left:10px; top:2px;
  border-radius:12px 12px 0 0;
  transform-origin:0 100%;
}
.heart:before{ transform:rotate(-45deg); }
.heart:after{ left:0; transform-origin:100% 100%; transform:rotate(45deg); }

@keyframes thump{
  0%{transform:scale(1) translateY(3px)}
  25%{transform:scale(1.08) translateY(3px)}
  50%{transform:scale(1) translateY(3px)}
  75%{transform:scale(1.08) translateY(3px)}
  100%{transform:scale(1) translateY(3px)}
}

/* tilt */
.tilt{transition: transform .2s ease; will-change:transform}

/* bóng bay */
.balloon-layer{position:fixed; inset:0; pointer-events:none; z-index:5}
.balloon{
  position:fixed; bottom:-120px; width:40px; height:52px;
  background:radial-gradient(circle at 50% 35%, #fff 0%, var(--pink-2) 30%, var(--pink) 70%);
  border-radius:50% 50% 50% 50% / 60% 60% 40% 40%;
  box-shadow:inset 0 -6px 10px rgba(0,0,0,.15);
  animation: floatUp linear forwards;
}
.balloon:after{
  content:""; position:absolute; left:50%; transform:translateX(-50%);
  bottom:-18px; width:2px; height:20px; background:#ffd5ea;
}
@keyframes floatUp{to{transform:translateY(-120vh)}}

/* canvas layer, spacer, tip */
#fx{position:fixed;inset:0;pointer-events:none;z-index:4}
.spacer{height:0; transition: height .4s ease}
.spacer.show{height:70vh}
.tip{position:fixed;right:16px;bottom:16px;background:rgba(0,0,0,.5);padding:8px 10px;border-radius:10px;font-size:13px;opacity:.85}

/* mở nắp */
.open .lid{transform:rotateX(-64deg)}

/* typewriter */
.type{display:inline-block; border-right:2px solid rgba(255,255,255,.7); white-space:nowrap; overflow:hidden}
.typing{animation: caret 1s steps(1) infinite}
@keyframes caret{50%{border-color:transparent}}

/* mobile */
@media (max-width:520px){
  .paper{padding:20px 16px 22px}
  .msg{padding:14px}
}
</style>
