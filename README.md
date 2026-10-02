<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <meta http-equiv="Content-Security-Policy" content="default-src 'self'; style-src 'self' 'unsafe-inline'; script-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self';" />
  <title>NeoMatch — Communication & Dating (Secure Demo)</title>
  <style>
    :root{--bg:#0b1020;--card:#121a31;--muted:#9ba8d0;--txt:#ecf1ff;--accent:#ff4d6a;--ok:#32d583}
    *{box-sizing:border-box} body{margin:0;font-family:system-ui,-apple-system,Segoe UI,Roboto,sans-serif;background:linear-gradient(160deg,#0a1020,#121b39 60%,#1b1030);color:var(--txt)}
    .app{max-width:520px;margin:16px auto;min-height:calc(100vh - 32px);background:rgba(255,255,255,.04);border:1px solid rgba(255,255,255,.1);border-radius:20px;overflow:hidden;backdrop-filter:blur(10px)}
    header{display:flex;justify-content:space-between;align-items:center;padding:14px 16px;border-bottom:1px solid rgba(255,255,255,.08)}
    h1{font-size:1.1rem;margin:0}.muted{color:var(--muted);font-size:.86rem}
    main{padding:12px}.card{background:var(--card);padding:14px;border-radius:14px;border:1px solid rgba(255,255,255,.08);margin-bottom:10px}
    input,textarea,button,select{width:100%;padding:10px;border-radius:10px;border:1px solid rgba(255,255,255,.15);background:#0f1730;color:var(--txt)}
    button{cursor:pointer;font-weight:700}.row{display:grid;grid-template-columns:1fr 1fr;gap:8px}.danger{background:#3b1320}.accent{background:linear-gradient(120deg,#ff4d6a,#ff7f98);border:none}
    .ok{color:var(--ok)} .hidden{display:none}
    nav{display:flex;gap:8px;padding:10px;border-top:1px solid rgba(255,255,255,.08)} nav button{font-size:.9rem}
    .msg{padding:8px 10px;border-radius:10px;margin:6px 0;max-width:85%}.me{margin-left:auto;background:#5f1d2a}.them{background:#202b52}
    .profile-avatar{width:64px;height:64px;border-radius:50%;display:grid;place-items:center;font-weight:700;background:#3952a3}
  </style>
</head>
<body>
<div class="app">
  <header>
    <h1>✦ NeoMatch</h1>
    <div id="geoStatus" class="muted">Геолокация: не запрошена</div>
  </header>
  <main>
    <section id="discover" class="screen">
      <div class="card">
        <div id="profilePreview"></div>
        <p class="muted" id="distance">Расстояние: —</p>
        <div class="row">
          <button id="btnNo" class="danger">✕ Пропустить</button>
          <button id="btnYes" class="accent">♥ Лайк</button>
        </div>
      </div>
      <div class="card">
        <button id="requestGeo">Разрешить геолокацию</button>
        <p class="muted">Геолокация используется только локально в браузере для демо.</p>
      </div>
    </section>

    <section id="chat" class="screen hidden">
      <div class="card">
        <div id="chatBox"></div>
        <div class="row" style="grid-template-columns:1fr auto;">
          <input id="chatInput" maxlength="280" placeholder="Сообщение (до 280 символов)" />
          <button id="send" style="width:auto">➤</button>
        </div>
      </div>
    </section>

    <section id="profile" class="screen hidden">
      <div class="card">
        <div class="profile-avatar" id="avatar">A</div>
        <label class="muted">Имя</label><input id="name" maxlength="40" />
        <label class="muted">Возраст</label><input id="age" type="number" min="18" max="99" />
        <label class="muted">Био</label><textarea id="bio" maxlength="300"></textarea>
        <button id="save" class="accent">Сохранить профиль</button>
        <p class="muted">Надёжность: данные валидируются и сохраняются atomically в localStorage.</p>
      </div>
    </section>
  </main>
  <nav>
    <button data-s="discover">🔥 Discover</button>
    <button data-s="chat">💬 Chat</button>
    <button data-s="profile">👤 Profile</button>
  </nav>
</div>
<script>
(() => {
  'use strict';
  const $ = s => document.querySelector(s);
  const screens = ['discover','chat','profile'];
  const key = 'neomatch_secure_v2';
  const defaults = { profile:{name:'Alex',age:26,bio:'Люблю кофе и прогулки',avatar:'A'}, likes:0, passes:0, chat:[{by:'them',text:'Привет! 👋'}], geo:null };

  function safeParse(v){ try{return JSON.parse(v)}catch{return null} }
  function load(){ return Object.assign({}, defaults, safeParse(localStorage.getItem(key)) || {}); }
  let state = load();

  function persist(){
    const snapshot = JSON.stringify(state);
    localStorage.setItem(key + '_tmp', snapshot);
    localStorage.setItem(key, snapshot);
    localStorage.removeItem(key + '_tmp');
  }

  function esc(s){ return String(s).replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c])); }
  function validateProfile(p){
    const name = (p.name||'').trim().slice(0,40) || defaults.profile.name;
    const ageN = Number(p.age); const age = Number.isInteger(ageN) && ageN>=18 && ageN<=99 ? ageN : defaults.profile.age;
    const bio = (p.bio||'').trim().slice(0,300) || defaults.profile.bio;
    return {name, age, bio, avatar: name.charAt(0).toUpperCase()};
  }

  function renderDiscover(){
    $('#profilePreview').innerHTML = `<strong>${esc(state.profile.name)}</strong>, ${state.profile.age}<br><span class="muted">${esc(state.profile.bio)}</span>`;
    $('#distance').textContent = state.geo ? `Расстояние: ~${Math.max(1,Math.round((Math.abs(state.geo.lat)+Math.abs(state.geo.lng))%12))} км` : 'Расстояние: неизвестно (нужна геолокация)';
    $('#geoStatus').textContent = state.geo ? `Геолокация: ${state.geo.lat.toFixed(3)}, ${state.geo.lng.toFixed(3)}` : 'Геолокация: не запрошена';
  }

  function renderChat(){
    const box = $('#chatBox'); box.innerHTML = '';
    state.chat.slice(-40).forEach(m => {
      const div = document.createElement('div');
      div.className = `msg ${m.by==='me'?'me':'them'}`;
      div.textContent = m.text;
      box.appendChild(div);
    });
    box.scrollTop = box.scrollHeight;
  }

  function renderProfile(){
    $('#name').value = state.profile.name; $('#age').value = state.profile.age; $('#bio').value = state.profile.bio; $('#avatar').textContent = state.profile.avatar;
  }

  function show(s){ screens.forEach(x => document.getElementById(x).classList.toggle('hidden', x!==s)); if(s==='chat') renderChat(); if(s==='profile') renderProfile(); if(s==='discover') renderDiscover(); }

  document.querySelectorAll('nav button').forEach(b => b.addEventListener('click', ()=>show(b.dataset.s)));
  $('#btnYes').addEventListener('click', ()=>{ state.likes++; state.chat.push({by:'them',text:'У нас мэтч! Хочешь созвон вечером?'}); persist(); renderDiscover(); });
  $('#btnNo').addEventListener('click', ()=>{ state.passes++; persist(); renderDiscover(); });

  $('#send').addEventListener('click', ()=>{
    const input = $('#chatInput'); const text = input.value.trim().slice(0,280); if(!text) return;
    state.chat.push({by:'me',text}); input.value='';
    setTimeout(()=>{ state.chat.push({by:'them',text:'Спасибо! Расскажи о себе 🙂'}); persist(); renderChat(); }, 350);
    persist(); renderChat();
  });

  $('#save').addEventListener('click', ()=>{
    state.profile = validateProfile({name:$('#name').value, age:$('#age').value, bio:$('#bio').value});
    persist(); renderProfile(); renderDiscover();
    alert('Профиль сохранён безопасно ✅');
  });

  $('#requestGeo').addEventListener('click', ()=>{
    if(!('geolocation' in navigator)){ $('#geoStatus').textContent = 'Геолокация: недоступна'; return; }
    navigator.geolocation.getCurrentPosition(
      pos => { state.geo = {lat:pos.coords.latitude, lng:pos.coords.longitude, t:Date.now()}; persist(); renderDiscover(); },
      err => { $('#geoStatus').textContent = `Геолокация: ошибка (${err.code})`; },
      {enableHighAccuracy:false, timeout:6000, maximumAge:120000}
    );
  });

  window.addEventListener('storage', e => { if (e.key===key && e.newValue){ const n=safeParse(e.newValue); if(n){ state=Object.assign({},defaults,n); renderDiscover(); renderChat(); renderProfile(); } } });
  show('discover');
})();
</script>
</body>
</html>
