# We'll create an updated HTML file that plays a short "eat" sound when the snake eats food.
# This code generates a short WAV tone (sine) in-memory, encodes it as base64, and embeds it into the HTML.
from pathlib import Path
import base64
import math
import struct

# Paths to images (from earlier in the session)
green_path = "/mnt/data/Generated Image September 05, 2025 - 11_48AM.png"
red_path = "/mnt/data/Generated Image September 05, 2025 - 11_56AM (1).png"
for p in [green_path, red_path]:
    if not Path(p).exists():
        raise FileNotFoundError(f"Required image not found: {p}")

def img_data_uri(path):
    data = Path(path).read_bytes()
    return "data:image/png;base64," + base64.b64encode(data).decode('ascii')

green_data = img_data_uri(green_path)
red_data = img_data_uri(red_path)

# Generate a short WAV (sine tone) for the eat sound
def make_sine_wav(freq=880.0, duration=0.12, sr=44100, amp=0.3):
    n_samples = int(sr * duration)
    samples = []
    # simple linear fade-in/out to avoid clicks
    fade_len = int(sr * 0.01)  # 10ms fade
    for i in range(n_samples):
        t = i / sr
        env = 1.0
        if i < fade_len:
            env = i / fade_len
        elif i > n_samples - fade_len:
            env = (n_samples - i) / fade_len
        v = amp * env * math.sin(2 * math.pi * freq * t)
        # 16-bit signed
        samples.append(int(max(-1.0, min(1.0, v)) * 32767))
    # WAV header (PCM 16-bit mono)
    byte_data = b''.join(struct.pack('<h', s) for s in samples)
    # RIFF header
    datasize = len(byte_data)
    fmt_chunk = struct.pack('<4sIHHIIHH', b'fmt ', 16, 1, 1, sr, sr*2, 2, 16)
    data_chunk = struct.pack('<4sI', b'data', datasize) + byte_data
    riff_chunk = struct.pack('<4sI4s', b'RIFF', 4 + (8 + 16) + (8 + datasize), b'WAVE')
    wav = riff_chunk + fmt_chunk + data_chunk
    return wav

wav_bytes = make_sine_wav(freq=880.0, duration=0.12, sr=44100, amp=0.35)
wav_b64 = base64.b64encode(wav_bytes).decode('ascii')
audio_data_uri = "data:audio/wav;base64," + wav_b64

