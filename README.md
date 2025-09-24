<!doctype html>

<html lang="fa">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>تیونر (سُری / کُرُن) — ربع‌پرده</title>
  <style>
    body{font-family:Tahoma, system-ui;direction:rtl;text-align:center;padding:20px;background:#f6f7fb;color:#111}
    .card{max-width:720px;margin:16px auto;padding:18px;border-radius:12px;background:#fff;box-shadow:0 6px 20px rgba(20,20,50,0.08)}
    h1{margin:0 0 8px;font-size:20px}
    .big{font-size:36px;font-weight:700}
    .note{font-size:28px;margin:8px 0}
    .meter{height:12px;background:#eee;border-radius:6px;overflow:hidden;margin:12px 0}
    .meter > i{display:block;height:100%;width:50%;background:linear-gradient(90deg,#6bd3a9,#3aa69b)}
    label{display:inline-block;margin:8px}
    button{padding:8px 12px;border-radius:8px;border:0;background:#3aa69b;color:#fff}
    small{color:#666}
    .warning{color:#b33}
    .controls{display:flex;gap:8px;justify-content:center;flex-wrap:wrap}
  </style>
</head>
<body>
  <div class="card">
    <h1>تیونر آنلاین (پشتیبانی از ربع‌پرده — سُری و کُرُن)</h1>
    <p>برای شروع دکمهٔ «شروع» را بزنید و اجازهٔ استفاده از میکروفون را بدهید.</p>
    <div class="controls">
      <button id="startBtn">شروع</button>
      <button id="stopBtn" disabled>توقف</button>
      <label><input type="checkbox" id="showQuarter" checked> نمایش ربع‌پرده (سُری/کُرُن)</label>
      <label><input type="checkbox" id="autoUpdate" checked> به‌روزرسانی خودکار</label>
    </div><div style="margin-top:18px">
  <div class="big" id="freq">- هرتز -</div>
  <div class="note" id="noteName">- نت -</div>
  <div class="note" id="quarterName"><small>- ربع‌پرده -</small></div>
  <div class="meter"><i id="bar"></i></div>
  <div><small id="cents">- سِنت -</small></div>
  <div style="margin-top:12px"><small class="warning">تذکر: نمایش «سُری» و «کُرُن» بر اساس فاصلهٔ ربع‌پرده نسبت به نت‌های مساوی‌صدا است. شما می‌توانید آرایهٔ نت‌ها را مطابق سنت مورد نظرتان تغییر دهید.</small></div>
</div>

  </div>  <script>
    // لیست نام نت‌ها (به فارسی) برای نمایش
    const NOTE_NAMES = ['دو','دو♯/ر♭','ر','ر♯/می♭','می','فا','فا♯/سل♭','سل','سل♯/لا♭','لا','لا♯/سی♭','سی'];

    let audioContext, analyser, mediaStreamSource, rafId;
    let bufferLength = 2048;

    const startBtn = document.getElementById('startBtn');
    const stopBtn = document.getElementById('stopBtn');
    const freqEl = document.getElementById('freq');
    const noteNameEl = document.getElementById('noteName');
    const quarterEl = document.getElementById('quarterName');
    const bar = document.getElementById('bar');
    const centsEl = document.getElementById('cents');
    const showQuarter = document.getElementById('showQuarter');

    startBtn.onclick = async () => {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({audio:true});
        audioContext = new (window.AudioContext || window.webkitAudioContext)();
        analyser = audioContext.createAnalyser();
        analyser.fftSize = 2048;
        mediaStreamSource = audioContext.createMediaStreamSource(stream);
        mediaStreamSource.connect(analyser);
        stopBtn.disabled = false;
        startBtn.disabled = true;
        tick();
      } catch (e){
        alert('دسترسی به میکروفون امکان‌پذیر نشد یا مرورگر از getUserMedia پشتیبانی نمی‌کند.');
      }
    };

    stopBtn.onclick = () => {
      if (audioContext) audioContext.close();
      cancelAnimationFrame(rafId);
      startBtn.disabled = false;
      stopBtn.disabled = true;
    };

    function tick(){
      const buffer = new Float32Array(bufferLength);
      analyser.getFloatTimeDomainData(buffer);
      const f = autoCorrelate(buffer, audioContext.sampleRate);
      if (f !== -1){
        freqEl.textContent = f.toFixed(1) + ' هرتز';
        const n = 69 + 12 * Math.log2(f / 440); // MIDI number (A4 = 440)
        const semitone = Math.round(n);
        const cents = Math.round((n - semitone) * 100);
        centsEl.textContent = (cents>=0?'+':'') + cents + ' سنت نسبت به ' + noteLabel(semitone);
        noteNameEl.textContent = noteLabel(semitone);

        // ربع‌پرده: نزدیک‌ترین نیم-پرده (که معادل ربع‌پرده است) را پیدا می‌کنیم
        if (showQuarter.checked){
          const halfStep = Math.round(n * 2) / 2; // مثلاً 60.5 یعنی بین 60 و 61
          const base = Math.floor(halfStep);
          const isHalf = (Math.abs(halfStep - base - 0.0) > 0.01) && (Math.abs(halfStep - base - 0.5) < 0.01);
          if (isHalf){
            // تصمیم‌گیری سُری یا کُرُن بر اساس اینکه نیم-پرده به سمت بالاتر یا پایین‌تر باشد
            const lower = Math.floor(halfStep);
            const direction = (halfStep - lower) > 0.0 ? 'سُری' : 'کُرُن';
            quarterEl.innerHTML = `<small>ربع‌پردهٔ نزدیک: ${noteLabel(lower)} — <strong>${direction}</strong></small>`;
          } else {
            quarterEl.innerHTML = `<small>ربع‌پردهٔ نزدیک: —</small>`;
          }
        } else {
          quarterEl.innerHTML = `<small>نمایش ربع‌پرده خاموش است</small>`;
        }

        // نماگر
        const pct = Math.max(0, Math.min(100, 50 + cents / 2));
        bar.style.width = pct + '%';
      } else {
        freqEl.textContent = '- هرتز -';
        noteNameEl.textContent = '- نت -';
        quarterEl.innerHTML = '<small>- ربع‌پرده -</small>';
        centsEl.textContent = '- سنت -';
        bar.style.width = '50%';
      }

      rafId = requestAnimationFrame(tick);
    }

    // تبدیل شمارهٔ MIDI به نام نتی به فارسی
    function noteLabel(midiNumber){
      const idx = (midiNumber + 120) % 12; // محافظت در برابر منفی
      const octave = Math.floor(midiNumber / 12) - 1;
      return NOTE_NAMES[idx] + ' (' + octave + ')';
    }

    // تابع اتوکورِلیشن برای تشخیص فرکانس (معتبر و سبک)
    function autoCorrelate(buf, sampleRate){
      // از روی float32 buffer ساده‌ترین الگوریتم اتوکورلیشن
      let SIZE = buf.length;
      let rms = 0;
      for (let i=0;i<SIZE;i++){ let val = buf[i]; rms += val*val; }
      rms = Math.sqrt(rms/SIZE);
      if (rms < 0.01) return -1; // صدا خیلی ضعیفه

      let r1=0, r2=SIZE-1, th=0.2;
      for (let i=0;i<SIZE/2;i++) if (Math.abs(buf[i])<th){ r1=i; break; }
      for (let i=1;i<SIZE/2;i++) if (Math.abs(buf[SIZE-i])<th){ r2=SIZE-i; break; }
      buf = buf.slice(r1, r2);
      SIZE = buf.length;

      const c = new Array(SIZE).fill(0);
      for (let i=0;i<SIZE;i++)
        for (let j=0;j<SIZE-i;j++) c[i] = c[i] + buf[j]*buf[j+i];

      let d = 0; while (c[d] > c[d+1]) d++;
      let maxval = -1, maxpos = -1;
      for (let i=d;i<SIZE;i++){
        if (c[i] > maxval){ maxval = c[i]; maxpos = i; }
      }
      if (maxval < 0.001) return -1;

      // پارابولیکِ interpolation برای دقت
      let T0 = maxpos;
      const x1 = c[T0-1], x2 = c[T0], x3 = c[T0+1];
      const a = (x1 + x3 - 2*x2)/2;
      const b = (x3 - x1)/2;
      if (a) T0 = T0 - b/(2*a);

      return sampleRate / T0;
    }

  </script></body>
</html>
