<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Snip — AI Background Remover</title>
<style>
  :root{
    --bg:#0b0f14;
    --panel:#141a22;
    --line:#222b36;
    --ink:#e8eef5;
    --muted:#7c8896;
    --accent:#4f8cff;
    --accent-2:#3ddc97;
    --danger:#ff6b6b;
    --radius:16px;
  }
  *{box-sizing:border-box;margin:0;padding:0}
  body{
    font-family:ui-sans-serif,system-ui,"Segoe UI",Roboto,Helvetica,Arial,sans-serif;
    background:radial-gradient(1200px 600px at 70% -10%, #16202c 0%, var(--bg) 55%);
    color:var(--ink);min-height:100vh;padding:36px 20px 70px;line-height:1.5;
  }
  .wrap{max-width:920px;margin:0 auto}
  header{margin-bottom:26px}
  .eyebrow{color:var(--accent-2);font-size:12px;letter-spacing:3px;text-transform:uppercase;
    font-family:ui-monospace,monospace;margin-bottom:10px}
  h1{font-size:38px;font-weight:800;letter-spacing:-1.5px;line-height:1.03}
  h1 span{background:linear-gradient(90deg,var(--accent),var(--accent-2));
    -webkit-background-clip:text;background-clip:text;color:transparent}
  .sub{color:var(--muted);font-size:14.5px;margin-top:12px;max-width:580px}

  .drop{
    border:1.5px dashed var(--line);border-radius:var(--radius);background:var(--panel);
    padding:52px 24px;text-align:center;cursor:pointer;margin-top:26px;
    transition:border-color .15s, background .15s;
  }
  .drop:hover,.drop.over{border-color:var(--accent);background:#101720}
  .drop b{color:var(--accent);font-size:16px}
  .drop p{color:var(--muted);font-size:13px;margin-top:8px}

  .stage{display:none;margin-top:26px}
  .stage.show{display:block}

  .compare{display:grid;grid-template-columns:1fr 1fr;gap:16px}
  @media(max-width:640px){.compare{grid-template-columns:1fr}}
  .cell{border:1px solid var(--line);border-radius:var(--radius);overflow:hidden;background:var(--panel)}
  .cap{font-size:11px;letter-spacing:1.5px;text-transform:uppercase;color:var(--muted);
    padding:10px 14px;border-bottom:1px solid var(--line);font-family:ui-monospace,monospace}
  .view{
    min-height:260px;display:flex;align-items:center;justify-content:center;
    background-image:
      linear-gradient(45deg,#1a222c 25%,transparent 25%),
      linear-gradient(-45deg,#1a222c 25%,transparent 25%),
      linear-gradient(45deg,transparent 75%,#1a222c 75%),
      linear-gradient(-45deg,transparent 75%,#1a222c 75%);
    background-size:22px 22px;background-position:0 0,0 11px,11px -11px,-11px 0;
    background-color:#0e141b;position:relative;
  }
  .view img{max-width:100%;max-height:56vh;display:block}
  .view.plain{background:#0e141b}

  .spinner{
    width:34px;height:34px;border:3px solid var(--line);border-top-color:var(--accent);
    border-radius:50%;animation:spin .8s linear infinite;
  }
  @keyframes spin{to{transform:rotate(360deg)}}
  .status{color:var(--muted);font-size:13px;margin-top:12px;text-align:center;
    font-family:ui-monospace,monospace}

  .btns{display:flex;gap:10px;flex-wrap:wrap;margin-top:20px;justify-content:center}
  button{
    font-family:inherit;font-size:14px;font-weight:600;border:1px solid var(--line);
    background:#1a222c;color:var(--ink);padding:12px 20px;border-radius:11px;cursor:pointer;
    transition:transform .05s, border-color .15s, background .15s;
  }
  button:hover{border-color:var(--accent)}
  button:active{transform:translateY(1px)}
  button:disabled{opacity:.45;cursor:not-allowed}
  button.primary{background:linear-gradient(90deg,var(--accent),#3f7bff);color:#04101f;border:none}

  footer{margin-top:40px;color:var(--muted);font-size:12px;text-align:center}
  footer a{color:var(--accent-2);text-decoration:none}
  .note{background:#101720;border:1px solid var(--line);border-radius:12px;padding:12px 16px;
    color:var(--muted);font-size:12.5px;margin-top:18px}
</style>
</head>
<body>
<div class="wrap">
  <header>
    <div class="eyebrow">100% in your browser · nothing uploaded</div>
    <h1>Vishal <span>AI background remover</span></h1>
    <p class="sub">Drop a photo and the background disappears — hair, edges, and all. An AI model runs entirely on your device, so your images never leave your computer.</p>
  </header>

  <div class="drop" id="drop">
    <b>Choose an image</b> or drop it here
    <p>PNG · JPG · WEBP — people, products, pets, logos</p>
    <input type="file" id="file" accept="image/*" hidden>
  </div>

  <div class="stage" id="stage">
    <div class="compare">
      <div class="cell">
        <div class="cap">Original</div>
        <div class="view plain" id="viewOrig"></div>
      </div>
      <div class="cell">
        <div class="cap">Background removed</div>
        <div class="view" id="viewOut">
          <div class="spinner" id="spinner"></div>
        </div>
      </div>
    </div>
    <div class="status" id="status">Loading AI model… (first run downloads it once)</div>
    <div class="btns">
      <button class="primary" id="download" disabled>Download PNG</button>
      <button id="new">New image</button>
    </div>
  </div>

  <div class="note">
    First image takes a bit longer while the model downloads (~a few MB, cached after that). Everything after is fast and offline-capable.
  </div>

  <footer>
    Free · no sign-in · no watermark · <a href="https://pages.github.com/" target="_blank" rel="noopener">Host on GitHub Pages</a>
  </footer>
</div>

<script type="module">
import { removeBackground } from "https://esm.sh/@imgly/background-removal@1.5.5";

const $ = id => document.getElementById(id);
const drop = $('drop'), fileInput = $('file'), stage = $('stage');
const viewOrig = $('viewOrig'), viewOut = $('viewOut'), spinner = $('spinner');
const statusEl = $('status'), downloadBtn = $('download');
let resultURL = null;

drop.onclick = () => fileInput.click();
fileInput.onchange = e => e.target.files[0] && handle(e.target.files[0]);
['dragover','dragenter'].forEach(ev => drop.addEventListener(ev, e=>{e.preventDefault();drop.classList.add('over')}));
['dragleave','drop'].forEach(ev => drop.addEventListener(ev, e=>{e.preventDefault();drop.classList.remove('over')}));
drop.addEventListener('drop', e => e.dataTransfer.files[0] && handle(e.dataTransfer.files[0]));

async function handle(file){
  stage.classList.add('show');
  drop.style.display = 'none';
  downloadBtn.disabled = true;
  if (resultURL){ URL.revokeObjectURL(resultURL); resultURL = null; }

  viewOrig.innerHTML = '';
  const origImg = new Image();
  origImg.src = URL.createObjectURL(file);
  viewOrig.appendChild(origImg);

  viewOut.innerHTML = '';
  viewOut.appendChild(spinner);
  statusEl.textContent = 'Removing background… hang tight.';

  try{
    const blob = await removeBackground(file, {
      progress: (key, current, total) => {
        if (key.includes('fetch')) {
          const pct = total ? Math.round(current/total*100) : 0;
          statusEl.textContent = `Downloading model… ${pct}%`;
        } else {
          statusEl.textContent = 'Processing image…';
        }
      }
    });
    resultURL = URL.createObjectURL(blob);
    const out = new Image();
    out.src = resultURL;
    viewOut.innerHTML = '';
    viewOut.appendChild(out);
    statusEl.textContent = 'Done. Download your transparent PNG below.';
    downloadBtn.disabled = false;
  }catch(err){
    viewOut.innerHTML = '';
    statusEl.textContent = 'Something went wrong: ' + (err?.message || err) + '. Try another image.';
    console.error(err);
  }
}

downloadBtn.onclick = () => {
  if (!resultURL) return;
  const a = document.createElement('a');
  a.download = 'no-background.png';
  a.href = resultURL;
  a.click();
};

$('new').onclick = () => {
  stage.classList.remove('show');
  drop.style.display = 'block';
  fileInput.value = '';
  if (resultURL){ URL.revokeObjectURL(resultURL); resultURL = null; }
};
</script>
</body>
</html>
