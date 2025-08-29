<script>
  import { onMount } from 'svelte';
  import { browser } from '$app/environment'; // สำหรับ SvelteKit
  
  // --------- ตั้งค่าทรัพยากร ----------
  const BASE_SRC = '/mockups/tshirt.png'; // ใส่ mockup เสื้อของแบรนด์
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

  // --------- Undo/Redo System ----------
  let history = [];
  let historyIndex = -1;
  const maxHistory = 50;

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

  // บันทึกสถานะลง history
  function saveState() {
    const state = {
      items: items.map(item => ({
        id: item.id,
        src: item.src, 
        x: item.x,
        y: item.y,
        scale: item.scale,
        rot: item.rot,
        w: item.w,
        h: item.h
      })),
      selectedId
    };
    
    // ลบ history ที่อยู่หลัง current index (ถ้ามี)
    history = history.slice(0, historyIndex + 1);
    
    // เพิ่ม state ใหม่
    history.push(state);
    historyIndex = history.length - 1;
    
    // จำกัดขนาด history
    if (history.length > maxHistory) {
      history = history.slice(-maxHistory);
      historyIndex = history.length - 1;
    }
  }

  // คืนค่าสถานะจาก history
  async function restoreState(state) {
    if (!browser) return;
    
    const restoredItems = [];
    for (const itemData of state.items) {
      const img = new Image();
      img.src = itemData.src;
      img.crossOrigin = 'anonymous';
      
      await new Promise(resolve => {
        img.onload = resolve;
        if (img.complete) resolve();
      });
      
      restoredItems.push({
        ...itemData,
        img
      });
    }
    
    items = restoredItems;
    selectedId = state.selectedId;
  }

  function undo() {
    if (historyIndex > 0) {
      historyIndex--;
      restoreState(history[historyIndex]);
    }
  }

  function redo() {
    if (historyIndex < history.length - 1) {
      historyIndex++;
      restoreState(history[historyIndex]);
    }
  }

  $: canUndo = historyIndex > 0;
  $: canRedo = historyIndex < history.length - 1;

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
      saveState(); // บันทึกหลังเพิ่มสติ๊กเกอร์
    };
  }

  function select(id) { selectedId = id; }

  function removeSelected() {
    if (!selectedId) return;
    items = items.filter(it => it.id !== selectedId);
    selectedId = null;
    saveState(); // บันทึกหลังลบ
  }

  function bringForward() {
    if (!selectedId) return;
    const idx = items.findIndex(i => i.id === selectedId);
    if (idx < 0 || idx === items.length - 1) return;
    const arr = [...items];
    [arr[idx], arr[idx + 1]] = [arr[idx + 1], arr[idx]];
    items = arr;
    saveState(); // บันทึกหลังเปลี่ยนเลเยอร์
  }

  function sendBackward() {
    if (!selectedId) return;
    const idx = items.findIndex(i => i.id === selectedId);
    if (idx <= 0) return;
    const arr = [...items];
    [arr[idx], arr[idx - 1]] = [arr[idx - 1], arr[idx]];
    items = arr;
    saveState(); // บันทึกหลังเปลี่ยนเลเยอร์
  }

  // เริ่มลาก/หมุน/สเกล
  function pointerDown(e, id, type, corner = null) {
    e.stopPropagation();
    e.preventDefault(); // ป้องกัน scroll บนมือถือ
    
    const it = items.find(i => i.id === id);
    if (!it) return;
    select(id);
    
    // รองรับทั้ง mouse และ touch
    const p = e.touches?.[0] || e;
    const { x, y } = toStageCoords(p.clientX, p.clientY);
    start = { x, y };
    startItem = { ...it };
    action = type;
    handleCorner = corner;
    
    // เพิ่ม class สำหรับ smooth animation
    const layer = e.currentTarget.closest('.layer');
    if (layer) layer.classList.add('dragging');
    
    // รองรับทั้ง pointer และ touch events
    const moveHandler = (e) => pointerMove(e);
    const upHandler = (e) => pointerUp(e);
    
    window.addEventListener('pointermove', moveHandler, { passive: false });
    window.addEventListener('pointerup', upHandler);
    window.addEventListener('touchmove', moveHandler, { passive: false });
    window.addEventListener('touchend', upHandler);
    window.addEventListener('touchcancel', upHandler);
  }

  function pointerMove(e) {
    if (!action || !selectedId) return;
    
    e.preventDefault(); // ป้องกัน scroll/zoom บนมือถือ
    const p = e.touches?.[0] || e;
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
      let newRot = startItem.rot + (a1 - a0);
      
      // ทำให้หมุนสมูธขึ้นด้วยการ normalize มุม
      while (newRot > Math.PI) newRot -= 2 * Math.PI;
      while (newRot < -Math.PI) newRot += 2 * Math.PI;
      
      it.rot = newRot;
    } else if (action === 'scale') {
      // scale ตามระยะจากจุดศูนย์กลาง
      const cx = startItem.x, cy = startItem.y;
      const d0 = Math.hypot(start.x - cx, start.y - cy);
      const d1 = Math.hypot(x - cx, y - cy);
      if (d0 > 0) { // ป้องกันการหารด้วย 0
        const s = Math.max(0.1, Math.min(5, startItem.scale * (d1 / d0))); // จำกัดขนาด
        it.scale = s;
      }
    }
    items = [...items]; // trigger update
  }

  function pointerUp() {
    const wasMoving = action !== null;
    action = null;
    startItem = null;
    handleCorner = null;
    
    // ลบ dragging class
    const layers = document.querySelectorAll('.layer.dragging');
    layers.forEach(layer => layer.classList.remove('dragging'));
    
    window.removeEventListener('pointermove', pointerMove);
    window.removeEventListener('pointerup', pointerUp);
    window.removeEventListener('touchmove', pointerMove);
    window.removeEventListener('touchend', pointerUp);
    window.removeEventListener('touchcancel', pointerUp);
    
    // บันทึก state หลังจากย้าย/หมุน/สเกลเสร็จ
    if (wasMoving) {
      saveState();
    }
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
    const isMobile = window.innerWidth <= 768;
    const maxW = isMobile 
      ? window.innerWidth - 32  // mobile: full width - padding
      : Math.min(1200, window.innerWidth - 320); // desktop: กันพื้นที่ให้ sidebar
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
    
    // บันทึก initial state
    saveState();
    
    return () => {
      window.removeEventListener('resize', fitStage);
    };
  });

  // ลบด้วยคีย์บอร์ด
  function onKey(e) {
    if (e.key === 'Delete' || e.key === 'Backspace') {
      removeSelected();
    } else if ((e.ctrlKey || e.metaKey) && e.key === 'z' && !e.shiftKey) {
      e.preventDefault();
      undo();
    } else if ((e.ctrlKey || e.metaKey) && (e.key === 'y' || (e.key === 'z' && e.shiftKey))) {
      e.preventDefault();
      redo();
    }
  }
