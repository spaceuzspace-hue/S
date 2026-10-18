<!DOCTYPE html>
<html lang="uz">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Space AI — matn va rasm yaratuvchi</title>
<style>
:root{--ink:#0b1030;--deep:#151a47;--line:#2c3380;--star:#f6c667;--violet:#8b7bff;--text:#eef0ff;--mute:#a4a9d6}
*{box-sizing:border-box;margin:0}
body{font-family:"Trebuchet MS",system-ui,sans-serif;background:var(--ink);color:var(--text);min-height:100vh;display:flex;flex-direction:column;align-items:center;padding:28px 16px 40px;overflow-x:hidden}
.orbit{position:fixed;top:-180px;right:-180px;width:460px;height:460px;border:1px solid var(--line);border-radius:50%;pointer-events:none;animation:spin 90s linear infinite}
.orbit::after{content:"";position:absolute;top:36px;left:60px;width:14px;height:14px;border-radius:50%;background:var(--star)}
@keyframes spin{to{transform:rotate(360deg)}}
@media(prefers-reduced-motion:reduce){.orbit{animation:none}}
main{width:100%;max-width:640px;position:relative}
h1{font-size:2.4rem;letter-spacing:.5px;margin:6px 0 4px}
h1 span{color:var(--star)}
.sub{color:var(--mute);margin-bottom:22px;line-height:1.5}
.tabs{display:flex;gap:8px;margin-bottom:12px}
.tab{flex:1;padding:12px;border:1px solid var(--line);background:transparent;color:var(--mute);border-radius:12px;font-size:1rem;cursor:pointer}
.tab[aria-selected=true]{background:var(--violet);color:#0b1030;border-color:var(--violet);font-weight:bold}
textarea{width:100%;min-height:120px;padding:14px;border-radius:14px;border:1px solid var(--line);background:var(--deep);color:var(--text);font:inherit;resize:vertical}
textarea:focus,button:focus-visible{outline:2px solid var(--star);outline-offset:2px}
.go{width:100%;margin-top:12px;padding:14px;border:0;border-radius:14px;background:var(--star);color:#0b1030;font-size:1.05rem;font-weight:bold;cursor:pointer}
.go:disabled{opacity:.6;cursor:wait}
#out{margin-top:20px}
.card{background:var(--deep);border:1px solid var(--line);border-radius:14px;padding:16px;line-height:1.6;white-space:pre-wrap;word-wrap:break-word}
.err{border-color:#ff7b7b;color:#ffc9c9}
img{width:100%;border-radius:14px;display:block}
.row{display:flex;gap:8px;margin-top:10px}
.row a,.row button{flex:1;text-align:center;padding:11px;border-radius:12px;border:1px solid var(--line);background:transparent;color:var(--text);text-decoration:none;font-size:.95rem;cursor:pointer}
.note{margin-top:26px;color:var(--mute);font-size:.85rem;line-height:1.5}
</style>
</head>
<body>
<div class="orbit" aria-hidden="true"></div>
<main>
  <h1>Space <span>AI</span></h1>
  <p class="sub">Bepul matn va rasm yarating. Barcha natijalar islom qoidalariga mos bo‘lishi uchun tekshiriladi.</p>

  <div class="tabs" role="tablist">
    <button class="tab" role="tab" id="t-text" aria-selected="true">Matn</button>
    <button class="tab" role="tab" id="t-image" aria-selected="false">Rasm</button>
  </div>

  <textarea id="prompt" placeholder="Nima yaratishni xohlaysiz? Masalan: Ona haqida qisqa she’r"></textarea>
  <button class="go" id="go">Yaratish</button>
  <div id="out" aria-live="polite"></div>

  <p class="note">Space AI xato qilishi mumkin. Muhim ma’lumotlarni tekshirib oling.</p>
</main>

<script>
// ====== SOZLAMA: Cloudflare Worker manzilingizni shu yerga qo'ying ======
const API_URL = "https://space-ai.YOUR-NAME.workers.dev";
// API kalit bu faylda EMAS — u worker'da yashirin saqlanadi (worker.js ga qarang).

let mode = "text";
const $ = id => document.getElementById(id);
const out = $("out"), go = $("go"), box = $("prompt");

function setMode(m){
  mode = m;
  $("t-text").setAttribute("aria-selected", m === "text");
  $("t-image").setAttribute("aria-selected", m === "image");
  box.placeholder = m === "text"
    ? "Nima yaratishni xohlaysiz? Masalan: Ona haqida qisqa she’r"
    : "Rasmni tasvirlab bering. Masalan: Tog‘lar ustida yulduzli osmon";
  out.innerHTML = "";
}
$("t-text").onclick = () => setMode("text");
$("t-image").onclick = () => setMode("image");

function showError(msg){
  out.innerHTML = "";
  const d = document.createElement("div");
  d.className = "card err";
  d.textContent = msg;
  out.appendChild(d);
}

go.onclick = async () => {
  const prompt = box.value.trim();
  if(!prompt){ showError("Avval so‘rovingizni yozing."); return; }
  go.disabled = true; go.textContent = "Yaratilmoqda…"; out.innerHTML = "";
  try{
    const res = await fetch(API_URL + "/" + mode, {
      method: "POST",
      headers: {"Content-Type": "application/json"},
      body: JSON.stringify({prompt})
    });
    if(!res.ok){
      let m = "Xatolik yuz berdi. Birozdan so‘ng qayta urining.";
      try{ m = (await res.json()).error || m; }catch(e){}
      throw new Error(m);
    }
    if(mode === "text"){
      const {text} = await res.json();
      const d = document.createElement("div");
      d.className = "card"; d.textContent = text;
      const row = document.createElement("div"); row.className = "row";
      const b = document.createElement("button"); b.textContent = "Nusxa olish";
      b.onclick = () => navigator.clipboard.writeText(text).then(() => b.textContent = "Nusxalandi");
      row.appendChild(b); out.append(d, row);
    }else{
      const url = URL.createObjectURL(await res.blob());
      out.innerHTML = '<img alt="Yaratilgan rasm" src="'+url+'"><div class="row"><a href="'+url+'" download="space-ai.jpg">Yuklab olish</a></div>';
    }
  }catch(e){
    showError(e.message || "Tarmoq xatosi. Internetni tekshiring.");
  }finally{
    go.disabled = false; go.textContent = "Yaratish";
  }
};
</script>
</body>
</html>
