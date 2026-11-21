<!doctype html>
<html lang="id">
<head>
<meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>SMART VIRAL CLIPPER AI</title>
<script src="https://cdn.tailwindcss.com"></script>
<style>
body{background:#0b1020;color:#eef2ff;font-family:Inter,Arial,sans-serif}
.panel{background:#151e33;border:1px solid #293550;border-radius:16px}
input,select{background:#0b1427;border:1px solid #334155;border-radius:9px;padding:11px;width:100%;color:white}
.btn{background:linear-gradient(100deg,#7c3aed,#2563eb);padding:12px 18px;border-radius:10px;font-weight:700}
.muted{color:#94a3b8}
video{width:100%;background:#000;border-radius:10px;max-height:300px}
</style>
</head>
<body>
<div class="max-w-7xl mx-auto p-4 md:p-8">
<header class="flex flex-wrap justify-between items-center gap-4 mb-8">
<div><div class="text-purple-400 font-bold tracking-widest text-xs">AI VIDEO WORKSPACE</div><h1 class="text-3xl font-black mt-1">SMART VIRAL CLIPPER</h1><p class="muted mt-1">Temukan momen terbaik dari video panjang secara otomatis.</p></div>
<div class="text-xs rounded-full px-4 py-2 bg-emerald-950 text-emerald-300" id="connection">● Memeriksa server...</div>
</header>

<div class="grid md:grid-cols-3 gap-4 mb-6">
<div class="panel p-5"><div class="muted text-sm">Status AI</div><div class="text-xl font-bold mt-2" id="aiStatus">Memeriksa...</div></div>
<div class="panel p-5"><div class="muted text-sm">Status pekerjaan</div><div class="text-xl font-bold mt-2" id="jobStatus">Belum ada proses</div></div>
<div class="panel p-5"><div class="muted text-sm">Jumlah klip</div><div class="text-3xl font-bold mt-1" id="clipTotal">0</div></div>
</div>

<section class="panel p-5 md:p-7 mb-6">
<h2 class="text-xl font-bold mb-4">01 · Upload video</h2>
<label class="block border-2 border-dashed border-purple-500/70 rounded-xl p-8 text-center cursor-pointer hover:bg-purple-900/10">
<div class="text-4xl mb-3">☁️</div><b>Pilih video untuk dianalisis AI</b><p class="muted text-sm mt-2">MP4, MOV, WEBM, MKV, AVI</p>
<input class="hidden" id="video" type="file" accept="video/*">
</label>
<div id="fileInfo" class="muted text-sm mt-3">Belum ada file dipilih.</div>
<div id="previewWrap" class="mt-4 hidden"><video id="preview" controls></video></div>
</section>

<section class="panel p-5 md:p-7 mb-6">
<h2 class="text-xl font-bold mb-4">02 · Pengaturan AI Clipper</h2>
<div class="grid sm:grid-cols-3 gap-4">
<label class="text-sm muted">Durasi target
<select id="seconds" class="mt-2"><option value="15">15 detik</option><option value="30" selected>30 detik</option><option value="45">45 detik</option><option value="60">60 detik</option><option value="90">90 detik</option><option value="120">120 detik</option></select></label>
<label class="text-sm muted">Maksimal klip
<select id="count" class="mt-2"><option>3</option><option>5</option><option selected>8</option><option>10</option><option>12</option></select></label>
<label class="text-sm muted">Rasio video
<select id="aspect" class="mt-2"><option value="9:16">9:16 · TikTok / Shorts</option><option value="1:1">1:1 · Square</option><option value="16:9">16:9 · YouTube</option><option value="original">Original</option></select></label>
</div>
<div class="mt-4"><label class="text-sm muted">API key aplikasi (jika admin mengaktifkan)</label><input id="apiKey" type="password" placeholder="Masukkan APP_API_KEY bila diperlukan" class="mt-2"></div>
<div class="flex flex-wrap gap-3 mt-5"><button id="start" class="btn">✂ ANALISIS & BUAT KLIP</button><button id="clear" class="bg-slate-700 px-5 py-3 rounded-lg">Reset tampilan</button></div>
</section>

<section class="panel p-5 md:p-7 mb-6">
<div class="flex justify-between gap-3"><h2 class="text-xl font-bold">03 · Progress pemrosesan</h2><b id="percent">0%</b></div>
<div class="w-full bg-slate-700 rounded-full h-2 mt-4 overflow-hidden"><div id="bar" class="h-2 bg-purple-500 rounded-full" style="width:0%"></div></div>
<p id="stage" class="muted text-sm mt-3">Menunggu video.</p><p id="error" class="text-red-400 text-sm mt-2"></p>
</section>

<section class="panel p-5 md:p-7">
<div class="flex flex-wrap justify-between items-center gap-3 mb-5"><h2 class="text-xl font-bold">04 · Hasil klip AI</h2><span class="muted text-xs">Urutan berdasarkan skor AI</span></div>
<div id="results" class="grid md:grid-cols-2 xl:grid-cols-3 gap-4"><p class="muted text-sm">Hasil klip akan muncul di sini setelah proses selesai.</p></div>
</section>
<footer class="muted text-center text-xs mt-8">SMART VIRAL CLIPPER AI · Node.js + FFmpeg</footer>
</div>
<script>
let currentJob=null, pollTimer=null, previewUrl=null;
const $=id=>document.getElementById(id);
function headers(){const key=$('apiKey').value.trim();return key?{Authorization:'Bearer '+key}:{}}
async function checkServer(){
 try{const r=await fetch('/api/health');const d=await r.json();$('connection').textContent='● Server terhubung';$('aiStatus').textContent=d.aiConfigured?'AI Aktif':'API key AI belum diatur';}
 catch{$('connection').textContent='● Server tidak terhubung';$('aiStatus').textContent='Offline';}
}
$('video').addEventListener('change',()=>{
 const f=$('video').files[0];if(!f)return;
 $('fileInfo').textContent=f.name+' · '+(f.size/1024/1024).toFixed(1)+' MB';
 if(previewUrl)URL.revokeObjectURL(previewUrl);
 previewUrl=URL.createObjectURL(f);$('preview').src=previewUrl;$('previewWrap').classList.remove('hidden');
});
$('start').onclick=async()=>{
 const file=$('video').files[0];if(!file)return alert('Pilih video terlebih dahulu.');
 $('start').disabled=true;$('error').textContent='';$('stage').textContent='Mengunggah video ke server...';
 const form=new FormData();form.append('video',file);form.append('clipSeconds',$('seconds').value);form.append('count',$('count').value);form.append('aspect',$('aspect').value);
 try{
  const r=await fetch('/api/upload',{method:'POST',headers:headers(),body:form});const d=await r.json();
  if(!r.ok)throw new Error(d.error||'Upload gagal');
  currentJob=d.jobId;$('jobStatus').textContent='Diproses';poll();
 }catch(e){$('error').textContent=e.message;$('start').disabled=false;}
};
async function poll(){
 try{
  const r=await fetch('/api/jobs/'+currentJob,{headers:headers()});const d=await r.json();if(!r.ok)throw new Error(d.error);
  $('bar').style.width=(d.progress||0)+'%';$('percent').textContent=(d.progress||0)+'%';$('stage').textContent=d.stage||d.status;
  $('jobStatus').textContent=d.status;
  if(d.status==='completed'){render(d.clips||[]);$('start').disabled=false;return;}
  if(d.status==='failed'){$('error').textContent=d.error||'Pemrosesan gagal';$('start').disabled=false;return;}
  pollTimer=setTimeout(poll,2000);
 }catch(e){$('error').textContent=e.message;$('start').disabled=false;}
}
function render(clips){
 $('clipTotal').textContent=clips.length;
 $('results').innerHTML=clips.map((c,i)=>`
 <article class="bg-slate-900 border border-slate-700 rounded-xl p-3">
 <video controls preload="metadata" src="${c.url}"></video>
 <div class="flex justify-between gap-2 mt-3"><b class="text-sm">${escapeHtml(c.title||'Clip '+(i+1))}</b><span class="text-xs text-emerald-300">Score ${c.score}/100</span></div>
 <p class="text-xs text-purple-300 mt-2">${escapeHtml(c.hook||'')}</p>
 <p class="muted text-xs mt-2">${c.duration}s · ${c.start.toFixed(1)}s – ${c.end.toFixed(1)}s</p>
 <p class="muted text-xs mt-2">${escapeHtml(c.reason||'')}</p>
 <a class="btn block text-center text-sm mt-3" href="${c.url}" download="${escapeHtml(c.filename||'clip.mp4')}">⬇ Download MP4</a>
 </article>`).join('');
}
function escapeHtml(s){return String(s).replace(/[&<>"']/g,c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]));}
$('clear').onclick=()=>{if(pollTimer)clearTimeout(pollTimer);currentJob=null;$('results').innerHTML='<p class="muted text-sm">Hasil klip akan muncul di sini.</p>';$('clipTotal').textContent='0';$('bar').style.width='0%';$('percent').textContent='0%';$('stage').textContent='Menunggu video.';$('error').textContent='';$('jobStatus').textContent='Belum ada proses';$('start').disabled=false;};
checkServer();
</script>
</body></html>
