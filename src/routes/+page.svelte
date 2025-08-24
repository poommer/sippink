<script>
  import { onMount } from 'svelte';
  import { browser } from '$app/environment'; // สำหรับ SvelteKit
  
  // --------- ตั้งค่าทรัพยากร ----------
  const BASE_SRC = '/tshirt.png'; // ใส่ mockup เสื้อของแบรนด์
  const STICKERS = [
    { id: 's1', src: '/stickers/1.png', label: 'Star' },
    { id: 's2', src: '/stickers/2.png', label: 'Smile' },
    { id: 's3', src: '/stickers/3.png', label: 'Flame' }
  ];

  // --------- สเตจ/ผืนผ้าใบ ----------
  let stageEl;         // div ของสเตจ
  let baseImg = null;  // เปลี่ยนเป็น null ก่อน
  
  let stageW = 720;    // ปรับตามหน้าจอภายหลัง
  let stageH = 900;    // จะเซ็ตตามอัตราส่วนรูปเสื้อ

  // --------- สถานะสติ๊กเกอร์แต่ละชิ้น ----------
  // item: { id, src, x, y, scale, rot, w, h, img }
  let items = [];
  let selectedId = null;

  // --------- การโต้ตอบ ----------
  let action = null;   // 'drag' | 'scale' | 'rotate'
  let start = { x: 0, y: 0 };
  let startItem = null;
  let handleCorner = null; // มุมที่ใช้ scale

  // แปลงพิกัดจอ -> พิกัดสเตจ
  function toStageCoords(clientX, clientY) {
    const r = stageEl.getBoundingClientRect();
    const x = (clientX - r.left);
    const y = (clientY - r.top);
    return { x, y };
  }

  function addSticker(src) {
    if (!browser) return; // ป้องกัน SSR
    
    const img = new Image();
    img.src = src;
    img.crossOrigin = 'anonymous';
    img.onload = () => {
      const baseSize = Math.min(stageW, stageH) * 0.28; // ขนาดตั้งต้น
      const scale = baseSize / img.width;
      const w = img.width;
      const h = img.height;
      const id = crypto.randomUUID();
      const item = {
        id, src, img, w, h,
        x: stageW / 2,
        y: stageH * 0.5,
        scale,
        rot: 0
      };
      items = [...items, item];
      selectedId = id;
    };
  }

  function select(id) { selectedId = id; }

  function removeSelected() {
    if (!selectedId) return;
    items = items.filter(it => it.id !== selectedId);
    selectedId = null;
  }

  function bringForward() {
    if (!selectedId) return;
    const idx = items.findIndex(i => i.id === selectedId);
    if (idx < 0 || idx === items.length - 1) return;
    const arr = [...items];
    [arr[idx], arr[idx + 1]] = [arr[idx + 1], arr[idx]];
    items = arr;
  }

  function sendBackward() {
    if (!selectedId) return;
    const idx = items.findIndex(i => i.id === selectedId);
    if (idx <= 0) return;
    const arr = [...items];
    [arr[idx], arr[idx - 1]] = [arr[idx - 1], arr[idx]];
    items = arr;
  }

  // เริ่มลาก/หมุน/สเกล
  function pointerDown(e, id, type, corner = null) {
    e.stopPropagation();
    const it = items.find(i => i.id === id);
    if (!it) return;
    select(id);
    const p = e.touches ? e.touches[0] : e;
    const { x, y } = toStageCoords(p.clientX, p.clientY);
    start = { x, y };
    startItem = { ...it };
    action = type; // 'drag' | 'rotate' | 'scale'
    handleCorner = corner;
    window.addEventListener('pointermove', pointerMove);
    window.addEventListener('pointerup', pointerUp);
    window.addEventListener('touchmove', pointerMove, { passive: false });
    window.addEventListener('touchend', pointerUp);
  }

  function pointerMove(e) {
    if (!action || !selectedId) return;
    const p = e.touches ? e.touches[0] : e;
    if (e.touches) e.preventDefault();
    const { x, y } = toStageCoords(p.clientX, p.clientY);
    const dx = x - start.x;
    const dy = y - start.y;
    const i = items.findIndex(it => it.id === selectedId);
    if (i < 0) return;
    const it = items[i];

    if (action === 'drag') {
      it.x = startItem.x + dx;
      it.y = startItem.y + dy;
    } else if (action === 'rotate') {
      const cx = startItem.x, cy = startItem.y;
      const a0 = Math.atan2(start.y - cy, start.x - cx);
      const a1 = Math.atan2(y - cy, x - cx);
      it.rot = startItem.rot + (a1 - a0);
    } else if (action === 'scale') {
      // scale ตามระยะจากจุดศูนย์กลาง
      const cx = startItem.x, cy = startItem.y;
      const d0 = Math.hypot(start.x - cx, start.y - cy);
      const d1 = Math.hypot(x - cx, y - cy);
      const s = Math.max(0.05, startItem.scale * (d1 / d0));
      it.scale = s;
    }
    items = [...items]; // trigger update
  }

  function pointerUp() {
    action = null;
    startItem = null;
    handleCorner = null;
    window.removeEventListener('pointermove', pointerMove);
    window.removeEventListener('pointerup', pointerUp);
    window.removeEventListener('touchmove', pointerMove);
    window.removeEventListener('touchend', pointerUp);
  }

  // คลิกพื้นที่ว่าง = ยกเลิกเลือก
  function clearSelection() {
    selectedId = null;
  }

  // ดาวน์โหลดภาพรวม (Base + สติ๊กเกอร์ทั้งหมด)
  async function downloadPNG() {
    if (!browser || !baseImg) return;
    
    await baseReady();
    const dpr = 2; // เพิ่มความคม
    const canvas = document.createElement('canvas');
    const W = baseImg.naturalWidth;
    const H = baseImg.naturalHeight;
    canvas.width = W * dpr / 2;     // ปรับสมดุลไฟล์ไม่ใหญ่เกิน
    canvas.height = H * dpr / 2;
    const ctx = canvas.getContext('2d');

    // วาดเสื้อฐานพอดีผืน
    ctx.drawImage(baseImg, 0, 0, canvas.width, canvas.height);

    // วาดสติ๊กเกอร์ทีละชิ้นตามสเกลที่สัมพันธ์กับ stage
    const scaleX = canvas.width / stageW;
    const scaleY = canvas.height / stageH;

    for (const it of items) {
      const cx = it.x * scaleX;
      const cy = it.y * scaleY;
      ctx.save();
      ctx.translate(cx, cy);
      ctx.rotate(it.rot);
      ctx.scale(it.scale * scaleX, it.scale * scaleY);
      ctx.drawImage(it.img, -it.w / 2, -it.h / 2);
      ctx.restore();
    }

    const url = canvas.toDataURL('image/png');
    const a = document.createElement('a');
    a.href = url;
    a.download = 'mockup.png';
    a.click();
  }

  function baseReady() {
    return new Promise((res) => {
      if (!baseImg) return res();
      if (baseImg.complete && baseImg.naturalWidth) return res();
      baseImg.onload = () => res();
    });
  }

  function fitStage() {
    if (!browser || !baseImg) return;
    
    // ทำให้สเตจยืดหยุ่นตามขนาด mockup และหน้าจอ
    const maxW = Math.min(900, window.innerWidth - 320); // กันพื้นที่ให้ sidebar
    const ratio = baseImg.naturalHeight / baseImg.naturalWidth || 1.25;
    stageW = Math.max(320, maxW);
    stageH = stageW * ratio;
  }

  function initializeBaseImage() {
    if (!browser) return;
    
    baseImg = new Image();
    baseImg.src = BASE_SRC;
    baseImg.crossOrigin = 'anonymous';
    baseImg.onload = () => {
      fitStage();
    };
  }

  onMount(async () => {
    initializeBaseImage();
    await baseReady();
    fitStage();
    window.addEventListener('resize', fitStage);
    
    return () => {
      window.removeEventListener('resize', fitStage);
    };
  });

  // ลบด้วยคีย์บอร์ด
  function onKey(e) {
    if (e.key === 'Delete' || e.key === 'Backspace') {
      removeSelected();
    }
  }