# Build HTML with embedded audio and play on eat
html = f"""<!doctype html>
<html lang="zh-Hant">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>貪食蛇（含吃到音效）</title>
<style>
  body {{ font-family: system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial; display:flex; gap:20px; align-items:flex-start; padding:20px; background:#fff7f7; }}
  #gameArea {{ background:#fff; border:1px solid #e6bcbc; padding:12px; border-radius:8px; box-shadow:0 6px 18px rgba(0,0,0,0.04); }}
  canvas {{ display:block; background:#fff5f5; border-radius:6px; }}
  .controls {{ display:grid; gap:12px; margin-top:12px; align-items:center; }}
  .row {{ display:flex; gap:12px; justify-content:center; }}
  button {{ padding:10px 16px; border-radius:10px; border:1px solid #d99; background:#fff; cursor:pointer; user-select:none; font-size:16px; }}
  button:active {{ transform:translateY(1px); }}
  .dir {{ width:84px; height:84px; font-size:22px; border-radius:12px; display:flex; align-items:center; justify-content:center; }}
  .main-btn {{ padding:12px 18px; font-size:16px; border-radius:12px; }}
  #info {{ margin-left:12px; max-width:360px; }}
  h2{{ margin:0 0 8px 0; font-size:18px; }}
  @media (max-width:600px) {{ .dir {{ width:68px; height:68px; font-size:18px; }} .main-btn {{ padding:10px 14px; font-size:15px; }} }}
</style>
</head>
<body>
  <div id="gameArea">
    <canvas id="c" width="640" height="480"></canvas>
    <div class="controls" aria-label="控制按鈕">
      <div class="row" style="gap:10px;">
        <button id="startBtn" class="main-btn">開始</button>
        <button id="stopBtn" class="main-btn">停止</button>
        <button id="resetBtn" class="main-btn">重置</button>
      </div>
      <div class="row" style="margin-top:6px;">
        <button id="upBtn" class="dir" aria-label="上">上</button>
      </div>
      <div class="row" style="margin-bottom:6px;">
        <button id="leftBtn" class="dir" aria-label="左">左</button>
        <button id="downBtn" class="dir" aria-label="下">下</button>
        <button id="rightBtn" class="dir" aria-label="右">右</button>
      </div>
    </div>
  </div>

  <div id="info">
    <h2>說明</h2>
    <p>已加入「吃到」音效。請於播放前按「開始」或任一按鈕授權音訊播放（多數瀏覽器需先有使用者互動）。</p>
    <p id="score">分數：0</p>
    <p style="font-size:12px;color:#666">下載後直接在瀏覽器開啟即可。</p>
  </div>

<audio id="eatSound" src="{audio_data_uri}" preload="auto"></audio>

<script>
const headImgSrc = "{green_data}";
const foodImgSrc = "{red_data}";

const canvas = document.getElementById('c');
const ctx = canvas.getContext('2d');
const cell = 32;
const cols = Math.floor(canvas.width / cell);
const rows = Math.floor(canvas.height / cell);

let snake = [];
let dir = {{x:1,y:0}};
let nextDir = {{x:1,y:0}};
let food = {{x:0,y:0}};
let running = false;
let timer = null;
let speed = 180;
let score = 0;

const headImg = new Image(); headImg.src = headImgSrc;
const foodImg = new Image(); foodImg.src = foodImgSrc;
const eatSound = document.getElementById('eatSound');

function resetGame() {{
  snake = [
    {{x: Math.floor(cols/2)-1, y: Math.floor(rows/2)}},
    {{x: Math.floor(cols/2)-2, y: Math.floor(rows/2)}}
  ];
  dir = {{x:1,y:0}}; nextDir = {{x:1,y:0}};
  score = 0;
  placeFood();
  updateScore();
  draw();
  stopLoop();
}}

function startLoop() {{
  if (running) return;
  running = true;
  if (timer) clearInterval(timer);
  timer = setInterval(step, speed);
}}

function stopLoop() {{
  running = false;
  if (timer) clearInterval(timer);
  timer = null;
}}

function placeFood() {{
  let tries = 0;
  while (true) {{
    const x = Math.floor(Math.random()*cols);
    const y = Math.floor(Math.random()*rows);
    if (!snake.some(s=>s.x===x && s.y===y)) {{ food = {{x,y}}; break; }}
    tries++; if (tries>1000) break;
  }}
}}

function step() {{
  if ((nextDir.x !== -dir.x) || (nextDir.y !== -dir.y)) dir = {{...nextDir}};
  const newHead = {{x: snake[0].x + dir.x, y: snake[0].y + dir.y}};

  // collisions
  if (newHead.x < 0 || newHead.x >= cols || newHead.y < 0 || newHead.y >= rows) {{ gameOver(); return; }}
  if (snake.some(seg=>seg.x===newHead.x && seg.y===newHead.y)) {{ gameOver(); return; }}

  const ate = (newHead.x === food.x && newHead.y === food.y);

  snake.unshift(newHead);

  if (ate) {{
    score += 1; updateScore();
    // play sound (reset to 0 so repeated plays work)
    try {{ eatSound.currentTime = 0; eatSound.play(); }} catch(e) {{ /* play may be blocked until user interacts */ }}
    placeFood();
  }} else {{
    snake.pop();
  }}

  draw();
}}

function gameOver() {{
  stopLoop();
  alert('遊戲結束！分數：' + score);
}}

function draw() {{
  ctx.clearRect(0,0,canvas.width,canvas.height);

  // grid
  ctx.strokeStyle = 'rgba(0,0,0,0.02)';
  for (let i=0;i<=cols;i++) {{ ctx.beginPath(); ctx.moveTo(i*cell,0); ctx.lineTo(i*cell,canvas.height); ctx.stroke(); }}
  for (let j=0;j<=rows;j++) {{ ctx.beginPath(); ctx.moveTo(0,j*cell); ctx.lineTo(canvas.width,j*cell); ctx.stroke(); }}

  // draw food
  if (foodImg.complete) ctx.drawImage(foodImg, food.x*cell, food.y*cell, cell, cell); else foodImg.onload = ()=> draw();

  // body
  for (let i=1;i<snake.length;i++) {{ 
    const s = snake[i];
    const x = s.x*cell + 4;
    const y = s.y*cell + 4;
    const size = cell - 8;
    ctx.fillStyle = '#ff4d4d';
    ctx.fillRect(x, y, size, size);
    ctx.strokeStyle = 'rgba(0,0,0,0.06)';
    ctx.strokeRect(x, y, size, size);
  }}

  // head
  if (headImg.complete) ctx.drawImage(headImg, snake[0].x*cell, snake[0].y*cell, cell, cell); else headImg.onload = ()=> draw();
}}

function setDirection(dx,dy) {{
  if (snake.length > 1 && dx === -dir.x && dy === -dir.y) return;
  nextDir = {{x:dx,y:dy}};
}}

// button listeners
document.getElementById('startBtn').addEventListener('click', ()=> startLoop());
document.getElementById('stopBtn').addEventListener('click', ()=> stopLoop());
document.getElementById('resetBtn').addEventListener('click', ()=> resetGame());
document.getElementById('upBtn').addEventListener('click', ()=> setDirection(0,-1));
document.getElementById('downBtn').addEventListener('click', ()=> setDirection(0,1));
document.getElementById('leftBtn').addEventListener('click', ()=> setDirection(-1,0));
document.getElementById('rightBtn').addEventListener('click', ()=> setDirection(1,0));

window.addEventListener('keydown', (e)=> {{ 
  if (['ArrowUp','ArrowDown','ArrowLeft','ArrowRight'].includes(e.key)) e.preventDefault();
  if (e.key === 'ArrowUp') setDirection(0,-1);
  if (e.key === 'ArrowDown') setDirection(0,1);
  if (e.key === 'ArrowLeft') setDirection(-1,0);
  if (e.key === 'ArrowRight') setDirection(1,0);
}});

function updateScore() {{ document.getElementById('score').textContent = '分數：' + score; }}

// init
resetGame();
</script>
</body>
</html>
"""

out_path = Path("/mnt/data/snake_game_with_sound.html")
out_path.write_text(html, encoding="utf-8")
out_path.exists(), str(out_path)