</script>

<style>

  .stage {
    position: relative;
    background: #1b1b80;
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
    transition: transform 0.05s ease-out;
  }
  .layer.dragging {
    transition: none;
  }
  .box {
    position: absolute;
    border: 2px dashed rgba(59,130,246,.6);
    inset: -12px -12px -12px -12px;
    border-radius: 8px;
    pointer-events: none;
    background: rgba(59,130,246,.05);
  }
  .handle {
    width: 20px; height: 20px;
    background: #fff;
    border: 2px solid #3b82f6;
    border-radius: 4px;
    position: absolute;
    transform: translate(-50%, -50%);
    cursor: nw-resize;
    touch-action: none;
  }
  .rotator {
    width: 24px; height: 24px; 
    border: 2px solid #ef4444; 
    background:#fff; 
    border-radius: 50%;
    position: absolute; 
    top: -48px; 
    left: 50%; 
    transform: translate(-50%, 0);
    cursor: crosshair;
    touch-action: none;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 12px;
  }
  .rotator::before {
    content: '↻';
    color: #ef4444;
    font-weight: bold;
  }

  .side {
    border: 1px solid #e5e7eb; border-radius: 12px; padding: 12px; background: #fff;
    display: flex; flex-direction: column; gap: 12px;
  }
  @media (max-width: 768px) {
    .side {
      order: -1; /* ย้าย sidebar ขึ้นบน */
    }
  }
  .sticker-grid {
    display: grid; grid-template-columns: repeat(3, 1fr); gap: 8px;
  }
  @media (max-width: 768px) {
    .sticker-grid {
      grid-template-columns: repeat(4, 1fr);
    }
  }
  .sticker-btn {
    border: 1px solid #e5e7eb; border-radius: 10px; background:#fafafa; padding: 8px; cursor: pointer;
    display: flex; align-items: center; justify-content: center; aspect-ratio: 1/1; overflow: hidden;
    transition: transform 0.1s, box-shadow 0.1s;
  }
  .sticker-btn:active {
    transform: scale(0.95);
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  }
  .sticker-btn img { width: 100%; height: 100%; object-fit: contain; pointer-events: none; }
  .toolbar { display:flex; gap:8px; flex-wrap:wrap; }
  @media (max-width: 768px) {
    .toolbar {
      grid-template-columns: repeat(2, 1fr);
      display: grid;
    }
  }
  .btn {
    background:#111; color:#fff; border:none; border-radius:10px; padding:12px 16px; cursor:pointer;
    font-size: 14px; font-weight: 500; 
    transition: all 0.1s;
    touch-action: manipulation; /* ป้องกัน double-tap zoom */
  }
  .btn:active {
    transform: scale(0.98);
    background: #374151;
  }
  .btn:disabled {
    background:#e5e7eb; color:#9ca3af; cursor:not-allowed; transform: none;
  }
  .btn.ghost { background:#f3f4f6; color:#111; }
  .btn.ghost:active { background: #e5e7eb; }
  .btn.ghost:disabled { background:#f9fafb; color:#d1d5db; }
  @media (max-width: 768px) {
    .btn {
      padding: 14px 12px;
      font-size: 13px;
    }
  }
  .hint { font-size: 12px; color:#6b7280; }
</style>

{#if browser}
<div class="w-full h-screen flex" on:keydown={onKey} tabindex="0">
    <!-- แถบสติ๊กเกอร์ + เครื่องมือ -->
  <aside class="w-3/12 bg-gray-50">
    <div class="bg-white p-4 flex items-center gap-2">
      <img src="logo.png" class="w-[120px]" alt="">
      <span class="uppercase font-bold text-4xl ml-4 text-gray-500">
        play ground
      </span>
    </div>
    <div class="toolbar  p-4">
      <!-- <button class="btn" on:click={downloadPNG}>ดาวน์โหลด PNG</button> -->
      <button class="btn ghost" on:click={undo} disabled={!canUndo}>↶ Undo</button>
      <button class="btn ghost" on:click={redo} disabled={!canRedo}>↷ Redo</button>
      <button class="btn ghost" on:click={bringForward}>เลเยอร์ขึ้น</button>
      <button class="btn ghost" on:click={sendBackward}>เลเยอร์ลง</button>
      <button class="btn ghost" on:click={removeSelected}>ลบที่เลือก</button>
    </div>
    <div class="hint">
      คลิกสติ๊กเกอร์เพื่อย้าย • จับมุมเพื่อย่อ/ขยาย • จับ <span style="color:#ef4444;">↻</span> เพื่อหมุน • ปุ่มลบ = Delete<br>
      <strong>Ctrl+Z</strong> = Undo • <strong>Ctrl+Y</strong> = Redo
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
  
  <!-- สเตจ/ผืนผ้าใบ -->
  <div
    class=" w-9/12 h-full flex justify-center items-center "
  >
    <!-- รูปเสื้อฐาน -->
     <div 
      class="stage" 
      bind:this={stageEl}
      on:click={clearSelection}>
       <img class="w-[50rem]" src={BASE_SRC} alt="t-shirt mockup" draggable="false" />
     </div>

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
          <!-- ปุ่มหมุน -->
          <div class="rotator" on:pointerdown={(e)=>pointerDown(e, it.id, 'rotate')}></div>
        {/if}
      </div>
    {/each}
  </div>
</div>
{:else}
<div style="padding: 2rem; text-align: center;">
  <p>กำลังโหลด...</p>
</div>
{/if}