</script>

<style>
  .wrap {
    display: grid;
    grid-template-columns: 1fr 280px;
    gap: 16px;
    padding: 16px;
  }
  .stage {
    position: relative;
    background: #f7f7f8;
    border: 1px solid #e5e7eb;
    border-radius: 12px;
    overflow: hidden;
    touch-action: none;
    user-select: none;
  }
  .base-img {
    width: 100%;
    height: 100%;
    object-fit: contain;
    pointer-events: none;
    display: block;
  }
  .layer {
    position: absolute;
    transform-origin: center center;
    cursor: move;
  }
  .box {
    position: absolute;
    border: 1.5px dashed rgba(0,0,0,.4);
    inset: -8px -8px -8px -8px;
    border-radius: 8px;
    pointer-events: none;
  }
  .handle {
    width: 14px; height: 14px;
    background: #fff;
    border: 1px solid #111;
    border-radius: 3px;
    position: absolute;
    transform: translate(-50%, -50%);
  }
  .handle.tl { top: -8px; left: -8px; }
  .handle.tr { top: -8px; right: -8px; transform: translate(50%,-50%); }
  .handle.bl { bottom: -8px; left: -8px; transform: translate(-50%,50%); }
  .handle.br { bottom: -8px; right: -8px; transform: translate(50%,50%); }
  .rotator {
    width: 16px; height: 16px; border: 1px solid #111; background:#fff; border-radius: 50%;
    position: absolute; top: -36px; left: 50%; transform: translate(-50%, 0);
  }

  .side {
    border: 1px solid #e5e7eb; border-radius: 12px; padding: 12px; background: #fff;
    display: flex; flex-direction: column; gap: 12px;
  }
  .sticker-grid {
    display: grid; grid-template-columns: repeat(2, 1fr); gap: 8px;
  }
  .sticker-btn {
    border: 1px solid #e5e7eb; border-radius: 10px; background:#fafafa; padding: 8px; cursor: pointer;
    display: flex; align-items: center; justify-content: center; aspect-ratio: 1/1; overflow: hidden;
  }
  .sticker-btn img { width: 100%; height: 100%; object-fit: contain; pointer-events: none; }
  .toolbar { display:flex; gap:8px; flex-wrap:wrap; }
  .btn {
    background:#111; color:#fff; border:none; border-radius:10px; padding:8px 12px; cursor:pointer;
  }
  .btn.ghost { background:#f3f4f6; color:#111; }
  .hint { font-size: 12px; color:#6b7280; }
</style>

{#if browser}
<div class="wrap" on:keydown={onKey} tabindex="0">
  <!-- สเตจ/ผืนผ้าใบ -->
  <div
    class="stage"
    bind:this={stageEl}
    style={`width:${stageW}px; height:${stageH}px`}
    on:click={clearSelection}
  >
    <!-- รูปเสื้อฐาน -->
    <img class="base-img" src={BASE_SRC} alt="t-shirt mockup" draggable="false" />

    <!-- วาดสติ๊กเกอร์เป็นชั้นๆ -->
    {#each items as it (it.id)}
      <!-- แปลงพิกัดสเตจ -> CSS -->
      <div
        class="layer"
        on:pointerdown={(e)=>pointerDown(e, it.id, 'drag')}
        on:touchstart={(e)=>pointerDown(e, it.id, 'drag')}
        on:click|stopPropagation={()=>select(it.id)}
        style={`left:0; top:0; transform: translate(${it.x}px, ${it.y}px) rotate(${it.rot}rad) translate(-50%, -50%)`}
      >
        <img
          src={it.src}
          alt="sticker"
          draggable="false"
          style={`width:${it.w * it.scale}px; height:${it.h * it.scale}px; pointer-events:none;`}
        />
        {#if selectedId === it.id}
          <div class="box"></div>
          <!-- มุม scale -->
          <div class="handle tl" on:pointerdown={(e)=>pointerDown(e, it.id, 'scale','tl')}></div>
          <div class="handle tr" on:pointerdown={(e)=>pointerDown(e, it.id, 'scale','tr')}></div>
          <div class="handle bl" on:pointerdown={(e)=>pointerDown(e, it.id, 'scale','bl')}></div>
          <div class="handle br" on:pointerdown={(e)=>pointerDown(e, it.id, 'scale','br')}></div>
          <!-- ปุ่มหมุน -->
          <div class="rotator" on:pointerdown={(e)=>pointerDown(e, it.id, 'rotate')}></div>
        {/if}
      </div>
    {/each}
  </div>

  <!-- แถบสติ๊กเกอร์ + เครื่องมือ -->
  <aside class="side">
    <div class="toolbar">
      <button class="btn" on:click={downloadPNG}>ดาวน์โหลด PNG</button>
      <button class="btn ghost" on:click={bringForward}>เลเยอร์ขึ้น</button>
      <button class="btn ghost" on:click={sendBackward}>เลเยอร์ลง</button>
      <button class="btn ghost" on:click={removeSelected}>ลบที่เลือก</button>
    </div>
    <div class="hint">
      คลิกสติ๊กเกอร์เพื่อย้าย • จับมุมเพื่อย่อ/ขยาย • จับวงกลมเพื่อหมุน • ปุ่มลบ = Delete
    </div>
    <h3>สติ๊กเกอร์</h3>
    <div class="sticker-grid">
      {#each STICKERS as s}
        <button class="sticker-btn" on:click={() => addSticker(s.src)} aria-label={s.label}>
          <img src={s.src} alt={s.label} />
        </button>
      {/each}
    </div>

    <h3>นำเข้าของคุณเอง</h3>
    <input type="file" accept="image/*" on:change={(e)=>{
      const file = e.currentTarget.files?.[0];
      if (!file) return;
      const url = URL.createObjectURL(file);
      addSticker(url);
    }} />
  </aside>
</div>
{:else}
<div style="padding: 2rem; text-align: center;">
  <p>กำลังโหลด...</p>
</div>
{/if}