
<script>
let currentJob = null;
let pollTimer = null;
let previewUrl = null;

const $ = id => document.getElementById(id);

function headers() {
  const key = $("apiKey").value.trim();
  return key ? { Authorization: "Bearer " + key } : {};
}

async function checkServer() {
  try {
    const r = await fetch("/api/health");
    if (!r.ok) throw new Error("Server error");

    const d = await r.json();

    $("connection").textContent =
      d.ffmpeg && d.ffprobe
        ? "● Server terhubung"
        : "● FFmpeg belum siap";

    $("aiStatus").textContent =
      d.aiConfigured ? "AI Aktif" : "API key AI belum diatur";

  } catch (e) {
    $("connection").textContent = "● Server tidak terhubung";
    $("aiStatus").textContent = "Offline";
  }
}

$("video").addEventListener("change", () => {
  const file = $("video").files[0];
  if (!file) return;

  $("fileInfo").textContent =
    file.name + " · " + (file.size / 1024 / 1024).toFixed(1) + " MB";

  if (previewUrl) URL.revokeObjectURL(previewUrl);

  previewUrl = URL.createObjectURL(file);
  $("preview").src = previewUrl;
  $("previewWrap").classList.remove("hidden");
});

$("start").onclick = async () => {
  const file = $("video").files[0];

  if (!file) {
    alert("Pilih video terlebih dahulu.");
    return;
  }

  $("start").disabled = true;
  $("error").textContent = "";
  $("results").innerHTML = "";
  $("clipTotal").textContent = "0";
  $("jobStatus").textContent = "Mengunggah";

  updateProgress(1, "Mengunggah video ke server...");

  try {
    const form = new FormData();

    form.append("video", file);
    form.append("clipSeconds", $("seconds").value);
    form.append("count", $("count").value);
    form.append("aspect", $("aspect").value);

    const response = await fetch("/api/upload", {
      method: "POST",
      headers: headers(),
      body: form
    });

    const data = await response.json();

    if (!response.ok) {
      throw new Error(data.error || "Upload video gagal.");
    }

    if (!data.jobId) {
      throw new Error("Server tidak mengirim ID pekerjaan.");
    }

    currentJob = data.jobId;
    $("jobStatus").textContent = "Diproses";

    poll();

  } catch (error) {
    showError(error.message);
  }
};

function updateProgress(value, message) {
  const percent = Math.max(0, Math.min(100, Number(value) || 0));

  $("bar").style.width = percent + "%";
  $("percent").textContent = percent + "%";
  $("stage").textContent = message || "Memproses...";
}

async function poll() {
  if (!currentJob) return;

  try {
    const response = await fetch(
      "/api/jobs/" + encodeURIComponent(currentJob),
      { headers: headers() }
    );

    const data = await response.json();

    if (!response.ok) {
      throw new Error(data.error || "Gagal membaca status pekerjaan.");
    }

    updateProgress(data.progress, data.message || data.status);

    $("jobStatus").textContent = data.status || "Diproses";

    if (data.status === "completed") {
      await loadResults(currentJob);
      updateProgress(100, "Selesai! Semua klip berhasil dibuat.");
      $("jobStatus").textContent = "Selesai";
      $("start").disabled = false;
      return;
    }

    if (data.status === "failed") {
      throw new Error(data.error || "Pemrosesan video gagal.");
    }

    pollTimer = setTimeout(poll, 2000);

  } catch (error) {
    showError(error.message);
  }
}

async function loadResults(jobId) {
  const response = await fetch(
    "/api/jobs/" + encodeURIComponent(jobId) + "/download",
    { headers: headers() }
  );

  const data = await response.json();

  if (!response.ok) {
    throw new Error(data.error || "Gagal mengambil hasil klip.");
  }

  render(data.clips || []);
}

function render(clips) {
  $("clipTotal").textContent = clips.length;

  if (!clips.length) {
    $("results").innerHTML =
      '<p class="muted text-sm">Tidak ada klip yang dihasilkan.</p>';
    return;
  }

  $("results").innerHTML = clips.map((c, i) => {
    const title = escapeHtml(c.title || "Clip " + (i + 1));
    const hook = escapeHtml(c.hook || "");
    const reason = escapeHtml(c.reason || "");
    const filename = escapeHtml(c.filename || "clip.mp4");
    const url = escapeHtml(c.url || "");
    const score = Number(c.score) || 0;
    const duration = Number(c.duration) || 0;
    const start = Number(c.start) || 0;
    const end = Number(c.end) || 0;

    return `
      <article class="bg-slate-900 border border-slate-700 rounded-xl p-3">
        <video controls preload="metadata" src="${url}"></video>

        <div class="flex justify-between gap-2 mt-3">
          <b class="text-sm">${title}</b>
          <span class="text-xs text-emerald-300">
            Score ${score}/100
          </span>
        </div>

        <p class="text-xs text-purple-300 mt-2">${hook}</p>

        <p class="muted text-xs mt-2">
          ${duration}s · ${start.toFixed(1)}s – ${end.toFixed(1)}s
        </p>

        <p class="muted text-xs mt-2">${reason}</p>

        <a class="btn block text-center text-sm mt-3"
           href="${url}"
           download="${filename}">
           ⬇ Download MP4
        </a>
      </article>
    `;
  }).join("");
}

function escapeHtml(value) {
  return String(value).replace(/[&<>"']/g, c => ({
    "&": "&amp;",
    "<": "&lt;",
    ">": "&gt;",
    '"': "&quot;",
    "'": "&#39;"
  }[c]));
}

function showError(message) {
  $("error").textContent = message;
  $("jobStatus").textContent = "Gagal";
  $("start").disabled = false;
  if (pollTimer) clearTimeout(pollTimer);
}

$("clear").onclick = () => {
  if (pollTimer) clearTimeout(pollTimer);

  currentJob = null;

  $("results").innerHTML =
    '<p class="muted text-sm">Hasil klip akan muncul di sini setelah proses selesai.</p>';

  $("clipTotal").textContent = "0";
  $("bar").style.width = "0%";
  $("percent").textContent = "0%";
  $("stage").textContent = "Menunggu video.";
  $("error").textContent = "";
  $("jobStatus").textContent = "Belum ada proses";
  $("start").disabled = false;
};

checkServer();
</script>
</body>
</html>
