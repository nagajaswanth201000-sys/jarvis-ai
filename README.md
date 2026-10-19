<!-- ===== JARVIS STAGE 1 — UI + Voice Commands (append to index.html) ===== -->
<style id="jarvis-stage1-styles">
  /* Basic layout and futuristic orb */
  :root { --bg:#000; --cyan:#00f0ff; --muted:#333; --glass: rgba(255,255,255,0.03); }
  body { background:var(--bg); color:#cfeff3; font-family:system-ui,Segoe UI,Roboto,"Helvetica Neue",Arial; }
  #jarvis-root{position:fixed; inset:12px; pointer-events:none; z-index:99999; display:flex; flex-direction:column; gap:12px; align-items:center;}
  #jarvis-card{pointer-events:auto; width:100%; max-width:760px; margin:auto; background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01)); border-radius:14px; padding:12px; box-shadow:0 6px 30px rgba(0,0,0,0.7); border:1px solid rgba(0,255,255,0.04);}
  .jarvis-row{display:flex; gap:12px; align-items:center;}
  .orb{width:84px;height:84px;border-radius:50%;background:radial-gradient(circle at 30% 30%, rgba(0,255,255,0.18), rgba(0,255,255,0.06) 40%, transparent 60%), linear-gradient(45deg, rgba(0,180,255,0.06), rgba(0,255,255,0.02)); box-shadow:0 0 40px rgba(0,255,255,0.14), inset 0 2px 12px rgba(255,255,255,0.02); display:flex;align-items:center;justify-content:center;position:relative;overflow:hidden;}
  .orb::after{content:'';position:absolute; inset:0; background:radial-gradient(circle at 50% 40%, rgba(0,255,255,0.06), transparent 35%); animation:orb-pulse 4s infinite;}
  @keyframes orb-pulse{0%{transform:scale(1)}50%{transform:scale(1.06)}100%{transform:scale(1)}}
  #jarvis-mic{width:56px;height:56px; border-radius:50%; background:linear-gradient(180deg, #001, #012); display:flex; align-items:center; justify-content:center; border:2px solid rgba(0,255,255,0.12); cursor:pointer;}
  #jarvis-mic .dot{width:18px;height:18px;border-radius:50%; background:var(--cyan); box-shadow:0 0 18px var(--cyan);}
  .status {font-size:13px;color:#9ee; opacity:0.95}
  #jarvis-log{height:180px; overflow:auto;background:rgba(0,0,0,0.45); padding:8px; border-radius:8px; border:1px solid rgba(255,255,255,0.02); font-size:13px;}
  .jarvis-btn {background:transparent;border:1px solid rgba(255,255,255,0.04); color:#cff; padding:8px 10px;border-radius:8px;cursor:pointer;}
  #jarvis-controls{display:flex; gap:8px; flex-wrap:wrap;}
  #gesture-preview{width:100%; max-height:320px; background:#080808; border-radius:8px; overflow:hidden; display:flex; align-items:center; justify-content:center; color:var(--muted);}
  /* Responsive for Realme 8i portrait */
  @media (max-width:420px){ #jarvis-card{padding:10px} .orb{width:64px;height:64px} #jarvis-mic{width:48px;height:48px} #jarvis-log{height:140px} }
</style>

<div id="jarvis-root" aria-live="polite" role="region">
  <div id="jarvis-card" role="application" aria-label="Jarvis AI Assistant">
    <div class="jarvis-row" style="justify-content:space-between;">
      <div style="display:flex;gap:12px;align-items:center;">
        <div class="orb" id="jarvis-orb" title="Jarvis Orb"></div>
        <div>
          <div style="font-weight:700;font-size:18px">JARVIS AI — Phone Assistant</div>
          <div class="status" id="jarvis-status">Status: Idle</div>
        </div>
      </div>
      <div style="display:flex;flex-direction:column;align-items:flex-end;gap:8px">
        <button id="jarvis-mic" title="Tap to talk" aria-pressed="false"><div class="dot"></div></button>
        <div style="font-size:12px;color:#9ee">Microphone: <span id="jarvis-mic-state">off</span></div>
      </div>
    </div>

    <div style="height:8px"></div>

    <div id="jarvis-controls" class="jarvis-row">
      <button class="jarvis-btn" id="jarvis-set-channel">Set gaming channel</button>
      <button class="jarvis-btn" id="jarvis-open-settings">Open settings</button>
      <button class="jarvis-btn" id="jarvis-start-gesture">Start camera (gestures)</button>
      <button class="jarvis-btn" id="jarvis-stop-gesture">Stop camera</button>
    </div>

    <div style="height:12px"></div>

    <div id="jarvis-log" aria-live="polite"></div>

    <div style="height:12px"></div>

    <div id="gesture-preview">Gesture preview inactive — camera off.</div>

  </div>
</div>

<script id="jarvis-stage1-script">
(function(){
  // Safety and capability flags
  const supportsSR = !!(window.SpeechRecognition || window.webkitSpeechRecognition);
  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition || null;
  const synth = window.speechSynthesis || null;

  // Utilities
  function log(msg, tone='info'){
    const el = document.getElementById('jarvis-log');
    const ts = new Date().toLocaleTimeString();
    el.insertAdjacentHTML('afterbegin', `<div style="margin:6px 0"><b>[${ts}]</b> ${msg}</div>`);
  }
  function speak(text, opts={}){
    if(!synth){ log('SpeechSynthesis not supported in this browser.'); return; }
    const ut = new SpeechSynthesisUtterance(text);
    ut.lang = opts.lang || 'en-US';
    ut.rate = opts.rate || 1;
    synth.cancel(); synth.speak(ut);
    log('Jarvis: ' + text);
  }

  // Create or reference UI
  const micBtn = document.getElementById('jarvis-mic');
  const micState = document.getElementById('jarvis-mic-state');
  const statusEl = document.getElementById('jarvis-status');
  const orb = document.getElementById('jarvis-orb');
  const gesturePreview = document.getElementById('gesture-preview');

  // local config
  const cfg = {
    gamingChannelKey: 'jarvis_gaming_channel',
    wakeWord: 'hey jarvis' // wake-word (case-insensitive)
  };

  function setStatus(text){
    statusEl.textContent = 'Status: ' + text;
  }

  // Speech Recognition setup
  let recog = null;
  let listening = false;
  let wakeMode = true; // require wake word first
  if(SpeechRecognition){
    recog = new SpeechRecognition();
    recog.continuous = false;
    recog.interimResults = false;
    recog.maxAlternatives = 1;
    recog.lang = 'en-US';
    recog.onstart = ()=>{ listening = true; micState.textContent='on'; micBtn.setAttribute('aria-pressed','true'); setStatus('Listening'); orb.style.filter='drop-shadow(0 0 14px rgba(0,255,255,0.8))'; };
    recog.onend = ()=>{ listening=false; micState.textContent='off'; micBtn.setAttribute('aria-pressed','false'); setStatus('Idle'); orb.style.filter='none'; };
    recog.onerror = (ev)=>{ log('Recognition error: '+ev.error); setStatus('Error: '+ev.error); };
    recog.onresult = (ev)=>{ const text = ev.results[0][0].transcript.trim(); log('Heard: '+text); processCommand(text); };
  } else {
    log('SpeechRecognition not supported in this browser. Use Chrome on Android for best results.');
  }

  // Start/Stop helpers
  function startListening(){
    if(!recog){ log('SpeechRecognition unavailable.'); return; }
    try { recog.start(); } catch(e){ log('Could not start recognition: '+e.message); }
  }
  function stopListening(){ if(recog){ recog.stop(); } }

  micBtn.addEventListener('click', ()=>{
    if(!recog){ speak('Speech recognition is not supported. Use Chrome on Android and grant microphone permission.'); return; }
    if(listening){ stopListening(); } else { startListening(); }
  });

  // Recognize spoken digit words to digits
  const digitMap = {
    'zero':'0','one':'1','two':'2','three':'3','four':'4','five':'5','six':'6','seven':'7','eight':'8','nine':'9',
    'oh':'0','o':'0','plus':'+', 'dash':'-'
  };
  function wordsToNumberWords(text){
    // Accept sequences like "nine eight seven six five four three two one zero" or "nine eight seven million ...".
    text = text.toLowerCase().replace(/[^a-z0-9 +\-]/g,' ').trim();
    const words = text.split(/\s+/);
    let out = '';
    for(const w of words){
      if(digitMap[w]!==undefined){ out += digitMap[w]; continue; }*

