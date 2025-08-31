<script>
  import { onMount, tick } from 'svelte';
  import { browser } from '$app/environment';
  import stickers from '$lib/assets/stickers.json';

  // --------- ตั้งค่าทรัพยากร ----------
  const BASE_SRC = '/mockups/tshirt.png';
  const stickerCategories = stickers;

  // --------- สเตจ/ผืนผ้าใบ ----------
  let stageEl;            // div ของสเตจ (แสดงผลเท่ารูปฐานบนจอ)
  let baseImg = null;     // สำหรับ export PNG

  let stageW = 720;       // อัปเดตตามขนาดจริง
  let stageH = 900;       // อัปเดตตามสัดส่วนรูปเสื้อ

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
  let handleCorner = null; // 'tl' | 'tr' | 'bl' | 'br'

  // เก็บอ้างอิง handler เพื่อถอดทิ้งได้ถูกตัว
  let moveHandlerRef = null;
  let upHandlerRef = null;

  let statusTab = true;

  // ---------------- Utils ----------------
  function uuid() {
    return crypto?.randomUUID?.() || Math.random().toString(36).slice(2);
  }

  // แปลงพิกัดจอ -> พิกัดสเตจ
  function toStageCoords(clientX, clientY) {
    const r = stageEl.getBoundingClientRect();
    const x = clientX - r.left;
    const y = clientY - r.top;
    return { x, y };
  }

  // บันทึกสถานะลง history
  function saveState() {
    const state = {
      items: items.map(it => ({
        id: it.id,
        src: it.src,
        x: it.x,
        y: it.y,
        scale: it.scale,
        rot: it.rot,
        w: it.w,
        h: it.h
      })),
      selectedId
    };

    history = history.slice(0, historyIndex + 1);
    history.push(state);
    historyIndex = history.length - 1;

    if (history.length > maxHistory) {
      history = history.slice(-maxHistory);
      historyIndex = history.length - 1;
    }
  }

  // คืนค่าสถานะจาก history
  async function restoreState(state) {
    if (!browser) return;
    const restoredItems = [];

    for (const data of state.items) {
      const img = new Image();
      img.crossOrigin = 'anonymous';
      img.src = data.src;
      await new Promise((resolve) => {
        img.onload = resolve;
        if (img.complete) resolve();
      });
      restoredItems.push({ ...data, img });
    }

    items = restoredItems;
    selectedId = state.selectedId;
    await tick();
  }

  function undo() {
    if (historyIndex > 0) {
      historyIndex--;
      return restoreState(history[historyIndex]);
    }
  }

  function redo() {
    if (historyIndex < history.length - 1) {
      historyIndex++;
      return restoreState(history[historyIndex]);
    }
  }

  $: canUndo = historyIndex > 0;
  $: canRedo = historyIndex < history.length - 1;

  function addSticker(src) {
    if (!browser) return;
    statusTab = false;

    const img = new Image();
    img.crossOrigin = 'anonymous';
    img.src = src;
    img.onload = () => {
      const base = Math.min(stageW, stageH) * 0.28; // ขนาดตั้งต้นสัมพันธ์เวที
      const scale = Math.max(0.1, base / img.width);

      const id = uuid();
      const item = {
        id,
        src,
        img,
        w: img.width,
        h: img.height,
        x: stageW / 2,
        y: stageH * 0.5,
        scale,
        rot: 0
      };

      items = [...items, item];
      selectedId = id;
      saveState();
    };
  }

  function select(id) { selectedId = id; }

  function removeSelected() {
    if (!selectedId) return;
    items = items.filter(it => it.id !== selectedId);
    selectedId = null;
    saveState();
  }

  function bringForward() {
    if (!selectedId) return;
    const idx = items.findIndex(i => i.id === selectedId);
    if (idx < 0 || idx === items.length - 1) return;
    const arr = [...items];
    [arr[idx], arr[idx + 1]] = [arr[idx + 1], arr[idx]];
    items = arr;
    saveState();
  }

  function sendBackward() {
    if (!selectedId) return;
    const idx = items.findIndex(i => i.id === selectedId);
    if (idx <= 0) return;
    const arr = [...items];
    [arr[idx], arr[idx - 1]] = [arr[idx - 1], arr[idx]];
    items = arr;
    saveState();
  }

  // ---------------- Interaction (Pointer Events เท่านั้น) ----------------
  function pointerDown(e, id, type, corner = null) {
    e.stopPropagation();
    e.preventDefault();

    const it = items.find(i => i.id === id);
    if (!it) return;
    select(id);

    const p = e; // ใช้ pointer events อย่างเดียว
    const { x, y } = toStageCoords(p.clientX, p.clientY);
    start = { x, y };
    startItem = { ...it };
    action = type;           // 'drag' | 'scale' | 'rotate'
    handleCorner = corner;   // 'tl' | 'tr' | 'bl' | 'br'

    const layer = e.currentTarget.closest('.layer');
    if (layer) layer.classList.add('dragging');

    moveHandlerRef = (ev) => pointerMove(ev);
    upHandlerRef = (ev) => pointerUp(ev);

    window.addEventListener('pointermove', moveHandlerRef, { passive: false });
    window.addEventListener('pointerup', upHandlerRef);
  }

  function pointerMove(e) {
    if (!action || !selectedId) return;
    e.preventDefault();

    const { x, y } = toStageCoords(e.clientX, e.clientY);
    const dx = x - start.x;
    const dy = y - start.y;

    const i = items.findIndex(it => it.id === selectedId);
    if (i < 0) return;
    const it = items[i];

    // ลดการชนกับการ scroll: ต้องขยับเกิน 3px จึงนับว่าลาก
    const movedEnough = Math.hypot(dx, dy) > 3;

    if (action === 'drag') {
      if (!movedEnough) return;
      it.x = startItem.x + dx;
      it.y = startItem.y + dy;

    } else if (action === 'rotate') {
      const cx = startItem.x, cy = startItem.y;
      const a0 = Math.atan2(start.y - cy, start.x - cx);
      const a1 = Math.atan2(x - cy + (cy - cy), y - cx + (cx - cx)); // เทียบเวคเตอร์จากจุดศูนย์กลาง
      // แก้ให้ถูกต้อง
      const a1b = Math.atan2(y - cy, x - cx);
      let newRot = startItem.rot + (a1b - a0);
      while (newRot > Math.PI) newRot -= 2 * Math.PI;
      while (newRot < -Math.PI) newRot += 2 * Math.PI;
      it.rot = newRot;

    } else if (action === 'scale') {
      const cx = startItem.x, cy = startItem.y;
      const d0 = Math.hypot(start.x - cx, start.y - cy);
      const d1 = Math.hypot(x - cx, y - cy);
      if (d0 > 0) {
        const s = Math.max(0.1, Math.min(5, startItem.scale * (d1 / d0)));
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

    document.querySelectorAll('.layer.dragging').forEach(el => el.classList.remove('dragging'));

    if (moveHandlerRef) {
      window.removeEventListener('pointermove', moveHandlerRef);
      moveHandlerRef = null;
    }
    if (upHandlerRef) {
      window.removeEventListener('pointerup', upHandlerRef);
      upHandlerRef = null;
    }

    if (wasMoving) saveState();
  }

  function clearSelection() { selectedId = null; }

  // ---------------- Export PNG ----------------
  async function downloadPNG() {
    if (!browser || !baseImg) return;
    await baseReady();

    const dpr = 2; // เพิ่มความคมเล็กน้อย
    const W = baseImg.naturalWidth;
    const H = baseImg.naturalHeight;

    const canvas = document.createElement('canvas');
    canvas.width = Math.floor((W * dpr) / 2);
    canvas.height = Math.floor((H * dpr) / 2);
    const ctx = canvas.getContext('2d');

    ctx.drawImage(baseImg, 0, 0, canvas.width, canvas.height);

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

  // ปรับขนาด stage ตามรูปฐานจริง (responsive)
  function fitStage() {
    if (!browser || !baseImg || !stageEl) return;

    const isMobile = window.innerWidth <= 768;
    const maxW = isMobile ? window.innerWidth - 32 : Math.min(1200, window.innerWidth - 340);
    const ratio = baseImg.naturalHeight / baseImg.naturalWidth || 1.25;
    stageW = Math.max(320, maxW);
    stageH = Math.round(stageW * ratio);
  }

  function initializeBaseImage() {
    if (!browser) return;
    baseImg = new Image();
    baseImg.crossOrigin = 'anonymous';
    baseImg.src = BASE_SRC;
    baseImg.onload = () => {
      fitStage();
    };
  }

  onMount(async () => {
    initializeBaseImage();
    await baseReady();
    fitStage();

    const ro = new ResizeObserver(() => fitStage());
    ro.observe(document.body);

    window.addEventListener('resize', fitStage);

    // บันทึกสถานะเริ่มต้น (ว่าง)
    saveState();

    // โฟกัสคีย์บอร์ดอัตโนมัติสำหรับช็อตคัต
    await tick();
    rootFocusable?.focus?.();

    return () => {
      window.removeEventListener('resize', fitStage);
      ro.disconnect();
    };
  });

  // ลบ/Undo/Redo ด้วยคีย์บอร์ด
  function onKey(e) {
    if (e.key === 'Delete' || e.key === 'Backspace') {
      removeSelected();
    } else if ((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === 'z' && !e.shiftKey) {
      e.preventDefault();
      undo();
    } else if ((e.ctrlKey || e.metaKey) && (e.key.toLowerCase() === 'y' || (e.key.toLowerCase() === 'z' && e.shiftKey))) {
      e.preventDefault();
      redo();
    }
  }

  // โหนดโฟกัสรับคีย์บอร์ด
  let rootFocusable;

  // UI
  let selectCategory = 'Basic';
</script>

<style>
  /* container หลัก: ให้เลื่อนทั้งหน้าได้ */
  .page { min-height: 100vh; }

  /* sidebar เลื่อนเองได้นิ่มบนมือถือ */
  aside { -webkit-overflow-scrolling: touch; }

  .stage {
    position: relative;
    overflow: hidden;
    /* มือถือยังเลื่อนแนวตั้งได้ */
    touch-action: pan-y;
    user-select: none;
    background: #fff;
    border-radius: 12px;
    border: 1px solid #e5e7eb;
  }
  .base-img { width: 100%; height: 100%; object-fit: contain; display: block; pointer-events: none; }
  .layer { position: absolute; transform-origin: center center; cursor: move; transition: transform 0.05s ease-out; touch-action: none; }
  .layer.dragging { transition: none; }
  .box { position: absolute; border: 2px dashed rgba(59,130,246,.6); inset: -12px; border-radius: 8px; pointer-events: none; background: rgba(59,130,246,.05); }
  .handle { width: 18px; height: 18px; background: #fff; border: 2px solid #3b82f6; border-radius: 4px; position: absolute; transform: translate(-50%, -50%); touch-action: none; }
  .handle.tl { left: -12px; top: -12px; cursor: nwse-resize; }
  .handle.tr { right: -12px; top: -12px; cursor: nesw-resize; }
  .handle.bl { left: -12px; bottom: -12px; cursor: nesw-resize; }
  .handle.br { right: -12px; bottom: -12px; cursor: nwse-resize; }
  .rotator { width: 24px; height: 24px; border: 2px solid #ef4444; background:#fff; border-radius: 50%; position: absolute; top: -48px; left: 50%; transform: translate(-50%, 0); cursor: crosshair; touch-action: none; display: flex; align-items: center; justify-content: center; font-size: 12px; }
  .rotator::before { content: '↻'; color: #ef4444; font-weight: bold; }

  .side { border: 1px solid #e5e7eb; border-radius: 12px; padding: 12px; background: #fff; display: flex; flex-direction: column; gap: 12px; }
  .sticker-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 8px; }
  @media (max-width: 768px) { .sticker-grid { grid-template-columns: repeat(4, 1fr); } }
  .sticker-btn { border: 1px solid #e5e7eb; border-radius: 10px; background:#fafafa; padding: 8px; cursor: pointer; display: flex; align-items: center; justify-content: center; aspect-ratio: 1/1; overflow: hidden; transition: transform 0.1s, box-shadow 0.1s; }
  .sticker-btn:active { transform: scale(0.95); box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
  .sticker-btn img { width: 100%; height: 100%; object-fit: contain; pointer-events: none; }
  .toolbar { display:flex; gap:8px; flex-wrap:wrap; }
  .btn { background:#111; color:#fff; border:none; border-radius:10px; padding:12px 16px; cursor:pointer; font-size: 14px; font-weight: 500; transition: all 0.1s; touch-action: manipulation; }
  .btn:active { transform: scale(0.98); background: #374151; }
  .btn:disabled { background:#e5e7eb; color:#9ca3af; cursor:not-allowed; transform: none; }
  .btn.ghost { background:#f3f4f6; color:#111; }
  .btn.ghost:active { background: #e5e7eb; }
</style>

{#if browser}
<div class="page w-full min-h-screen relative md:flex" on:keydown={onKey} tabindex="-1" bind:this={rootFocusable}>
  <!-- แถบสติ๊กเกอร์ + เครื่องมือ -->
  <aside class="w-full md:w-3/12 md:relative z-[30] overflow-auto">
    <div class="bg-white p-4 flex items-center gap-2 relative z-[40] shadow-sm">
      <img src="/logo.png" class="w-[60px] md:w-[120px]" alt="logo" />
      <span class="uppercase font-bold text-2xl md:text-4xl ml-4 text-gray-600">play ground</span>
    </div>

    <div class={`md:h-auto ${statusTab ? 'relative' : 'hidden md:block'} z-[35] bg-gray-50 overflow-auto`}>
      <div class="p-4 flex gap-2 flex-wrap">
        {#each stickerCategories as Category}
          <button class={selectCategory === Category.name ? 'btn' : 'btn ghost'} on:click={() => selectCategory = Category.name}>
            {Category.name}
          </button>
        {/each}
      </div>

      <div class="p-4">
        {#each stickerCategories as Category}
          {#if Category.name === selectCategory}
            <div class="sticker-grid">
              {#each Category.stickers as sticker}
                <button class="sticker-btn" on:click={() => addSticker(`stickers/${sticker.src}`)} aria-label={sticker.label}>
                  <img src={`stickers/${sticker.src}`} alt={sticker.label} />
                </button>
              {/each}
            </div>
          {/if}
        {/each}
      </div>
    </div>
  </aside>

  <!-- สเตจ/ผืนผ้าใบ -->
  <div class="md:w-9/12 h-full flex justify-center items-center relative">

    <div class="absolute top-0 md:top-16 md:top-0 right-0 flex justify-between m-4 md:m-8 p-4 border-2 border-gray-200 bg-white rounded-xl shadow-lg z-[20]">
      <div class="flex gap-2">
        <button class={`btn ${statusTab ? 'md:hidden' : 'md:hidden ghost'}`} on:click={() => { statusTab = !statusTab; }}>☰</button>
        <button class="btn ghost" on:click={undo} disabled={!canUndo}>↶</button>
        <button class="btn ghost" on:click={redo} disabled={!canRedo}>↷</button>
        <button class="btn ghost" on:click={bringForward} title="Bring forward">⬆︎</button>
        <button class="btn ghost" on:click={sendBackward} title="Send backward">⬇︎</button>
      </div>
      <div class="flex gap-2 ml-4 md:ml-8">
        <button class="btn ghost" on:click={removeSelected} title="Delete selected">🗑</button>
      </div>
    </div>

    <!-- ผืนสเตจ ขนาดสัมพันธ์กับรูปฐานจริง -->
     <div class="h-screen w-full flex  justify-center items-center">

       <div class="stage bg-fuchsia-400" bind:this={stageEl} style={`width:40rem; height:40rem;`} on:click={clearSelection}>
         <img class="base-img" src={BASE_SRC} alt="t-shirt mockup" draggable="false" />
   
         <!-- วาดสติ๊กเกอร์เป็นชั้น ๆ -->
         {#each items as it (it.id)}
           <div class="layer"
                style={`left:0; top:0; transform: translate(${it.x}px, ${it.y}px) rotate(${it.rot}rad) translate(-50%, -50%)`}
                on:pointerdown={(e) => pointerDown(e, it.id, 'drag')}
                on:click|stopPropagation={() => select(it.id)}>
             <img src={it.src} alt="sticker" draggable="false" style={`width:${it.w * it.scale}px; height:${it.h * it.scale}px; pointer-events:none;`} />
   
             {#if selectedId === it.id}
               <div class="box"></div>
               <div class="rotator" on:pointerdown={(e) => pointerDown(e, it.id, 'rotate')}></div>
             {/if}
           </div>
         {/each}
       </div>
     </div>
  </div>
</div>
{:else}
<div style="padding: 2rem; text-align: center;">
  <p>กำลังโหลด...</p>
</div>
{/if}
