# Strumenti di test del Calibro (3 ottobre 2026)

Un blocco per strumento (il Progetto non accetta gli zip). Salva ogni blocco con il nome del titolo nella stessa cartella, insieme a `mcube.py` e ai file estratti da `index.html` con `estrai_parti.py`. Le istruzioni sono nel primo blocco.

## LEGGIMI.txt

````text
Banco di prova del Calibro per Mirror Cube (aggiornato al 2 ottobre 2026)

Preparazione (una volta):
  pip install playwright --break-system-packages && python3 -m playwright install chromium
  python3 estrai_parti.py index.html .        -> cubejs.js, core.js, app.js, cubejs_node.js
  per il 3D nelle prove: npm install three@0.128 (THREE_JS=node_modules/three/build/three.min.js)
  salva in questa cartella gli altri blocchi di strumenti-test.md e mcube.py

Rimettere il codice nella pagina dopo le modifiche:
  python3 build_index.py index.html core.js app.js index_nuovo.html

Fotogrammi di un video (30 al secondo, verticali):
  mkdir -p frames/NOME && ffmpeg -i video.mp4 -vf "fps=30,scale=720:1280" -q:v 3 frames/NOME/%05d.jpg

Replay di un video nella pagina vera (circa 2 volte la durata del video):
  INIT=init3.js python3 replay.py index_nuovo.html NOME uscita.json 66
  Opzioni (variabili d'ambiente):
    TRUTH=verita.json    nel sintetico conferma come nome la faccia vera (altrimenti accetta la proposta)
    TAP=1                tocca davvero etichetta, «Cambia» e le schede; TAPHOLD=1 lascia la barra visibile
    CONFIRMFIRST=1       conferma solo la prima faccia (le altre richieste restano in sospeso)
    EXTERR="0,1400,TL,up,10"     una larghezza sbagliata apposta in un intervallo di tempo
    MISALIGN="800,5800,rows,0,0.13"   uno strato girato simulato (righe o colonne)
    VERIFY=1             avvia il controllo finale («e' risolto?») invece della scansione; esito in result.verify
    SHOTS=4.2,12.6       fotografie della schermata a quei secondi; THREE_JS=... per il 3D
  Nel risultato: events (caselle nel tempo), log (ogni lettura), result (fase, stato ricostruito, mosse,
  red = pezzi rossi), dbg (ogni controllo delle sei facce: pezzi rossi e distanza D).

Video sintetici (mcube.py, verita' esatta):
  FOC=1400 DIST=260 python3 gen_live.py s3 "L2 B' U F2 D R' B2 L' F U2" "H,Tu20,Td20,Tl20,Tr20,Ml,H,Tu20,Td20,Tl20,Tr20,Ml,H,Tu20,Td20,Tl20,Tr20,Ml,H,Tu20,Td20,Tl20,Tr20,Md,H,Tu20,Td20,Tl20,Tr20,Muu,H,Tu20,Td20,Tl20,Tr20" 9
  FOC=1400 DIST=260 python3 gen_live.py p1 "L2 B' U F2 D R' B2 L' F U2" "H,Tu20,Td20,Tl20,Tr20,Ml,H,H,Mr,H,H" 9
  (H tieni di fronte, Tu/Td/Tl/Tr inclina di 20 gradi, Ml/Mr/Md/Muu gira il cubo; TRUTH_ONLY=1 rifa' solo la verita')
  node eval_syn.js uscita.json s3_truth.json     nomi, giri, tasselli sbagliati, mosse

Mescolata nota sul cubo vero (video del 2 ottobre, R U F' L2 D B' R2 U'):
  python3 k7truth.py "R U F' L2 D B' R2 U'" k7_true.json   larghezze e altezze vere di ogni faccia
  node k7compare.js uscita.json k7_true.json                faccia vera di ogni lettura, misure fuori
  node k7persp.js uscita.json k7_true.json                  prospettiva, rilievo, ricostruzione
  node k7check.js uscita.json 129400                        pezzi rossi dalle larghezze al tempo di un controllo
  node k7walls.js uscita.json                               fasce nere tra i blocchi (prova offline, non in uso)

Altre analisi:
  node summ.js uscita.json          riassunto del replay (conferme, fine)
  node test_relief.js uscita.json verita.json   rilievo misurato inclinando contro la verita' (sintetici)
  node test_align3.js uscita.json verita.json   righe e colonne fuori linea, mediana su tutte le viste
  node decodifica_dati.js           ridecodifica le 12 misure di ogni faccia ricopiate dall'immagine dei dati

Pagina della soluzione:
  THREE_JS=... python3 test_solve.py index_nuovo.html    testo, disegno e 3D cambiano a ogni mossa
  python3 test_finish.py index_nuovo.html                 «Verifica che sia risolto» compare solo alla fine
  (entrambi leggono uno stato mescolato noto da /tmp/scr.txt:
   node -e "const C=require('./cubejs_node.js');const c=new C();c.move(\"R U F' L2 D B' R2 U'\");require('fs').writeFileSync('/tmp/scr.txt',c.asString())")

La pagina come app (3 ottobre):
  python3 test_android.py index_nuovo.html CARTELLA_DEL_SITO
  (CARTELLA_DEL_SITO con sw.js, manifest, icone, three.min.js e i caratteri: errori della fotocamera, caratteri dal sito,
   fotocamera fermata e riaperta cambiando app, tasto Indietro, funzionamento senza rete)
  Nei replay e nelle prove della soluzione THREE_JS=percorso/three.min.js copia la libreria 3D accanto alla pagina.
````

## replay.py

````python
"""Replays a recorded live scan through the real page (headless Chromium).

usage: python3 replay.py <index.html> <vid> <out.json> [delta_ms] [t0_ms]
The frames of <vid> must be in $FRAMES/<vid>/00001.jpg ... (30 fps, 720x1280); default ./frames.
"""
import base64, functools, http.server, json, os, re, sys, threading, time

from playwright.sync_api import sync_playwright

src, vid, out = sys.argv[1], sys.argv[2], sys.argv[3]
delta = float(sys.argv[4]) if len(sys.argv) > 4 else 66
t0 = float(sys.argv[5]) if len(sys.argv) > 5 else 0
HERE = os.path.dirname(os.path.abspath(__file__))
FR = os.path.abspath(os.environ.get("FRAMES", os.path.join(HERE, "frames")))
nf = len([x for x in os.listdir(f"{FR}/{vid}") if x.endswith(".jpg")])

serve = os.path.join(HERE, f"serve_{os.getpid()}")
os.makedirs(serve, exist_ok=True)
html = open(src, encoding="utf-8").read()
# expose the page state to the harness (test copy only)
html = html.replace("const st = { phase:", "const st = window.__st = { phase:", 1)
html = html.replace("const live = { on:", "const live = window.__live = { on:", 1)
html = html.replace("const trk = { anchor:", "const trk = window.__trk = { anchor:", 1)
assert "window.__st" in html and "window.__live" in html
open(f"{serve}/index.html", "w", encoding="utf-8").write(html)
if not os.path.exists(f"{serve}/frames"):
    os.symlink(FR, f"{serve}/frames")


class Quiet(http.server.SimpleHTTPRequestHandler):
    def log_message(self, *a):
        pass


srv = http.server.ThreadingHTTPServer(("127.0.0.1", 0), functools.partial(Quiet, directory=serve))
port = srv.server_address[1]
threading.Thread(target=srv.serve_forever, daemon=True).start()

errors = []
with sync_playwright() as p:
    b = p.chromium.launch(args=["--autoplay-policy=no-user-gesture-required", "--use-fake-ui-for-media-stream"])
    ctx = b.new_context(viewport={"width": 390, "height": 844}, device_scale_factor=1)
    THREE_JS = os.environ.get("THREE_JS")
    ctx.route("https://cdnjs.cloudflare.com/**", (lambda r: r.fulfill(path=THREE_JS, content_type="application/javascript")) if THREE_JS else (lambda r: r.abort()))   # 3D only when asked
    ctx.add_init_script(path=os.path.join(HERE, os.environ.get("INIT", "init.js")))
    page = ctx.new_page()
    page.on("pageerror", lambda e: errors.append(str(e)))
    if os.environ.get("THREE_JS"):
        open(os.path.join(serve, "three.min.js"), "wb").write(open(os.environ["THREE_JS"], "rb").read())
    if os.environ.get("TRUTH"):
        open(os.path.join(serve, "truth.json"), "w").write(open(os.environ["TRUTH"]).read())
    if os.environ.get("ORI"):
        open(os.path.join(serve, "ori.json"), "w").write(open(os.environ["ORI"]).read())
    page.goto(f"http://127.0.0.1:{port}/index.html?vid={vid}&nf={nf}&delta={delta}&t0={t0}&shots={os.environ.get('SHOTS', '')}&ori={1 if os.environ.get('ORI') else ''}&truth={1 if os.environ.get('TRUTH') else ''}&tap={1 if os.environ.get('TAP') else ''}&tapHold={1 if os.environ.get('TAPHOLD') else ''}&okAt={os.environ.get('OKAT', '')}&mis={os.environ.get('MISALIGN', '')}&confirmFirst={1 if os.environ.get('CONFIRMFIRST') else ''}&extErr={os.environ.get('EXTERR', '')}")
    page.wait_for_timeout(800)
    page.evaluate("window.__cam.ensure()")
    page.evaluate("window.__verify()" if os.environ.get("VERIFY") else "document.querySelector('[data-act=\"live\"]').click()")
    t_start = time.time()
    last = None
    while True:
        for _ in range(10):
            time.sleep(0.3)
            sr = page.evaluate("window.__shotReady || 0")
            if sr:
                page.wait_for_timeout(700)
                page.screenshot(path=out.replace(".json", f"_shot_{sr/1000:.1f}.png"))
                page.evaluate("window.__shotReady = 0; window.__shotGo && window.__shotGo()")
        s = page.evaluate("({ t: window.__vt.t, ended: window.__cam.ended, on: window.__live.on, phase: window.__st.phase, n: window.__log.length })")
        if last is None or s["n"] // 150 != last["n"] // 150:
            print(f"  t={s['t']/1000:.1f}s analyses={s['n']} live={s['on']} phase={s['phase']} ({time.time()-t_start:.0f}s)", flush=True)
        last = s
        if s["ended"] or (not s["on"] and s["n"] > 0):
            break
        if s["n"] == 0 and time.time() - t_start > 60:
            errors.append("no analysis started")
            break
    res = page.evaluate("""() => {
      const st = window.__st, d = st.decoded;
      const slots = st.slots.map((x) => x.res ? { ok: x.res.ok, ext: x.res.ext, sizes: x.res.sizes, plx: x.res.plx ? x.res.plx.turned.length : null, warn: !!x.warn } : null);
      return { verify: window.__verifyResult === undefined ? null : window.__verifyResult, red: window.__st.red ? [...window.__st.red.idx] : null, phase: st.phase, liveOn: window.__live.on, pill: document.querySelector('#pillText').textContent,
        slots, round: st.round || 0, rechecks: st.rechecks || null, flagged: st.flagged ? [...st.flagged] : null,
        decoded: d ? { facelets: d.facelets, thick: d.thick, worst: d.worst, worstFrom: d.worstFrom, margin: d.frame && d.frame.margin,
          plx: d.plx ? { n: d.plx.n, bad: d.plx.bad } : null, persp: d.persp ? { n: d.persp.n, bad: d.persp.bad, off: !!d.persp.off } : null,
          placed: d.layout && d.layout.placed, pairCost: d.layout && d.layout.pairCost, secondPairCost: d.layout && d.layout.secondPairCost } : null,
        moves: st.sol ? st.sol.moves : null, afterErr: window.__afterErr || null,
        forced: (() => { try { const s = st.slots; if (!s.every((x) => x.res && x.res.ok)) return null;
          const CLS = { A: [16.8, 49.6], B: [22.7, 43.7], C: [29.4, 36.9] }, placed = {}, turned = {};
          s.forEach((x, j) => { if (j && !x.guess) { placed[x.face] = j; turned[x.face] = x.turn || 0; } });
          const pick = (d) => ({ facelets: d.facelets, thick: d.thick, worst: d.worst, margin: d.frame && d.frame.margin, pairCost: d.layout.pairCost, hinted: !!d.layout.hinted, freePairCost: d.layout.freePairCost, hintPairCost: d.layout.hintPairCost, placed: d.layout.placed, turned: d.layout.turned });
          return { hint: pick(MirrorVision.decodeFree(s.map((x) => x.res), CLS, Object.keys(placed).length ? { placed, turned } : null)), free: pick(MirrorVision.decodeFree(s.map((x) => x.res), CLS)),
            slots: s.map((x) => ({ face: x.face, turn: x.turn || 0, guess: !!x.guess, src: x.src || null })) }; } catch (e) { return String(e); } })() };
    }""")
    # the data image of the app ("Salva i dati della scansione")
    try:
        page.evaluate("document.querySelector('[data-act=\"export\"]').click()")
        page.wait_for_timeout(500)
        url = page.evaluate("(() => { const i = document.querySelector('#exportBox img, #exportBoxScan img'); return i ? i.src : null; })()")
        if url:
            open(out.replace(".json", "_dati.png"), "wb").write(base64.b64decode(url.split(",", 1)[1]))
    except Exception as e:  # noqa
        errors.append("export: " + str(e))
    log = page.evaluate("window.__log")
    events = page.evaluate("window.__events")
    checks = page.evaluate("window.__trkChecks || []")
    dbg = page.evaluate("window.__dbgOrbit || []")
    taps = page.evaluate("window.__taps || []")
    okd = page.evaluate("window.__okDone || null")
    m3red = page.evaluate("window.__m3red || null")
    mist = page.evaluate("window.__misTapped || null")
    b.close()
srv.shutdown()
json.dump({"vid": vid, "delta": delta, "t0": t0, "nf": nf, "result": res, "errors": errors, "log": log, "events": events, "checks": checks, "dbg": dbg, "taps": taps, "okDone": okd, "m3red": m3red, "misTapped": mist}, open(out, "w"))
print(json.dumps({"phase": res["phase"], "pill": res["pill"], "round": res["round"], "errors": errors[:3],
                  "facelets": res["decoded"] and res["decoded"]["facelets"], "moves": res["moves"]}, ensure_ascii=False))
````

## init3.js

````javascript
// Replay of a real recording through the real page, deterministically:
// - the "camera" shows the video frame of the current virtual time;
// - every analysis in the worker takes DELTA ms of virtual time (as on the phone),
//   whatever the speed of this machine;
// - performance.now() returns the virtual time.
(() => {
  const P = new URLSearchParams(location.search);
  if (!P.get("vid")) return;
  const DELTA = +(P.get("delta") || 66), FPS = 30, NF = +(P.get("nf") || 0), VID = P.get("vid");
  const T0 = +(P.get("t0") || 0);           // start the replay this far into the video (ms)
  const VT = (window.__vt = { t: 0, delta: DELTA });
  performance.now = () => VT.t;
  // headless replay: the overlay does not need 60 redraws a second (results do not depend on it:
  // the virtual clock only moves with the analyses)
  window.requestAnimationFrame = (cb) => setTimeout(() => cb(VT.t), 30);
  window.cancelAnimationFrame = (id) => clearTimeout(id);
  const cam = document.createElement("canvas");
  cam.width = 720; cam.height = 1280;
  const cctx = cam.getContext("2d");
  const C = (window.__cam = { frame: null, idx: -1, stream: null, ended: false });
  const pad = (i) => String(i).padStart(5, "0");
  const idxAt = (t) => Math.max(1, Math.min(NF, Math.floor(((t + T0) * FPS) / 1000) + 1));
  C.idxAt = idxAt;
  C.ensure = async () => {
    const i = idxAt(VT.t);
    if (i >= NF) C.ended = true;
    if (i === C.idx) return;
    const r = await fetch(`/frames/${VID}/${pad(i)}.jpg`);
    const bmp = await createImageBitmap(await r.blob());
    if (C.frame && C.frame.close) C.frame.close();
    C.frame = bmp; C.idx = i;
  };
  navigator.mediaDevices.getUserMedia = async () => {
    C.stream = cam.captureStream(30);
    const paint = () => { if (C.frame) cctx.drawImage(C.frame, 0, 0, 720, 1280); };
    paint(); setInterval(paint, 50);
    return C.stream;
  };
  const orig = CanvasRenderingContext2D.prototype.drawImage;
  CanvasRenderingContext2D.prototype.drawImage = function (src, ...a) {
    if (C.stream && src instanceof HTMLVideoElement && src.srcObject === C.stream && C.frame) return orig.call(this, C.frame, ...a);
    return orig.call(this, src, ...a);
  };
  window.__log = []; window.__trkChecks = []; if (P.get('extErr')) { const q = P.get('extErr').split(','); window.__extErr = [+q[0], +q[1], q[2], q[3], +q[4]]; } if (P.get('mis')) { const q = P.get('mis').split(','); window.__mis = [+q[0], +q[1], q[2], +q[3], +q[4]]; }
  // with a turned layer simulated: tap «Fatto, l'ho raddrizzata» when it shows (as someone who fixed it)
  if (P.get('mis')) setInterval(() => {
    const b = [...document.querySelectorAll('.pend-bar')].find((x) => !x.hidden && /Fatto, l'ho raddrizzata/.test(x.textContent));
    if (b && !window.__misTapped) { window.__misTapped = VT.t; b.querySelector('.ok').click(); }
  }, 400);
  // the user who confirms the name on the face: takes the page's proposal, or (synthetic scans,
  // truth.json) the face really in front, as an attentive user would
  window.__autoConfirm = (prop) => prop;
  // tap mode: nobody confirms by code; the label is tapped where it is drawn, as a finger would
  // (the event goes to whatever element is on top at that point). Every second face: «Cambia»,
  // then the proposed name in the list.
  if (P.get('tap')) {
    // tap mode: nobody confirms by code. One face: a tap on the name drawn on the face (the event
    // goes to whatever element is on top there); the next: «Cambia» in the bar, then the first card.
    window.__autoConfirm = null; window.__labTest = true; window.__taps = [];
    let n = 0, waitCard = false;
    setInterval(() => {
      if (waitCard) {
        const card = document.querySelector('.chooser .card');
        if (card) { window.__taps.push({ t: VT.t, card: card.dataset.name, cards: [...document.querySelectorAll('.chooser .card')].map((x) => x.dataset.name) }); card.click(); waitCard = false; }
        return;
      }
      const c = document.getElementById('stage'), L = window.__labBox, bar = document.querySelector('.pend-bar');
      if (!c || !L || !bar || bar.hidden) return;
      if (n % 2 === 1) { bar.querySelector('.chg').click(); window.__taps.push({ t: VT.t, what: 'cambia', name: L.name }); waitCard = true; n++; return; }
      const r = c.getBoundingClientRect(), bx = L.box;
      const x = r.left + ((bx[0] + bx[2]) / 2) * (r.width / L.w), y = r.top + ((bx[1] + bx[3]) / 2) * (r.height / L.h);
      const el = document.elementFromPoint(x, y);
      if (!el) return;
      el.dispatchEvent(new PointerEvent('pointerup', { clientX: x, clientY: y, bubbles: true }));
      window.__taps.push({ t: VT.t, on: el.id || el.className || el.tagName, what: 'nome', name: L.name });
      n++;
    }, 400);
  }
  // pictures of the confirmation: the bar while a face waits, then the list after «Cambia»
  if (P.get('tapHold')) {
    window.__autoConfirm = null; window.__labTest = true;
    let opened = false;
    setInterval(() => {
      const bar = document.querySelector('.pend-bar');
      if (!opened && VT.t > 6000 && bar && !bar.hidden) { bar.querySelector('.chg').click(); opened = true; }
    }, 300);
  }
  // «Va bene così» tapped at a chosen time, when the bar shows
  if (P.get('okAt')) setInterval(() => {
    const t = +P.get('okAt'), bars = [...document.querySelectorAll('.pend-bar')].filter((b) => !b.hidden && /Va bene così/.test(b.textContent));
    if (VT.t >= t && bars.length && !window.__okDone) { window.__okDone = VT.t; bars[0].querySelector('button').click(); }
  }, 300);
  // only the first face is confirmed by code; the next questions wait (to see them go away)
  if (P.get('confirmFirst')) window.__autoConfirm = (prop) => { window.__autoConfirm = null; return prop; };
  if (P.get('truth')) fetch('truth.json').then((r) => r.json()).then((j) => {
    const T = j.truth || [];
    window.__autoConfirm = (prop, faces, free) => {
      let b = T[0]; for (const x of T) { if (x.t <= VT.t) b = x; else break; }
      const k = b ? faces.indexOf(b.P) : -1;
      return free.includes(k) ? k : prop;
    };
  }); window.__dbgOrbit = [];
  // motion sensors recorded with the video (or written by the synthetic generator): replayed as
  // deviceorientation events, frame by frame
  if (P.get('ori')) fetch('ori.json').then((r) => r.json()).then((j) => {
    window.__ori = (j.samples || []).map((s) => (Array.isArray(s) ? s : [s.t, s.ori[0], s.ori[1], s.ori[2]])).sort((a, b) => a[0] - b[0]);
    let k = 0, lastSent = -1;
    window.__oriAt = (tv) => {
      const L = window.__ori; if (!L.length) return;
      while (k + 1 < L.length && L[k + 1][0] <= tv) k++;
      while (k > 0 && L[k][0] > tv) k--;
      if (k === lastSent) return;
      lastSent = k;
      window.dispatchEvent(new DeviceOrientationEvent('deviceorientation', { alpha: L[k][1], beta: L[k][2], gamma: L[k][3], absolute: false }));
    };
  }); window.__shots = (P.get('shots') || '').split(',').filter(Boolean).map(Number);
  window.__events = [];
  let lastSig = "";
  const sig = (st) => JSON.stringify(st.slots.map((s) => (s.res && s.res.ok ? s.res.ext : null)));
  window.__after = (res, tSend, fIdx) => {
    const st = window.__st, live = window.__live;
    if (!st || !live) return;
    const e = {
      t: Math.round(tSend), f: fIdx, ok: !!res.ok, why: res.why || "", miss: res.missing,
      ext: res.ext || null, sizes: res.sizes || null, g: res.g || null, h: res.h || null, m: res.m, gw: res.gw,
      edges: res.edges || null, badAt: res.badAt || null,
      tip: live.tip, cur: st.cur, turn: !!live.turnPh, pend: live.pending ? live.pending.slot : null,
      flagged: st.flagged ? [...st.flagged] : null, round: st.round || 0,
      plx: st.slots.map((s) => (s.res && s.res.plx ? s.res.plx.turned.length : -1)),
      done: st.slots.map((s) => !!(s.res && s.res.ok)),
      recent: live.recent ? live.recent.length : 0,
      frac: live.cand ? +(live.cand.frac || 0).toFixed(2) : null,
      wait: live.waitTurn ? live.waitTurn.slot : null,
      lastFace: live.lastFace ? live.lastFace.slot : null,
      roi: res.job && res.job.roi ? [res.job.roi.x, res.job.roi.y, res.job.roi.w, res.job.roi.h].map((v) => Math.round(v * 10) / 10) : null,
      k: res.job ? res.job.k : null,
      th: res.theta != null ? +res.theta.toFixed(4) : null,
      cen: (() => { try { if (!res.g || !res.job) return null; const q = MirrorVision.toImage(res, 0.5 * (res.g[0] + res.g[1]), 0.5 * (res.h[0] + res.h[1]));
        return [+(res.job.roi.x + q[0] / res.job.k).toFixed(1), +(res.job.roi.y + q[1] / res.job.k).toFixed(1), +((res.g[1] - res.g[0]) / res.job.k).toFixed(1), +((res.h[1] - res.h[0]) / res.job.k).toFixed(1)]; } catch (e) { return null; } })(),
      anchor: window.__trk && window.__trk.anchor ? { slot: window.__trk.anchor.slot, q: window.__trk.anchor.q } : null,
    };
    window.__log.push(e);
    const s = sig(st);
    if (s !== lastSig) {
      lastSig = s;
      window.__events.push({ t: Math.round(VT.t), f: fIdx, kind: "slots", slots: st.slots.map((x) => (x.res && x.res.ok ? { ext: x.res.ext, sizes: x.res.sizes, src: x.src || null, guess: !!x.guess, turn: x.turn || 0, pred: x.pred || null, why: x.why || null, orbit: x.res.orbit || null, ring: x.res.plx && x.res.plx.ring ? x.res.plx.ring.slice() : null } : null)), cur: st.cur, flagged: e.flagged, round: e.round });
    }
  };
  const RW = window.Worker;
  class FW extends RW {
    set onmessage(fn) {
      super.onmessage = async (ev) => {
        const tSend = VT.t, fIdx = C.idx;
        // a width read wrong, simulated: one of the twelve measures of the face changed for a while
        if (window.__extErr && ev.data && ev.data.res && ev.data.res.ext) {
          const X = window.__extErr, e = ev.data.res.ext;
          if (VT.t >= X[0] && VT.t <= X[1] && e[X[2]] && typeof e[X[2]][X[3]] === 'number') e[X[2]][X[3]] += X[4];
        }
        // a layer left turned, simulated: the tiles of one row (or column) shifted for a while
        if (window.__mis && ev.data && ev.data.res && ev.data.res.edges && ev.data.res.edges.C) {
          const M = window.__mis, e = ev.data.res.edges;
          if (VT.t >= M[0] && VT.t <= M[1]) {
            const w = e.C.u1 - e.C.u0, h = e.C.v1 - e.C.v0;
            const RW = [["TL", "T", "TR"], ["L", "C", "R"], ["BL", "B", "BR"]], CL = [["TL", "L", "BL"], ["T", "C", "B"], ["TR", "R", "BR"]];
            for (const nm of (M[2] === 'rows' ? RW : CL)[M[3]]) { const b = e[nm]; if (!b) continue; if (M[2] === 'rows') { b.u0 += M[4] * w; b.u1 += M[4] * w; } else { b.v0 += M[4] * h; b.v1 += M[4] * h; } }
          }
        }
        if (window.__ori) window.__oriAt(((fIdx - 1) * 1000) / 30);   // the phone's orientation for the frame analysed
        VT.t += VT.delta;
        await C.ensure();
        fn(ev);
        try { window.__after(ev.data.res, tSend, fIdx); } catch (err) { window.__afterErr = String(err); }
        if (window.__shots && window.__shots.length && VT.t >= window.__shots[0] * 1000) {
          window.__shots.shift(); window.__shotReady = VT.t;
          await new Promise((res) => { window.__shotGo = res; });
        }
      };
    }
    get onmessage() { return super.onmessage; }
  }
  window.Worker = FW;
})();
````

## build_index.py

````python
"""Rebuild index.html from index_orig.html with a new core (script id=core-src) and a new app script."""
import re, sys
src, core_p, app_p, out = sys.argv[1:5]
html = open(src, encoding="utf-8").read()
ms = list(re.finditer(r"(<script([^>]*)>)(.*?)(</script>)", html, re.S))
assert len(ms) == 3 and 'core-src' in ms[1].group(2)
core = open(core_p, encoding="utf-8").read(); app = open(app_p, encoding="utf-8").read()
for body in (core, app):
    assert "</script" not in body
parts, pos = [], 0
for i, m in enumerate(ms):
    parts.append(html[pos:m.start(3)])
    parts.append(m.group(3) if i == 0 else "\n" + (core if i == 1 else app))
    pos = m.end(3)
parts.append(html[pos:])
open(out, "w", encoding="utf-8").write("".join(parts))
print("scritto", out, len("".join(parts)), "byte")
````

## estrai_parti.py

````python
"""Estrae da index.html le tre parti: cubejs.js, core.js (script id="core-src") e app.js,
piu' cubejs_node.js (cubejs utilizzabile da Node per test_decode.js).
uso: python3 estrai_parti.py index.html [cartella]"""
import os, re, sys
html = open(sys.argv[1], encoding="utf-8").read()
out = sys.argv[2] if len(sys.argv) > 2 else "."
parts = re.findall(r"<script([^>]*)>(.*?)</script>", html, re.S)
assert len(parts) == 3, len(parts)
names = ["cubejs.js", "core.js", "app.js"]
assert 'core-src' in parts[1][0]
for n, (_, body) in zip(names, parts):
    open(os.path.join(out, n), "w", encoding="utf-8").write(body.lstrip("\n"))
open(os.path.join(out, "cubejs_node.js"), "w", encoding="utf-8").write(parts[0][1].replace("require('./cube')", "module.exports"))
print("ok:", ", ".join(names + ["cubejs_node.js"]))
````

## gen_live.py

````python
"""Synthetic live scan of a scrambled mirror cube: the camera goes around the cube like a hand
turning it (holds, side tilts, quarter turns left/right/up/down, a half turn in one go, a face seen
again, a small roll, some tremor). Writes 720x1280 frames at 30 fps (every rendered frame twice)
and a truth file: for every frame, the face in front and the quarter turns of its photo.

usage: python3 gen_live.py <name> "<scramble>" <path code> [seed]
path code: comma separated tokens: H (hold), T<d><deg> (tilt out and back, d in l r u d),
M<d>[<d>...] (turn one quarter per letter, in one go), R<deg> (roll).
"""
import json, os, sys
import numpy as np
from PIL import Image, ImageFilter
import mcube
from mcube import MirrorCube

name, scramble, code = sys.argv[1], sys.argv[2], sys.argv[3]
seed = int(sys.argv[4]) if len(sys.argv) > 4 else 1
rng = np.random.default_rng(seed)
W, H, FPS = 720, 1280, 15
FOC, DIST = float(os.environ.get("FOC", 620)), float(os.environ.get("DIST", 210))
mcube.MARKS.clear()                      # the real cube has no paper marks
cube = MirrorCube().move(scramble)

def Rx(a): c, s = np.cos(a), np.sin(a); return np.array([[1, 0, 0], [0, c, -s], [0, s, c]])
def Ry(a): c, s = np.cos(a), np.sin(a); return np.array([[c, 0, s], [0, 1, 0], [-s, 0, c]])
def Rz(a): c, s = np.cos(a), np.sin(a); return np.array([[c, -s, 0], [s, c, 0], [0, 0, 1]])
# the front face ends up looking d (camera frame: x right, y up, z towards the camera)
TURN = {'l': (Ry, -1), 'r': (Ry, 1), 'd': (Rx, 1), 'u': (Rx, -1)}
ease = lambda u: 0.5 - 0.5 * np.cos(np.pi * np.clip(u, 0, 1))

segs, t = [], 0.0          # (t0, t1, fn(u) -> rotation applied on the left of the base, final base change)
base = np.eye(3); roll = 0.0
def about(axis_fn, ang):     # rotation about the cube's own (rolled) axis
    return Rz(roll) @ axis_fn(ang) @ Rz(-roll)
for tok in code.split(','):
    tok = tok.strip()
    if tok == 'H':
        segs.append((t, t + 1.4, lambda u, b=base: b)); t += 1.4
    elif tok[0] == 'T':
        d, deg = tok[1], np.radians(float(tok[2:]))
        fn, sg = TURN[d]
        r0 = roll
        def f(u, b=base, fn=fn, sg=sg, deg=deg, r0=r0):
            a = deg * sg * (ease(u / 0.35) if u < 0.35 else 1.0 if u < 0.65 else ease((1 - u) / 0.35))
            return Rz(r0) @ fn(a) @ Rz(-r0) @ b
        segs.append((t, t + 1.1, f)); t += 1.1
    elif tok[0] == 'M':
        ds = tok[1:]
        dur = 1.0 + 0.8 * (len(ds) - 1)
        r0 = roll
        def f(u, b=base, ds=ds, r0=r0):
            # quarter turns one after the other, one smooth motion
            q = ease(u) * len(ds); M = b
            for i, d in enumerate(ds):
                fn, sg = TURN[d]
                frac = min(1.0, max(0.0, q - i))
                M = Rz(r0) @ fn(sg * np.pi / 2 * frac) @ Rz(-r0) @ M
            return M
        segs.append((t, t + dur, f)); t += dur
        for d in ds:
            fn, sg = TURN[d]; base = about(fn, sg * np.pi / 2) @ base
        base = np.round(base * 1e9) / 1e9
    elif tok[0] == 'O':
        deg = np.radians(float(tok[1:])); r0 = roll
        def f(u, b=base, deg=deg, r0=r0):
            amp = deg * (ease(u / 0.12) if u < 0.12 else 1.0 if u < 0.88 else ease((1 - u) / 0.12))
            psi = 2 * np.pi * u
            ax = np.array([-np.sin(psi), np.cos(psi), 0.0])          # the normal leans towards psi
            K = np.array([[0, -ax[2], ax[1]], [ax[2], 0, -ax[0]], [-ax[1], ax[0], 0]])
            Rr = np.eye(3) + np.sin(amp) * K + (1 - np.cos(amp)) * K @ K
            return Rz(r0) @ Rr @ Rz(-r0) @ b
        segs.append((t, t + 3.2, f)); t += 3.2
    elif tok[0] == 'X':
        # the cube turned upside down in front of the phone (about the phone's horizontal axis):
        # the view changes, the phone does not move
        def f(u, b=base):
            return Rx(np.pi * ease(u)) @ b
        segs.append((t, t + 1.6, f, 'flip', base)); t += 1.6
        base = np.round((Rx(np.pi) @ base) * 1e9) / 1e9
    elif tok[0] == 'R':
        deg = np.radians(float(tok[1:])); r0 = roll
        segs.append((t, t + 0.8, lambda u, b=base, deg=deg: Rz(deg * ease(u)) @ b)); t += 0.8
        base = Rz(deg) @ base; roll += deg
segs.append((t, t + 1.5, lambda u, b=base: b)); t += 1.5
T_END = t

def pose(tt):
    for sg in segs:
        a, b, fn = sg[0], sg[1], sg[2]
        if tt <= b: return fn((tt - a) / (b - a))
    return segs[-1][2](1.0)
# where the cube is in the world, for the phone's orientation: it only changes during a flip
def cube_in_world(tt):
    W = np.eye(3)
    for sg in segs:
        a, b = sg[0], sg[1]
        if len(sg) > 3 and sg[3] == 'flip':
            M0, Mt = sg[4], sg[2](min(1.0, max(0.0, (tt - a) / (b - a))))
            if tt <= a: break
            W = W @ M0.T @ Mt                      # keeps W @ M^T (the phone) still during the flip
            if tt <= b: break
    return W

FACEN = {P: np.array(v[0]) for P, v in mcube.FACES.items()}
def truth_of(M):
    # face in front and quarter turns q of its photo (photo frame -> net view, as placeFace)
    zc = {P: (M @ n)[2] for P, n in FACEN.items()}
    P = max(zc, key=zc.get)
    ang = np.degrees(np.arccos(np.clip(zc[P], -1, 1)))
    r = M.T @ np.array([1.0, 0, 0]); u = M.T @ np.array([0, 1.0, 0])
    n0, rn, un = (np.array(v) for v in mcube.FACES[P])
    best = None
    for q in range(4):
        rr, uu = rn.copy(), un.copy()
        for _ in range(q): rr, uu = -uu, rr
        sc = rr @ r + uu @ u
        if best is None or sc > best[1]: best = (q, sc)
    return P, best[0], float(ang)

# what the phone's motion sensors would say (W3C deviceorientation: R = Rz(alpha) Rx(beta) Ry(gamma),
# device frame x right, y up, z out of the screen, which is the camera frame of this renderer).
# The earth frame is the cube turned by a fixed 40 degrees, far from the gimbal lock of the angles.
Q40 = Rx(np.radians(40))
def orientation_of(M, W=None):
    R = Q40 @ (W if W is not None else np.eye(3)) @ M.T
    if abs(R[2][1]) > 1 - 1e-9:
        b = np.degrees(np.arcsin(np.clip(R[2][1], -1, 1))); g = 0.0; a = np.degrees(np.arctan2(R[1][0], R[0][0]))
    else:
        b = np.degrees(np.arcsin(R[2][1])); g = np.degrees(np.arctan2(-R[2][0], R[2][2])); a = np.degrees(np.arctan2(-R[0][1], R[1][1]))
    return [round(float(a) % 360, 3), round(float(b), 3), round(float(g), 3)]

def heights_of(M, P):
    n0, rn, un = (np.array(v, float) for v in mcube.FACES[P])
    r = M.T @ np.array([1.0, 0, 0]); u = M.T @ np.array([0, 1.0, 0])
    ax = int(np.argmax(np.abs(n0))); sg = np.sign(n0[ax])
    out = {}
    on = [cb for cb in cube.cubies if any(tuple(int(x) for x in s[0]) == tuple(int(x) for x in n0) for s in cb['st'])]
    cen = [cb for cb in on if len(cb['st']) == 1]
    if not cen: return None
    c0 = (cen[0]['lo'] + cen[0]['hi']) / 2          # the middle layer is not at the middle of the cube
    for cb in on:
        c = (cb['lo'] + cb['hi']) / 2 - c0
        surf = cb['hi'][ax] if sg > 0 else -cb['lo'][ax]
        cr, cu = c @ r, c @ u
        col = 0 if cr < -cube.S * 0.1 else 2 if cr > cube.S * 0.1 else 1
        row = 0 if cu > cube.S * 0.1 else 2 if cu < -cube.S * 0.1 else 1
        out[[['TL', 'T', 'TR'], ['L', 'C', 'R'], ['BL', 'B', 'BR']][row][col]] = surf
    if 'C' not in out: return None
    return {k: round(float((v - out['C']) / cube.S * 100), 2) for k, v in out.items()}

def render_pose(M, seed):
    """Ray-cast the cube (fixed, axis aligned) from a camera at M^T (0,0,DIST): as mcube.render."""
    Rc = M.T
    trem = np.array([1.5 * np.sin(1.3 * seed / FPS * 6.1), 1.5 * np.cos(1.1 * seed / FPS * 5.3), 0.0])
    cam = Rc @ (np.array([0.0, 0.0, DIST]) + trem)
    f = FOC
    def proj(p):
        d = Rc.T @ (p - cam)
        return W / 2 + f * d[0] / -d[2], H / 2 - f * d[1] / -d[2]
    lo = np.array([c['lo'] for c in cube.cubies]); hi = np.array([c['hi'] for c in cube.cubies])
    depth = np.full((H, W), np.inf)
    kind = np.zeros((H, W), np.int8)
    lu = np.zeros((H, W)); lv = np.zeros((H, W)); shade = np.ones((H, W)); tone = np.ones((H, W))
    rim, rad = 0.0235 * cube.S, 0.03 * cube.S
    light = np.array([0.3, 0.5, 0.8]); light /= np.linalg.norm(light)
    rg = np.random.default_rng(7)
    for ci, cb in enumerate(cube.cubies):
        stick = {tuple(int(x) for x in s[0]) for s in cb['st']}
        brushed_u = rg.random() < 0.5
        tone_c = 0.92 + 0.12 * rg.random()
        for ax in range(3):
            for sgn in (-1, 1):
                pv = cb['hi'][ax] if sgn > 0 else cb['lo'][ax]
                ctr = (cb['lo'] + cb['hi']) / 2; ctr[ax] = pv
                nrm = np.zeros(3); nrm[ax] = sgn
                if np.dot(cam - ctr, nrm) <= 1e-9:
                    continue
                o = [a for a in range(3) if a != ax]
                corners = []
                for a_ in (cb['lo'][o[0]], cb['hi'][o[0]]):
                    for b_ in (cb['lo'][o[1]], cb['hi'][o[1]]):
                        p = np.zeros(3); p[ax] = pv; p[o[0]] = a_; p[o[1]] = b_
                        corners.append(proj(p))
                us = [c[0] for c in corners]; vs = [c[1] for c in corners]
                x0 = max(0, int(np.floor(min(us))) - 1); x1 = min(W, int(np.ceil(max(us))) + 1)
                y0 = max(0, int(np.floor(min(vs))) - 1); y1 = min(H, int(np.ceil(max(vs))) + 1)
                if x1 <= x0 or y1 <= y0:
                    continue
                ys, xs = np.mgrid[y0:y1, x0:x1].astype(np.float64)
                ray = np.stack([(xs + 0.5 - W / 2) / f, -(ys + 0.5 - H / 2) / f, -np.ones_like(xs)], -1) @ Rc.T
                with np.errstate(divide='ignore', invalid='ignore'):
                    tt = (pv - cam[ax]) / ray[..., ax]
                hit = cam + tt[..., None] * ray
                sub = depth[y0:y1, x0:x1]
                m = (tt > 0) & (tt < sub)
                for a in o:
                    m &= (hit[..., a] >= cb['lo'][a]) & (hit[..., a] <= cb['hi'][a])
                if not m.any():
                    continue
                sub[m] = tt[m]
                sh = 0.55 + 0.45 * max(0.0, float(np.dot(Rc.T @ nrm, light)))
                shade[y0:y1, x0:x1][m] = sh
                if tuple(int(x) for x in nrm) in stick:
                    a0, a1 = o
                    u = hit[..., a0][m] - cb['lo'][a0]; v = hit[..., a1][m] - cb['lo'][a1]
                    wu = cb['hi'][a0] - cb['lo'][a0]; wv = cb['hi'][a1] - cb['lo'][a1]
                    qu = np.clip(np.abs(u - wu / 2) - (wu / 2 - rim - rad), 0, None)
                    qv = np.clip(np.abs(v - wv / 2) - (wv / 2 - rim - rad), 0, None)
                    inside = (np.hypot(qu, qv) <= rad) & (np.abs(u - wu / 2) <= wu / 2 - rim) & (np.abs(v - wv / 2) <= wv / 2 - rim)
                    kind[y0:y1, x0:x1][m] = np.where(inside, 2, 1)
                    lu[y0:y1, x0:x1][m] = u if brushed_u else v
                    lv[y0:y1, x0:x1][m] = ci * 7.1 + ax
                    tone[y0:y1, x0:x1][m] = tone_c
                else:
                    kind[y0:y1, x0:x1][m] = 1
    r2 = np.random.default_rng(seed)
    img = np.empty((H, W, 3))
    ys, xs = np.mgrid[0:H, 0:W]
    g = ((xs % 70) < 3) | ((ys % 70) < 3)
    img[..., 0] = 150.0; img[..., 1] = 108.0; img[..., 2] = 96.0
    img += r2.normal(0, 5, (H, W, 1))
    img[g] = [185, 180, 172]
    black = kind == 1
    img[black] = np.array([20.0, 19.0, 21.0]) * shade[black][:, None] + 6
    gold = kind == 2
    stripes = np.sin(lu[gold] * 9.0 + lv[gold]) * 0.5 + np.sin(lu[gold] * 23.0 + 2 * lv[gold]) * 0.5
    fac = (tone[gold] * shade[gold] * (0.93 + 0.05 * stripes))[:, None]
    img[gold] = np.clip(np.array([232.0, 196.0, 92.0]) * fac + 18 * (shade[gold][:, None] - 0.8), 0, 255)
    img += r2.normal(0, 2.5, img.shape)
    return Image.fromarray(np.clip(img, 0, 255).astype(np.uint8)).filter(ImageFilter.GaussianBlur(0.8))

out = f"frames/{name}"
os.makedirs(out, exist_ok=True)
truth = []
n = int(T_END * FPS)
for i in range(n):
    tt = i / FPS
    M = pose(tt)
    # tremor: a small wobble of the hand
    M = Rx(np.radians(1.2 * np.sin(2.1 * tt))) @ Ry(np.radians(1.2 * np.sin(1.7 * tt + 1))) @ M
    P, q, ang = truth_of(M)
    if os.environ.get("TRUTH_ONLY"):
        truth.append({"t": round(tt * 1000), "P": P, "q": q, "ang": round(ang, 1), "z": heights_of(M, P) if ang < 45 else None, "ori": orientation_of(M, cube_in_world(tt))})
        continue
    im = render_pose(M, i)
    for k in (0, 1):
        idx = 2 * i + k + 1
        im.save(f"{out}/{idx:05d}.jpg", quality=88) if k == 0 else os.link(f"{out}/{2 * i + 1:05d}.jpg", f"{out}/{idx:05d}.jpg")
    truth.append({"t": round(tt * 1000), "P": P, "q": q, "ang": round(ang, 1), "z": heights_of(M, P) if ang < 45 else None, "ori": orientation_of(M, cube_in_world(tt))})
json.dump({"scramble": scramble, "code": code, "truth": truth, "facelets": cube.facelets()}, open(f"{name}_truth.json", "w"))
print(f"{name}: {n} fotogrammi ({2 * n} a 30 fps), {T_END:.1f} s")
````

## eval_syn.js

````javascript
// Synthetic live scan: every confirmation against the face really in front, and the final state
// against the scramble (truth.js). usage: node eval_syn.js replay.json truth.json
const fs = require("fs");
const { compare } = require("./truth.js");
const L = JSON.parse(fs.readFileSync(process.argv[2], "utf8").replace(/-?Infinity/g, "1e999").replace(/NaN/g, "null"));
const T = JSON.parse(fs.readFileSync(process.argv[3], "utf8"));
const NAMES = ["Davanti", "Destra", "Dietro", "Sinistra", "Sopra", "Sotto"], FACES = "FRBLUD";
const truthAt = (t) => { let b = T.truth[0]; for (const x of T.truth) if (x.t <= t) b = x; return b; };
const r = L.result;
console.log(`${L.vid} ${L.delta} ms: fine phase=${r.phase} pill="${r.pill}" round=${r.round} errori ${JSON.stringify(L.errors)}`);
let prev = [null, null, null, null, null, null], nOk = 0, nConf = 0;
for (const ev of L.events) {
  ev.slots.forEach((s, j) => {
    const a = JSON.stringify(prev[j] && prev[j].ext), b = JSON.stringify(s && s.ext);
    if (a === b) return;
    if (!s) { console.log(`  ${(ev.t / 1000).toFixed(2)}s casella ${NAMES[j]} svuotata`); return; }
    const tr = truthAt(ev.t), good = FACES[j] === tr.P && (s.turn || 0) === tr.q;
    nConf++; if (good) nOk++;
    console.log(`  ${(ev.t / 1000).toFixed(2)}s ${NAMES[j].padEnd(8)} [${(s.src || "-").padEnd(7)}] giro ${s.turn}  vero: ${NAMES[FACES.indexOf(tr.P)]} giro ${tr.q}  ${good ? "OK" : "SBAGLIATO"}${s.pred ? `  (movimento: ${NAMES[s.pred.slot]} giro ${s.pred.turn} [${s.pred.steps}] conf ${s.pred.conf})` : ""}`);
  });
  prev = ev.slots;
}
console.log(`  conferme giuste ${nOk}/${nConf}`);
const chk = (lab, d) => { if (!d || !d.facelets) { console.log(`  ${lab}: nessuno stato`); return; } const c = compare(d.facelets, d.thick, T.scramble); console.log(`  ${lab}: tasselli sbagliati ${c.bad}/48${d.pairCost != null ? `, costo angoli ${d.pairCost.toFixed(1)}` : ""}${d.hinted != null ? `, disposizione seguita ${d.hinted ? "usata" : "scartata"} (seguita ${d.hintPairCost != null ? d.hintPairCost.toFixed(1) : "-"}, libera ${d.freePairCost != null ? d.freePairCost.toFixed(1) : "-"})` : ""}`); };
if (r.decoded) chk("stato della pagina", Object.assign({}, r.decoded, { pairCost: r.decoded.pairCost }));
if (r.forced && typeof r.forced === "object") { chk("decodifica con la disposizione seguita", r.forced.hint); chk("decodifica libera", r.forced.free); }
else console.log(`  decodifica forzata: ${JSON.stringify(r.forced)}`);
console.log(`  mosse: ${r.moves ? r.moves.length + " (" + r.moves.join(" ") + ")" : "-"}`);
````

## truth.js

````javascript
// Compare a decoded state with the true state of a known scramble, up to how the cube was held
// (24 orientations while scrambling x 24 orientations of the scan frame).
const Cube = require(require("path").join(__dirname, "cubejs_node.js"));
const TH = { U: 16.8, D: 49.6, F: 22.7, B: 43.7, R: 29.4, L: 36.9 };
const ORDER = "URFDLB";
const FACES = { U: [[0, 1, 0], [1, 0, 0], [0, 0, -1]], R: [[1, 0, 0], [0, 0, -1], [0, 1, 0]], F: [[0, 0, 1], [1, 0, 0], [0, 1, 0]],
  D: [[0, -1, 0], [1, 0, 0], [0, 0, 1]], L: [[-1, 0, 0], [0, 0, 1], [0, 1, 0]], B: [[0, 0, -1], [-1, 0, 0], [0, 1, 0]] };
const add = (a, b) => a.map((x, i) => x + b[i]), mul = (a, k) => a.map((x) => x * k), dot = (a, b) => a[0] * b[0] + a[1] * b[1] + a[2] * b[2];
const pos = [], nrm = [];
for (const P of ORDER) for (let r = 0; r < 3; r++) for (let c = 0; c < 3; c++) { const [n, rt, up] = FACES[P]; pos.push(add(add(n, mul(rt, c - 1)), mul(up, 1 - r))); nrm.push(n); }
const mats = [];
for (const p of [[0, 1, 2], [0, 2, 1], [1, 0, 2], [1, 2, 0], [2, 0, 1], [2, 1, 0]]) for (let sg = 0; sg < 8; sg++) {
  const M = [0, 1, 2].map((i) => { const row = [0, 0, 0]; row[p[i]] = sg & (1 << i) ? -1 : 1; return row; });
  const det = M[0][0] * (M[1][1] * M[2][2] - M[1][2] * M[2][1]) - M[0][1] * (M[1][0] * M[2][2] - M[1][2] * M[2][0]) + M[0][2] * (M[1][0] * M[2][1] - M[1][1] * M[2][0]);
  if (det === 1) mats.push(M);
}
const app = (M, v) => M.map((row) => dot(row, v));
const faceOf = (n) => ORDER.split("").find((P) => dot(FACES[P][0], n) === 1);
const idxOf = (p, n) => { const P = faceOf(n), [nn, rt, up] = FACES[P], d = add(p, mul(nn, -1)); return ORDER.indexOf(P) * 9 + (1 - dot(d, up)) * 3 + (dot(d, rt) + 1); };
const perm = mats.map((M) => pos.map((p, i) => idxOf(app(M, p), app(M, nrm[i]))));   // facelet i -> perm[i]
const relab = mats.map((M) => { const m = {}; for (const P of ORDER) m[P] = faceOf(app(M, FACES[P][0])); return m; });
function compare(decFacelets, decThick, scramble) {
  const c = new Cube(); c.move(scramble); const tp = c.asString();
  const dec = decFacelets.split("").map((x) => decThick[x]);
  let best = null;
  for (const q of relab) {                      // physical face in each holding position
    const th = tp.split("").map((x) => TH[q[x]]);
    for (const pm of perm) {                    // holding frame -> scan frame
      const sc = new Array(54); th.forEach((v, i) => { sc[pm[i]] = v; });
      let bad = 0; const where = [];
      for (let i = 0; i < 54; i++) if (i % 9 !== 4 && Math.abs(sc[i] - dec[i]) > 1) { bad++; where.push(i); }
      if (!best || bad < best.bad) best = { bad, where, truth: sc };
    }
  }
  return best;
}
module.exports = { compare };
if (require.main === module) {
  const fs = require("fs");
  const L = JSON.parse(fs.readFileSync(process.argv[2], "utf8").replace(/-?Infinity/g, "1e999").replace(/NaN/g, "null"));
  const d = L.result.decoded;
  if (!d) { console.log("nessuno stato decodificato"); process.exit(0); }
  const b = compare(d.facelets, d.thick, process.argv[3] || "R U F' L2 D B' R2 U'");
  console.log("facelets sbagliati (su 48):", b.bad, "a", b.where.map((i) => "URFDLB"[Math.floor(i / 9)] + (i % 9)).join(" "));
}
````

## summ.js

````javascript
// Summary of a replay of a scrambled cube: confirmations, what changed, tips, end.
const fs = require("fs");
const L = JSON.parse(fs.readFileSync(process.argv[2], "utf8").replace(/-?Infinity/g, "1e999").replace(/NaN/g, "null"));
const NAMES = ["Davanti", "Destra", "Dietro", "Sinistra", "Sopra", "Sotto"];
const r = L.result;
console.log(`video ${L.vid}, delta ${L.delta} ms, analisi ${L.log.length}, errori ${JSON.stringify(L.errors)}`);
console.log(`fine: phase=${r.phase} pill="${r.pill}" round=${r.round} rechecks=${JSON.stringify(r.rechecks)} flagged=${JSON.stringify(r.flagged)}`);
let prev = [null, null, null, null, null, null];
for (const ev of L.events) {
  ev.slots.forEach((s, j) => {
    const a = JSON.stringify(prev[j] && prev[j].ext), b = JSON.stringify(s && s.ext);
    if (a !== b) console.log(`t=${(ev.t / 1000).toFixed(2)}s casella ${j + 1} (${NAMES[j]}) ${s ? `<- faccia${s.src ? " [" + s.src + "]" : ""}${s.guess ? " (provvisoria)" : ""} giro ${s.turn}${s.pred ? ` previsione ${NAMES[s.pred.slot]} giro ${s.pred.turn} [${s.pred.steps}] conf ${s.pred.conf}` : ""}` : "svuotata"} round=${ev.round} flagged=${JSON.stringify(ev.flagged)}`);
  });
  prev = ev.slots;
}
if (r.decoded) console.log(`decodifica: ${r.decoded.facelets} peggiore ${r.decoded.worst && r.decoded.worst.toFixed(1)} margine ${r.decoded.margin && r.decoded.margin.toFixed(0)} costo angoli ${r.decoded.pairCost && r.decoded.pairCost.toFixed(1)} (secondo ${r.decoded.secondPairCost && r.decoded.secondPairCost.toFixed(1)}) plx ${JSON.stringify(r.decoded.plx && { n: r.decoded.plx.n, bad: r.decoded.plx.bad.length })}`);
console.log(`mosse: ${JSON.stringify(r.moves)}`);
// tips timeline (changes only)
if (process.argv[3]) { let last = ""; for (const e of L.log) { if (e.tip !== last) { console.log(`  ${(e.t / 1000).toFixed(2)} ${e.tip}`); last = e.tip; } } }
````

## analyze.js

````javascript
// Reads a replay log and explains what happened, face by face.
// Every reading of the solved cube is matched against the six faces of the solved model
// (reference thicknesses THICK0), in all four rotations.
const fs = require("fs");
const MV = require(require("path").resolve(process.env.CORE || "core.js"));
const TH = { U: 16.8, D: 49.6, F: 22.7, B: 43.7, R: 29.4, L: 36.9 };
const NAMES = ["Davanti", "Destra", "Dietro", "Sinistra", "Sopra", "Sotto"];
const EXP = {};
for (const P of "URFDLB") {
  const ext = {};
  for (const nm in MV.TARGET[P]) for (const d in MV.TARGET[P][nm]) (ext[nm] = ext[nm] || {})[d] = TH["URFDLB"[Math.floor(MV.TARGET[P][nm][d] / 9)]];
  EXP[P] = ext;
}
function classify(ext) {
  if (!ext) return null;
  const res = [];
  for (const P of "URFDLB") for (let k = 0; k < 4; k++) {
    const ex = MV.rotateReading({ ext: EXP[P], sizes: {} }, k).ext;
    let s = 0, n = 0, worst = 0;
    for (const nm in ex) for (const d in ex[nm]) { const x = ext[nm] && ext[nm][d]; if (typeof x === "number") { const e = Math.abs(x - ex[nm][d]); s += e; n++; worst = Math.max(worst, e); } }
    if (n >= 6) res.push({ P, k, v: s / n, n, worst });
  }
  res.sort((a, b) => a.v - b.v);
  if (!res.length) return null;
  const best = res[0], second = res.find((q) => q.P !== best.P);
  return { P: best.P, k: best.k, v: +best.v.toFixed(2), worst: +best.worst.toFixed(1), n: best.n, second: second && second.P, gap: second ? +(second.v - best.v).toFixed(2) : 99 };
}
const fmtExt = (e) => { if (!e) return "-"; const o = []; for (const nm of ["TL", "T", "TR", "L", "R", "BL", "B", "BR"]) for (const d in e[nm] || {}) o.push(`${nm}.${d[0]}${e[nm][d].toFixed(1)}`); return o.join(" "); };
module.exports = { classify, EXP, fmtExt };
if (require.main === module) {
  const L = JSON.parse(fs.readFileSync(process.argv[2], "utf8").replace(/-?Infinity/g, "1e999").replace(/NaN/g, "null"));
  const r = L.result;
  console.log(`video ${L.vid}, delta ${L.delta} ms, analyses ${L.log.length}, errors ${JSON.stringify(L.errors)}`);
  console.log(`end: phase=${r.phase} pill="${r.pill}" round=${r.round} rechecks=${JSON.stringify(r.rechecks)} flagged=${JSON.stringify(r.flagged)}`);
  // confirmations: slots whose content changes
  let prev = [null, null, null, null, null, null];
  for (const ev of L.events) {
    ev.slots.forEach((s, j) => {
      const a = JSON.stringify(prev[j] && prev[j].ext), b = JSON.stringify(s && s.ext);
      if (a !== b && s) {
        const c = classify(s.ext);
        console.log(`t=${(ev.t / 1000).toFixed(2)}s f=${ev.f} slot ${j + 1} (${NAMES[j]}) <- face ${c.P} rot ${c.k} err ${c.v} (worst ${c.worst}, 2nd ${c.second} +${c.gap}) round=${ev.round} flagged=${JSON.stringify(ev.flagged)}`);
        console.log(`      ${fmtExt(s.ext)}`);
      }
    });
    prev = ev.slots;
  }
  if (r.slots) {
    console.log("final slots:");
    r.slots.forEach((s, j) => { const c = s && s.ext ? classify(s.ext) : null; console.log(`  ${j + 1}. ${NAMES[j]}: ${c ? `${c.P} rot ${c.k} err ${c.v} worst ${c.worst} (2nd ${c.second} +${c.gap}) plx ${s.plx}` : "-"}`); });
    for (let i = 0; i < 6; i++) for (let j = i + 1; j < 6; j++) {
      const a = r.slots[i], b = r.slots[j];
      if (!a || !b || !a.ext || !b.ext) continue;
      const sf = MV.sameFace({ ext: a.ext }, { ext: b.ext }, 6), ss = MV.sameFaceStrict({ ext: a.ext, sizes: {} }, { ext: b.ext, sizes: {} });
      if (sf < 5 || ss) console.log(`  pair ${i + 1}-${j + 1}: sameFace ${sf.toFixed(2)} strict ${ss}`);
    }
  }
  if (r.decoded) console.log(`decoded: ${r.decoded.facelets} thick ${JSON.stringify(r.decoded.thick)} worst ${r.decoded.worst && r.decoded.worst.toFixed(1)} margin ${r.decoded.margin && r.decoded.margin.toFixed(0)} placed ${JSON.stringify(r.decoded.placed)} plx ${JSON.stringify(r.decoded.plx && { n: r.decoded.plx.n, bad: r.decoded.plx.bad.length })}`);
  console.log(`moves: ${JSON.stringify(r.moves)}`);
}
````

## timeline.js

````javascript
// Compact timeline of a replay: one line per run of similar analyses.
const fs = require("fs");
const { classify } = require("./analyze.js");
const L = JSON.parse(fs.readFileSync(process.argv[2], "utf8").replace(/-?Infinity/g, "1e999").replace(/NaN/g, "null"));
const from = +(process.argv[3] || 0) * 1000, to = +(process.argv[4] || 1e9) * 1000;
let run = null;
const flush = () => { if (run) console.log(`${(run.t0 / 1000).toFixed(2)}-${(run.t1 / 1000).toFixed(2)}s x${run.n} ${run.key} | ${run.tip}`); };
for (const e of L.log) {
  if (e.t < from || e.t > to) continue;
  const c = e.ext ? classify(e.ext) : null;
  const read = e.ok ? `ok ${c ? c.P + c.k + " e" + c.v.toFixed(1) : "?"}` : `no:${e.why}${c ? " (" + c.P + c.k + " e" + c.v.toFixed(1) + ")" : ""}`;
  const key = `${read.replace(/ e[0-9.]+/, "")} cur=${e.cur + 1} done=${e.done.map((x) => (x ? 1 : 0)).join("")} fl=${e.flagged ? e.flagged.map((x) => x + 1).join(",") : "-"}${e.turn ? " TURN" : ""}`;
  if (run && run.key === key && run.tip === e.tip) { run.n++; run.t1 = e.t; continue; }
  flush();
  run = { key, tip: e.tip, t0: e.t, t1: e.t, n: 1 };
}
flush();
````

## test_decode.js

````javascript
// Decode test without images: readings computed from random cube states (reference layer
// thicknesses, measure noise, perspective size ratios), faces given in random order and turned
// at random. Checks decodeFree against the true state; runs on the old and on the new core.
const path = require("path");
const cores = process.argv.slice(2).map((p) => path.resolve(p));
if (!cores.length) { console.log("uso: node test_decode.js core_vecchio.js core_nuovo.js"); process.exit(1); }
const Cube = require(path.join(__dirname, "cubejs_node.js"));
const TH = { U: 16.8, D: 49.6, F: 22.7, B: 43.7, R: 29.4, L: 36.9 };
const CLS = { A: [16.8, 49.6], B: [22.7, 43.7], C: [29.4, 36.9] };
let seed = 12345; const rnd = () => ((seed = (seed * 1103515245 + 12345) % 2147483648) / 2147483648);
const gauss = () => { let u = 0, v = 0; while (!u) u = rnd(); while (!v) v = rnd(); return Math.sqrt(-2 * Math.log(u)) * Math.cos(2 * Math.PI * v); };
function readings(MV, fac, noise) {
  const out = {};
  for (const P of "URFDLB") {
    const ext = {};
    for (const nm in MV.TARGET[P]) for (const d in MV.TARGET[P][nm]) (ext[nm] = ext[nm] || {})[d] = TH[fac[MV.TARGET[P][nm][d]]] + noise * gauss();
    const f = "URFDLB".indexOf(P), sizes = {};
    for (const [k, pos] of [["T", 1], ["B", 7], ["L", 3], ["R", 5]]) sizes[k] = { c: Math.exp(0.003 * (TH[P] - TH[fac[f * 9 + pos]]) + 0.004 * gauss()), e: 1 };
    out[P] = { ext, sizes, ok: true };
  }
  return out;
}
function run(corePath, n, noise, solvedOnly) {
  delete require.cache[require.resolve(corePath)];
  const MV = require(corePath);
  seed = 777 + Math.round(noise * 10) + (solvedOnly ? 5 : 0);
  let exact = 0, solvedOk = 0, lowMargin = 0;
  for (let i = 0; i < n; i++) {
    const mr = Math.random; Math.random = rnd;
    const fac = solvedOnly ? "UUUUUUUUURRRRRRRRRFFFFFFFFFDDDDDDDDDLLLLLLLLLBBBBBBBBB" : Cube.random().asString();
    Math.random = mr;
    const R = readings(MV, fac, noise);
    const others = ["R", "B", "L", "U", "D"].sort(() => rnd() - 0.5);
    const list = [R.F].concat(others.map((P) => MV.rotateReading(R[P], Math.floor(rnd() * 4))));
    const d = MV.decodeFree(list, CLS);
    if (d.facelets === fac) exact++;
    if (d.frame.margin < 20) lowMargin++;
  }
  return { exact, lowMargin };
}
for (const [noise, n, solved] of [[1.0, 48, false], [2.0, 48, false], [1.0, 20, true], [2.0, 20, true]]) {
  const line = cores.map((c) => { const r = run(c, n, noise, solved); return `${path.basename(c)}: ${r.exact}/${n} exact, margin<20 ${r.lowMargin}`; });
  console.log(`${solved ? "solved   " : "scrambled"} noise ${noise}:  ${line.join("  |  ")}`);
}
````

## test_follow.js

````javascript
// followTurn on the changes of face of a replay log of the SOLVED cube (truth from the model).
const fs = require("fs");
const MV = require(require("path").resolve(process.env.CORE || "core_new.js"));
const { classify } = require("./analyze.js");
const L = JSON.parse(fs.readFileSync(process.argv[2], "utf8").replace(/-?Infinity/g, "1e999").replace(/NaN/g, "null"));
const entry = (e) => MV.viewEntry({ sizes: e.sizes, g: e.g, h: e.h, theta: e.th, edges: e.edges }, e.t, e.cen ? e.cen[0] : 0, e.cen ? e.cen[1] : 0, e.cen ? 0.5 * (e.cen[2] + e.cen[3]) : 0);
const rows = L.log.map((e) => {
  const en = entry(e);
  const c = e.ext ? classify(e.ext) : null;
  const front = en.asp && Math.abs(en.asp - 1) < 0.05 && en.x != null && en.y != null && Math.abs(en.x) < 0.03 && Math.abs(en.y) < 0.03;
  return { e, en, c: c && front && c.v < 2.6 && c.gap > 1 ? c : null };
});
const runs = [];
for (const r of rows) {
  if (!r.c) continue;
  const last = runs[runs.length - 1];
  if (last && last.P === r.c.P && last.k === r.c.k && r.e.t - last.t1 < 1500) { last.t1 = r.e.t; last.n++; last.rs.push(r.en); continue; }
  runs.push({ P: r.c.P, k: r.c.k, t0: r.e.t, t1: r.e.t, n: 1, rs: [r.en] });
}
const med = (a) => { const b = a.filter((x) => x != null).sort((p, q) => p - q); return b.length ? b[b.length >> 1] : 0; };
const bias = (rs) => ({ x: med(rs.map((r) => r.x)), y: med(rs.map((r) => r.y)) });
const big = runs.filter((r) => r.n >= 3);
let ok = 0, tot = 0, wrong = 0;
for (let i = 1; i < big.length; i++) {
  const a = big[i - 1], b = big[i];
  if (a.P === b.P && a.k === b.k) continue;
  const seq = rows.filter((r) => r.e.t >= a.t1 && r.e.t <= b.t0).map((r) => r.en);
  const f = MV.followTurn(seq, { biasA: bias(a.rs.slice(-8)), biasB: bias(b.rs.slice(0, 8)) });
  let fr = MV.photoFrame(a.P, (4 - a.k) % 4), pred = null;
  if (f && f.steps.length && f.conf > 0) { for (const d of f.steps) fr = MV.turnFrame(fr, d); const ff = MV.frameFace(fr); pred = { P: ff.P, q: (((ff.q - f.roll) % 4) + 4) % 4 }; }
  const truth = { P: b.P, q: (4 - b.k) % 4 };
  const good = pred && pred.P === truth.P && pred.q === truth.q;
  tot++; if (good) ok++; else if (pred) wrong++;
  if (process.env.DBG) console.log(JSON.stringify(f && f.info));
  console.log(`${a.P}${a.k}->${b.P}${b.k} ${(a.t1 / 1000).toFixed(2)}-${(b.t0 / 1000).toFixed(2)}: ${f ? `steps [${f.steps.join(",")}] conf ${f.conf} roll ${f.roll}` : "null"} -> ${pred ? pred.P + pred.q : "non so"} (vero ${truth.P}${truth.q}) ${good ? "OK" : pred ? "SBAGLIATO" : "-"}`);
}
console.log(`${ok}/${tot} giusti, ${wrong} sbagliati, ${tot - ok - wrong} senza risposta`);
````

## test_follow_syn.js

````javascript
// followTurn on the synthetic scan: frontal runs and faces from the truth file.
const fs = require("fs");
const MV = require(require("path").resolve(process.env.CORE || "core_new.js"));
const L = JSON.parse(fs.readFileSync(process.argv[2], "utf8").replace(/-?Infinity/g, "1e999").replace(/NaN/g, "null"));
const T = JSON.parse(fs.readFileSync(process.argv[3], "utf8"));
function alignTo(a, b) {   // as in the app
  if (!a || !b || !a.ext || !b.ext) return -1;
  let best = -1, bs = Infinity;
  for (let k = 0; k < 4; k++) {
    const rb = k ? MV.rotateReading(b, k).ext : b.ext, d = [];
    for (const nm in a.ext) for (const dd in a.ext[nm]) { const y = rb[nm] && rb[nm][dd]; if (typeof y === "number") d.push(Math.abs(a.ext[nm][dd] - y)); }
    if (d.length < 8) continue;
    const s = d.reduce((p, q) => p + q, 0) / d.length;
    if (d.filter((x) => x > 4).length <= 1 && Math.max(...d) <= 7 && s < 2.5 && s < bs) { bs = s; best = k; }
  }
  return best;
}
const same = (a, b) => (!a || !b || !a.ext || !b.ext ? null : alignTo(a, b) >= 0);
const truthAt = (t) => { let b = T.truth[0]; for (const x of T.truth) if (x.t <= t) b = x; return b; };
const rows = L.log.map((e) => {
  const en = MV.viewEntry({ sizes: e.sizes, g: e.g, h: e.h, theta: e.th, edges: e.edges }, e.t, e.cen ? e.cen[0] : 0, e.cen ? e.cen[1] : 0, e.cen ? 0.5 * (e.cen[2] + e.cen[3]) : 0);
  en.res = e.ok && e.ext ? { ext: e.ext, sizes: e.sizes } : null;
  const tr = truthAt(e.t);
  return { e, en, tr, front: tr.ang < 6 && e.ok && en.asp > 0.96 && en.asp < 1.04 };
});
const runs = [];
for (const r of rows) {
  if (!r.front) continue;
  const last = runs[runs.length - 1];
  if (last && last.P === r.tr.P && last.q === r.tr.q && r.e.t - last.t1 < 800) { last.t1 = r.e.t; last.n++; continue; }
  runs.push({ P: r.tr.P, q: r.tr.q, t0: r.e.t, t1: r.e.t, n: 1 });
}
const big = runs.filter((r) => r.n >= 3);
let ok = 0, tot = 0, wrong = 0;
for (let i = 1; i < big.length; i++) {
  const a = big[i - 1], b = big[i];
  if (a.P === b.P && a.q === b.q) continue;
  const seq = rows.filter((r) => r.e.t >= a.t1 && r.e.t <= b.t0).map((r) => r.en);
  const f = MV.followTurn(seq, { sameFace: same });
  let fr = MV.photoFrame(a.P, a.q), pred = null;
  if (f && f.steps.length && f.conf > 0) { for (const d of f.steps) fr = MV.turnFrame(fr, d); const ff = MV.frameFace(fr); pred = { P: ff.P, q: (((ff.q - f.roll) % 4) + 4) % 4 }; }
  const good = pred && pred.P === b.P && pred.q === b.q;
  tot++; if (good) ok++; else if (pred) wrong++;
  console.log(`${a.P}${a.q}->${b.P}${b.q} ${(a.t1 / 1000).toFixed(2)}-${(b.t0 / 1000).toFixed(2)}: ${f ? `[${f.steps.join(",")}] conf ${f.conf} roll ${f.roll}${f.same ? " stessa" : ""}` : "null"} -> ${pred ? pred.P + pred.q : "non so"} ${good ? "OK" : pred ? "SBAGLIATO" : "-"}`);
  if (process.env.DBG && !good) console.log("   ", JSON.stringify(f && f.info).slice(0, 600));
}
console.log(`${ok}/${tot} giusti, ${wrong} sbagliati, ${tot - ok - wrong} senza risposta`);
````

## test_solve.py

````python
"""The solution page: from a known scrambled state, press «Avanti» and check that the text and
the picture of the move change at every step. usage: python3 test_solve.py index.html"""
import sys, os, hashlib, http.server, threading, functools, shutil
from playwright.sync_api import sync_playwright
src = sys.argv[1]
html = open(src, encoding="utf-8").read()
html = html.replace("const st = { phase:", "const st = window.__st = { phase:", 1)
html = html.replace("  function solveNow() {", "  window.__solveNow = () => solveNow();\n  function solveNow() {", 1)
html = html.replace("  function build3D() {", "  function build3D() { window.__b3 = (window.__b3 || 0) + 1; try { window.__b3state = stateAtStep(); } catch (e) { window.__b3state = String(e); }", 1)
d = "/tmp/srv_solve"; os.makedirs(d, exist_ok=True)
if os.environ.get("THREE_JS"): open(os.path.join(d, "three.min.js"), "wb").write(open(os.environ["THREE_JS"], "rb").read())
open(os.path.join(d, "index.html"), "w", encoding="utf-8").write(html)
class Q(http.server.SimpleHTTPRequestHandler):
    def log_message(self, *a): pass
srv = http.server.ThreadingHTTPServer(("127.0.0.1", 0), functools.partial(Q, directory=d))
threading.Thread(target=srv.serve_forever, daemon=True).start()
fac = open("/tmp/scr.txt").read().strip()
errs = []
with sync_playwright() as p:
    b = p.chromium.launch(); pg = b.new_page(viewport={"width": 390, "height": 844})
    pg.on("pageerror", lambda e: errs.append(str(e)))
    three = os.environ.get("THREE_JS")
    pg.route("https://cdnjs.cloudflare.com/**", (lambda r: r.fulfill(path=three, content_type="application/javascript")) if three else (lambda r: r.abort()))
    pg.goto(f"http://127.0.0.1:{srv.server_address[1]}/index.html"); pg.wait_for_timeout(800)
    pg.evaluate("""(f) => { const st = window.__st; st.decoded = { facelets: f, thick: { U: 16.8, D: 49.6, F: 22.7, B: 43.7, R: 29.4, L: 36.9 }, worst: 1, persp: { n: 0, bad: [] }, plx: { n: 0, bad: [] }, frame: { margin: 100 }, slotOf: () => 0, worstFrom: 'F' }; window.__solveNow(); }""", fac)
    pg.wait_for_timeout(6000)
    seen = []
    for i in range(6):
        txt = pg.evaluate("[document.querySelector('#moveCount').textContent, document.querySelector('#moveCode').textContent, document.querySelector('#moveText').textContent]")
        img = pg.evaluate("document.querySelector('#art').innerHTML")
        b3 = pg.evaluate("[window.__b3 || 0, (window.__b3state || '').slice(0, 54)]")
        seen.append((txt, hashlib.md5((img + b3[1]).encode()).hexdigest()[:8], b3))
        pg.click("#btnNext"); pg.wait_for_timeout(400)
    pill = pg.evaluate("document.querySelector('#pillText').textContent")
    b.close()
srv.shutdown()
for t, h, b3 in seen: print(f"  {t[0]:<14} {t[1]:<4} figura {h}  vista 3D costruita {b3[0]} volte, stato {hashlib.md5(b3[1].encode()).hexdigest()[:6]}  «{t[2]}»")
print("stati 3D tutti diversi:", len(set(b[1] for _, _, b in seen)) == len(seen), "| figure tutte diverse:", len(set(h for _, h, _ in seen)) == len(seen), "| stato in alto:", pill, "| errori JavaScript:", errs or "nessuno")
````

## test_finish.py

````python
"""End of the solution: after the last move the button «Verifica che sia risolto» shows.
usage: python3 test_finish.py index.html"""
import sys, os, http.server, threading, functools
from playwright.sync_api import sync_playwright
html = open(sys.argv[1], encoding="utf-8").read()
html = html.replace("const st = { phase:", "const st = window.__st = { phase:", 1)
html = html.replace("  function solveNow() {", "  window.__solveNow = () => solveNow();\n  function solveNow() {", 1)
d = "/tmp/srv_fin"; os.makedirs(d, exist_ok=True)
if os.environ.get("THREE_JS"): open(os.path.join(d, "three.min.js"), "wb").write(open(os.environ["THREE_JS"], "rb").read()); open(os.path.join(d, "index.html"), "w", encoding="utf-8").write(html)
class Q(http.server.SimpleHTTPRequestHandler):
    def log_message(self, *a): pass
srv = http.server.ThreadingHTTPServer(("127.0.0.1", 0), functools.partial(Q, directory=d)); threading.Thread(target=srv.serve_forever, daemon=True).start()
fac = open("/tmp/scr.txt").read().strip(); errs = []
with sync_playwright() as p:
    b = p.chromium.launch(); pg = b.new_page(viewport={"width": 390, "height": 844}); pg.on("pageerror", lambda e: errs.append(str(e)))
    pg.route("https://cdnjs.cloudflare.com/**", lambda r: r.abort())
    pg.goto(f"http://127.0.0.1:{srv.server_address[1]}/index.html"); pg.wait_for_timeout(800)
    pg.evaluate("""(f) => { const st = window.__st; st.decoded = { facelets: f, thick: { U: 16.8, D: 49.6, F: 22.7, B: 43.7, R: 29.4, L: 36.9 }, worst: 1, persp: { n: 0, bad: [] }, plx: { n: 0, bad: [] }, frame: { margin: 100 }, slotOf: () => 0, worstFrom: 'F' }; window.__solveNow(); }""", fac)
    pg.wait_for_timeout(5000)
    n = pg.evaluate("window.__st.sol.moves.length")
    vis0 = pg.evaluate("[...document.querySelectorAll('button')].some((x) => x.textContent === 'Verifica che sia risolto' && getComputedStyle(x).display !== 'none')")
    for _ in range(n): pg.click("#btnNext"); pg.wait_for_timeout(120)
    txt = pg.evaluate("document.querySelector('#moveText').textContent")
    vis1 = pg.evaluate("[...document.querySelectorAll('button')].some((x) => x.textContent === 'Verifica che sia risolto' && getComputedStyle(x).display !== 'none')")
    b.close()
srv.shutdown()
print(f"mosse {n}: pulsante visibile durante le mosse {vis0}, alla fine {vis1} | testo finale: «{txt}» | errori JavaScript: {errs or 'nessuno'}")
````

## test_android.py

````python
"""The page as an app: camera error messages, fonts from the site, camera stopped and opened again when
another app comes in front, back button, working offline. usage: python3 test_android.py index.html assetsdir"""
import sys, os, shutil, http.server, threading, functools, json
from playwright.sync_api import sync_playwright
src, assets = sys.argv[1], sys.argv[2]
d = "/tmp/srv_android"; shutil.rmtree(d, ignore_errors=True); os.makedirs(d)
html = open(src, encoding="utf-8").read().replace("const live = { on:", "const live = window.__live = { on:", 1).replace("const st = { phase:", "const st = window.__st = { phase:", 1)
open(os.path.join(d, "index.html"), "w", encoding="utf-8").write(html)
for f in os.listdir(assets): shutil.copy(os.path.join(assets, f), d)
class Q(http.server.SimpleHTTPRequestHandler):
    extensions_map = {**http.server.SimpleHTTPRequestHandler.extensions_map, ".webmanifest": "application/manifest+json", ".woff2": "font/woff2"}
    def log_message(self, *a): pass
srv = http.server.ThreadingHTTPServer(("127.0.0.1", 0), functools.partial(Q, directory=d)); threading.Thread(target=srv.serve_forever, daemon=True).start()
url = f"http://127.0.0.1:{srv.server_address[1]}/index.html"
out = []
with sync_playwright() as p:
    # 1. camera errors
    b = p.chromium.launch()
    for name in ["NotAllowedError", "NotReadableError", "NotFoundError"]:
        pg = b.new_page()
        pg.add_init_script(f"navigator.mediaDevices.getUserMedia = () => Promise.reject(new DOMException('x', '{name}'));")
        pg.goto(url); pg.wait_for_timeout(600)
        pg.click('[data-act="live"]'); pg.wait_for_timeout(600)
        out.append(f"errore {name}: «{pg.evaluate('document.querySelector(\"#scanResult\").textContent').strip()[:110]}»")
        pg.close()
    b.close()
    # 2-4. fake camera: fonts, interruption, back button
    b = p.chromium.launch(args=["--use-fake-ui-for-media-stream", "--use-fake-device-for-media-stream"])
    ctx = b.new_context(permissions=["camera"]); pg = ctx.new_page(); errs = []; reqs = []
    pg.on("pageerror", lambda e: errs.append(str(e))); pg.on("request", lambda r: reqs.append(r.url))
    pg.goto(url); pg.wait_for_timeout(1200)
    out.append("caratteri dal sito: " + str(pg.evaluate("document.fonts.check('600 16px \"Barlow\"') && document.fonts.check('500 16px \"Barlow Semi Condensed\"')")) + " | richieste a servizi esterni: " + str([u for u in reqs if not u.startswith("http://127.0.0.1")] or "nessuna"))
    pg.click('[data-act="live"]'); pg.wait_for_timeout(1500)
    a = pg.evaluate("window.__live.on")
    pg.evaluate("Object.defineProperty(document, 'hidden', { value: true, configurable: true }); document.dispatchEvent(new Event('visibilitychange'))"); pg.wait_for_timeout(400)
    b1 = pg.evaluate("[window.__live.on, !!window.__live.resumeWith]")
    pg.evaluate("Object.defineProperty(document, 'hidden', { value: false, configurable: true }); document.dispatchEvent(new Event('visibilitychange'))"); pg.wait_for_timeout(1500)
    c = pg.evaluate("window.__live.on")
    out.append(f"altra app davanti: fotocamera accesa {a} -> spenta {not b1[0]} (da riaprire {b1[1]}) -> di nuovo accesa {c}")
    pg.go_back(); pg.wait_for_timeout(600)
    out.append(f"tasto Indietro: fotocamera spenta {not pg.evaluate('window.__live.on')}, pagina ancora aperta {pg.url.endswith('/index.html')}")
    out.append("errori JavaScript: " + str(errs or "nessuno"))
    # 5. offline: service worker, then no network
    pg.evaluate("navigator.serviceWorker.register('sw.js').then(() => navigator.serviceWorker.ready)"); pg.wait_for_timeout(3000)
    cache = pg.evaluate("caches.keys().then(async (ks) => { const o = {}; for (const k of ks) o[k] = (await (await caches.open(k)).keys()).length; return o; })")
    ctx.set_offline(True)
    pg.reload(); pg.wait_for_timeout(1500)
    ok = pg.evaluate("!!document.querySelector('[data-act=\"live\"]') && document.fonts.check('600 16px \"Barlow\"')")
    three = pg.evaluate("new Promise((res) => { const s = document.createElement('script'); s.src = 'three.min.js'; s.onload = () => res(!!window.THREE); s.onerror = () => res(false); document.head.append(s); })")
    out.append(f"senza rete: memoria {cache}, pagina e caratteri {ok}, libreria 3D {three}")
    b.close()
srv.shutdown()
print("\n".join(out))
````

## test_relief.js

````javascript
// reliefFromViews on synthetic scans: is the side (sign) right, and are the heights in the right order?
const fs = require("fs");
const reliefFromViews = require("./core.js").reliefFromViews;
for (const [logf, truthf] of process.argv.slice(2).reduce((a, x, i, l) => (i % 2 ? a : a.concat([[x, l[i + 1]]])), [])) {
  const L = JSON.parse(fs.readFileSync(logf, "utf8").replace(/-?Infinity/g, "1e999").replace(/NaN/g, "null"));
  const T = JSON.parse(fs.readFileSync(truthf, "utf8"));
  const truthAt = (t) => { let b = T.truth[0]; for (const x of T.truth) if (x.t <= t) b = x; return b; };
  const rows = L.log.map((e) => ({ e, tr: truthAt(e.t) }));
  const faces = [...new Set(T.truth.filter((x) => x.ang < 3).map((x) => x.P))];
  let right = 0, sureRight = 0, sureAll = 0, tot = 0, ordOk = 0, ordAll = 0;
  const line = [];
  for (const P of faces) {
    const fr = rows.filter((r) => r.tr.P === P && r.tr.ang < 3 && r.e.ok && r.e.edges && r.e.edges.C && Object.keys(r.e.edges).length >= 9);
    if (!fr.length) continue;
    const f0 = fr[fr.length >> 1].e, tz = truthAt(f0.t).z;
    const vw = rows.filter((r) => r.tr.P === P && r.tr.ang >= 6 && r.tr.ang <= 32 && r.e.edges && r.e.edges.C && Object.keys(r.e.edges).length >= 6).map((r) => ({ edges: r.e.edges }));
    const res = reliefFromViews({ edges: f0.edges }, vw);
    if (!res || !tz) { line.push(`${P}: -`); continue; }
    const nm = Object.keys(res.z).filter((k) => tz[k] != null);
    const corr = nm.reduce((s, k) => s + res.z[k] * tz[k], 0);
    const ok = corr > 0;
    tot++; if (ok) right++; if (res.sure) { sureAll++; if (ok) sureRight++; }
    // order of neighbours: which of two adjacent tiles is higher (steps of more than 3 points)
    const G = { TL: [0, 0], T: [0, 1], TR: [0, 2], L: [1, 0], C: [1, 1], R: [1, 2], BL: [2, 0], B: [2, 1], BR: [2, 2] };
    const zt = (k) => (k === "C" ? 0 : tz[k]), ze = (k) => (k === "C" ? 0 : res.z[k]);
    for (const a in G) for (const b in G) {
      const [ra, ca] = G[a], [rb, cb] = G[b];
      if (!(Math.abs(ra - rb) + Math.abs(ca - cb) === 1 && (ra < rb || (ra === rb && ca < cb)))) continue;
      if (zt(a) == null || zt(b) == null || ze(a) == null || ze(b) == null || Math.abs(zt(a) - zt(b)) <= 3) continue;
      ordAll++; if (Math.sign(ze(a) - ze(b)) === Math.sign(zt(a) - zt(b))) ordOk++;
    }
    line.push(`${P}:${ok ? "giusto" : "SBAGLIATO"}${res.sure ? "" : "(incerto)"} ${res.margin} v${res.n}`);
  }
  console.log(`${logf}: verso giusto ${right}/${tot} (sicuro ${sureRight}/${sureAll}); gradini nel verso giusto ${ordOk}/${ordAll} | ${line.join("  ")}`);
}
````

## test_align3.js

````javascript
// Row/column offsets averaged over all the views of a face (frontal and turned), positions
// normalised by the centre tile in every view (as for the relief). Aligned cubes: how small?
const fs = require("fs");
const RW = [["TL", "T", "TR"], ["L", "C", "R"], ["BL", "B", "BR"]], CL = [["TL", "L", "BL"], ["T", "C", "B"], ["TR", "R", "BR"]];
function offsetsOf(e) {
  const C = e.C; if (!C) return null;
  const w = C.u1 - C.u0, h = C.v1 - C.v0;
  const nu = (x) => (x - C.u0) / w, nv = (y) => (y - C.v0) / h;      // in centre tiles, from the centre's corner
  const mid = (a, b, ka, kb, n) => (e[a] && e[b] ? 0.5 * (n(e[a][ka]) + n(e[b][kb])) : null);
  const rowG = RW.map(([a, b, c]) => [mid(a, b, "u1", "u0", nu), mid(b, c, "u1", "u0", nu)]);
  const colG = CL.map(([a, b, c]) => [mid(a, b, "v1", "v0", nv), mid(b, c, "v1", "v0", nv)]);
  const sh = (G) => [0, 1, 2].map((i) => { const d = []; for (let k = 0; k < 2; k++) { const own = G[i][k], oth = [0, 1, 2].filter((j) => j !== i).map((j) => G[j][k]).filter((x) => x != null); if (own != null && oth.length) d.push(own - oth.reduce((s, x) => s + x, 0) / oth.length); } return d.length ? d.reduce((s, x) => s + x, 0) / d.length : null; });
  return { rows: sh(rowG), cols: sh(colG) };
}
for (let a = 2; a < process.argv.length; a += 2) {
  const L = JSON.parse(fs.readFileSync(process.argv[a], "utf8").replace(/-?Infinity/g, "1e999").replace(/NaN/g, "null"));
  const T = process.argv[a + 1] !== "-" ? JSON.parse(fs.readFileSync(process.argv[a + 1], "utf8")) : null;
  const truthAt = (t) => { let b = T.truth[0]; for (const x of T.truth) if (x.t <= t) b = x; return b; };
  // group the readings by face: truth (synthetic) or runs of the analyser's frontal face (real: by time windows of the slots)
  const groups = {};
  for (const e of L.log) {
    if (!e.edges || !e.edges.C || Object.keys(e.edges).length < 8 || !e.g) continue;
    const key = T ? truthAt(e.t).P : Math.floor(e.t / 6000);
    if (T && truthAt(e.t).ang > 30) continue;
    (groups[key] = groups[key] || []).push(e);
  }
  const single = [], avg = [];
  for (const k in groups) {
    const offs = groups[k].map((e) => offsetsOf(e.edges)).filter(Boolean);
    if (offs.length < 6) continue;
    for (const kind of ["rows", "cols"]) for (let i = 0; i < 3; i++) {
      const v = offs.map((o) => o[kind][i]).filter((x) => x != null);
      if (v.length < 6) continue;
      v.forEach((x) => single.push(Math.abs(x)));
      const sorted = v.slice().sort((p, q) => p - q), med = sorted[sorted.length >> 1];
      avg.push(Math.abs(med));
    }
  }
  const q = (arr, p) => { const s = arr.slice().sort((x, y) => x - y); return s[Math.min(s.length - 1, Math.floor(p * s.length))]; };
  console.log(`${process.argv[a].padEnd(16)} singole viste: 90% ${q(single, 0.9).toFixed(3)} max ${q(single, 1).toFixed(3)} | mediana di tutte le viste per riga/colonna: n ${avg.length}, 90% ${q(avg, 0.9).toFixed(3)}, max ${q(avg, 1).toFixed(3)}, sopra 0.07: ${avg.filter((x) => x > 0.07).length}`);
}
````

## decodifica_dati.js

````javascript
// The measures of the scan of 1 October (data image), decoded again: which stickers would be red?
const MV = require("./core.js");
const txt = {
  F: "TL.u35.4 TL.l47.8 T.u42.4 TR.u20.6 TR.r16.3 L.l23.4 R.r48.8 BL.d46.7 BL.l40.1 B.d23.4 BR.d48.0 BR.r33.2",
  R: "TL.u21.5 TL.l28.1 T.u42.6 TR.u16.0 TR.r43.1 L.l51.6 R.r45.6 BL.d39.3 BL.l46.6 B.d30.4 BR.d31.9 BR.r18.5",
  B: "TL.u15.9 TL.l28.1 T.u10.3 TR.u28.1 TR.r40.6 L.l24.0 R.r43.9 BL.d34.1 BL.l12.5 B.d30.2 BR.d47.1 BR.r27.1",
  L: "TL.u38.2 TL.l18.5 T.u28.5 TR.u40.2 TR.r25.6 L.l39.6 R.r29.3 BL.d50.1 BL.l23.0 B.d34.0 BR.d45.9 BR.r52.0",
  U: "TL.u16.1 TL.l44.0 T.u21.9 TR.u42.1 TR.r27.4 L.l14.0 R.r16.4 BL.d21.2 BL.l46.1 B.d43.0 BR.d23.2 BR.r14.2",
  D: "TL.u55.5 TL.l42.4 T.u37.1 TR.u56.0 TR.r35.2 L.l19.5 R.r52.3 BL.d25.7 BL.l36.5 B.d44.1 BR.d24.1 BR.r17.8",
};
const read = (s) => { const ext = {}; for (const w of s.split(/\s+/)) { const m = w.match(/^(\w+)\.([ulrd])([\d.]+)$/); (ext[m[1]] = ext[m[1]] || {})[{ u: "up", l: "left", r: "right", d: "down" }[m[2]]] = +m[3]; } return { ext, sizes: {}, ok: true }; };
const order = ["F", "R", "B", "L", "U", "D"], NAMES = { F: "Davanti", R: "Destra", B: "Dietro", L: "Sinistra", U: "Sopra", D: "Sotto" };
const list = order.map((P) => read(txt[P]));
const CLS = { A: [16.8, 49.6], B: [22.7, 43.7], C: [29.4, 36.9] };
const d = MV.decodeFree(list, CLS, { placed: { R: 1, B: 2, L: 3, U: 4, D: 5 }, turned: { R: 0, B: 0, L: 0, U: 0, D: 0 } });
const phone = "BLLFUUBFDLLFURDFRLDUFFFRRBRUDUBDBRFBULRULDFLBUDLRBBDRD";
console.log("stato ricostruito di nuovo uguale a quello del telefono:", d.facelets === phone, "| peggiore", d.worst.toFixed(1), "(" + NAMES[d.worstFrom] + ")");
const ORD = "URFDLB", POSN = ["in alto a sinistra", "in alto al centro", "in alto a destra", "al centro a sinistra", "al centro", "al centro a destra", "in basso a sinistra", "in basso al centro", "in basso a destra"];
const red = [];
for (let i = 0; i < 54; i++) for (const o of d.meas[i] || []) {
  if (o.w && o.w < 1) continue;
  const off = Math.abs(o.x - d.thick[d.facelets[i]]);
  if (off > 6) red.push(`tassello ${i} (${NAMES[ORD[Math.floor(i / 9)]]}, ${POSN[i % 9]}): misurato su «${NAMES[o.P]}» ${o.x.toFixed(1)}, nel cubo ricostruito ${d.thick[d.facelets[i]].toFixed(1)}, fuori di ${off.toFixed(1)}`);
}
console.log(red.length ? "rossi con la regola nuova:\n  " + red.join("\n  ") : "nessun rosso");
````

## k7truth.py

````python
"""True measures of every face for a known scramble: the twelve widths (% of the side, as the
page measures them, in the net frame of mcube.FACES) and the height of every tile from the centre
tile. usage: python3 k7truth.py "R U F' L2 D B' R2 U'" out.json  (both front/back thicknesses)"""
import sys, json, numpy as np, mcube
seq, out = sys.argv[1], sys.argv[2]
AX = {'x': (0, 'L', 'R'), 'y': (1, 'D', 'U'), 'z': (2, 'B', 'F')}
def measures(frac):
    c = mcube.MirrorCube(frac=frac)
    for mv in seq.split(): c.move(mv)
    S = c.S
    res = {}
    for P in 'URFDLB':
        n, r, u = (np.array(v, float) for v in mcube.FACES[P])
        def axis(v):
            a = int(np.argmax(np.abs(v))); s = np.sign(v[a]); name = 'xyz'[a]
            _, neg, pos = AX[name]
            lo, hi = sorted([s * c.cut[neg], s * c.cut[pos]])
            return a, s, lo, hi
        ar, sr, lowR, highR = axis(r); au, su, lowU, highU = axis(u); an, sn, _, _ = axis(n)
        on = [cb for cb in c.cubies if any(tuple(int(x) for x in st[0]) == tuple(int(x) for x in n) for st in cb['st'])]
        rng = lambda cb, a, s: sorted([s * cb['lo'][a], s * cb['hi'][a]])
        cen = [cb for cb in on if len(cb['st']) == 1][0]
        top0 = rng(cen, an, sn)[1]
        ext, z = {}, {}
        for cb in on:
            mid = (cb['lo'] + cb['hi']) / 2
            pr, pu = sr * mid[ar], su * mid[au]
            col = 0 if pr < lowR else 2 if pr > highR else 1
            row = 0 if pu > highU else 2 if pu < lowU else 1
            nm = [['TL', 'T', 'TR'], ['L', 'C', 'R'], ['BL', 'B', 'BR']][row][col]
            R_, U_ = rng(cb, ar, sr), rng(cb, au, su)
            e = {}
            if row == 0: e['up'] = round((U_[1] - highU) / S * 100, 1)
            if row == 2: e['down'] = round((lowU - U_[0]) / S * 100, 1)
            if col == 0: e['left'] = round((lowR - R_[0]) / S * 100, 1)
            if col == 2: e['right'] = round((R_[1] - highR) / S * 100, 1)
            if e: ext[nm] = e
            z[nm] = round((rng(cb, an, sn)[1] - top0) / S * 100, 1)
        res[P] = {'ext': ext, 'z': z}
    return res
base = {'U': 0.168, 'D': 0.496, 'F': 0.227, 'B': 0.437, 'R': 0.294, 'L': 0.369}
swap = dict(base, F=0.437, B=0.227)
json.dump({'FB_227_437': measures(base), 'FB_437_227': measures(swap)}, open(out, 'w'))
m = measures(base)
for P in 'FRBLUD':
    print(P, ' '.join(f"{k}.{d[0]}{v}" for k in ['TL', 'T', 'TR', 'L', 'R', 'BL', 'B', 'BR'] for d, v in m[P]['ext'].get(k, {}).items()), '| altezze', m[P]['z'])
````

## k7compare.js

````javascript
// A scan of a cube with a known scramble against the truth (k7truth.py): every face read, the
// true face and turn it matches best, the widths that are off, the heights from the turned views.
// usage: node k7compare.js replay.json k7_true.json [variant]
const fs = require("fs");
const MV = require("./core.js");
const L = JSON.parse(fs.readFileSync(process.argv[2], "utf8").replace(/-?Infinity/g, "1e999").replace(/NaN/g, "null"));
const TRUE = JSON.parse(fs.readFileSync(process.argv[3], "utf8"));
const variants = process.argv[4] ? [process.argv[4]] : Object.keys(TRUE);
const NAMES = ["Davanti", "Destra", "Dietro", "Sinistra", "Sopra", "Sotto"], FN = { F: "Davanti", R: "Destra", B: "Dietro", L: "Sinistra", U: "Sopra", D: "Sotto" };
const DIRS = { up: "u", down: "d", left: "l", right: "r" };
// final reading of every slot, and when it was stored
const conf = {}, ext = {}, turn = {};
let prev = [null, null, null, null, null, null];
for (const ev of L.events) { ev.slots.forEach((s, j) => { if (s && (!prev[j] || JSON.stringify(prev[j].ext) !== JSON.stringify(s.ext))) { conf[j] = ev.t; ext[j] = s.ext; turn[j] = s.turn || 0; } }); prev = ev.slots; }
const flat = (e) => { const o = {}; for (const nm in e) for (const d in e[nm]) o[nm + "." + DIRS[d]] = e[nm][d]; return o; };
for (const V of variants) {
  console.log(`\n=== verità con davanti/dietro ${V.replace("FB_", "").replace("_", "/")} ===`);
  let tot = 0, nOff = 0, nAll = 0;
  for (let j = 0; j < 6; j++) {
    if (!ext[j]) { console.log(`${NAMES[j]}: non letta`); continue; }
    // best true face and turn
    let best = null;
    for (const P of "FRBLUD") for (let k = 0; k < 4; k++) {
      const rot = MV.rotateReading({ ext: ext[j], ok: true }, k);
      const a = flat(rot.ext), b = flat(TRUE[V][P].ext);
      const keys = Object.keys(b).filter((x) => a[x] != null);
      if (keys.length < 8) continue;
      const err = keys.reduce((s, x) => s + Math.abs(a[x] - b[x]), 0) / keys.length;
      if (!best || err < best.err) best = { P, k, err, a, b, keys };
    }
    const second = (() => { let s = null; for (const P of "FRBLUD") { if (P === best.P) continue; for (let k = 0; k < 4; k++) { const a = flat(MV.rotateReading({ ext: ext[j], ok: true }, k).ext), b = flat(TRUE[V][P].ext); const keys = Object.keys(b).filter((x) => a[x] != null); if (keys.length < 8) continue; const err = keys.reduce((q, x) => q + Math.abs(a[x] - b[x]), 0) / keys.length; if (!s || err < s) s = err; } } return s; })();
    const off = best.keys.filter((x) => Math.abs(best.a[x] - best.b[x]) > 4).map((x) => `${x} ${best.a[x].toFixed(1)}≠${best.b[x]}`);
    nOff += off.length; nAll += best.keys.length; tot += best.err;
    console.log(`${NAMES[j]} (letta a ${(conf[j] / 1000).toFixed(1)} s): è «${FN[best.P]}» vera girata ${best.k}, errore medio ${best.err.toFixed(1)} (seconda scelta ${second.toFixed(1)}) | fuori di più di 4: ${off.join(", ") || "nessuna"}`);
  }
  console.log(`misure fuori di più di 4 punti: ${nOff} su ${nAll}`);
}
````

## k7persp.js

````javascript
// Widths of the known-scramble scan against the truth: error vs (true width x height of the tile),
// the perspective of a piece nearer or farther than the centre tile. Then the relief of every face
// (heights from the turned views) and the cube decoded from the widths, against the true state.
const fs = require("fs");
const MV = require("./core.js");
const Cube = require("./cubejs_node.js");
const reliefFromViews = MV.reliefFromViews;
const L = JSON.parse(fs.readFileSync(process.argv[2], "utf8").replace(/-?Infinity/g, "1e999").replace(/NaN/g, "null"));
const TR = JSON.parse(fs.readFileSync(process.argv[3], "utf8")).FB_227_437;
const NAMES = ["Davanti", "Destra", "Dietro", "Sinistra", "Sopra", "Sotto"], FACE = "FRBLUD";
const conf = {}, ext = {}, firstConf = {};
let prev = [null, null, null, null, null, null];
for (const ev of L.events) { ev.slots.forEach((s, j) => { if (s && (!prev[j] || JSON.stringify(prev[j].ext) !== JSON.stringify(s.ext))) { if (firstConf[j] == null) firstConf[j] = ev.t; conf[j] = ev.t; ext[j] = s.ext; } }); prev = ev.slots; }
// 1. widths: error against w * z
const pts = [];
for (let j = 0; j < 6; j++) {
  const P = FACE[j], te = TR[P].ext, tz = TR[P].z;
  for (const nm in te) for (const d in te[nm]) {
    const m = ext[j] && ext[j][nm] && ext[j][nm][d];
    if (typeof m !== "number") continue;
    pts.push({ f: NAMES[j], nm, d, w: te[nm][d], z: tz[nm], e: m - te[nm][d] });
  }
}
const sxx = pts.reduce((s, p) => s + (p.w * p.z) ** 2, 0), sxy = pts.reduce((s, p) => s + p.w * p.z * p.e, 0), k = sxy / sxx;
const rms = (arr) => Math.sqrt(arr.reduce((s, x) => s + x * x, 0) / arr.length);
const res0 = pts.map((p) => p.e), res1 = pts.map((p) => p.e - k * p.w * p.z);
console.log(`misure ${pts.length}: errore quadratico medio ${rms(res0).toFixed(2)} punti; fuori di più di 4: ${res0.filter((x) => Math.abs(x) > 4).length}`);
console.log(`errore ≈ larghezza × altezza / D con D = ${(1 / k).toFixed(0)} (in % del lato, cioè circa ${(57 / (100 * k)).toFixed(0)} mm dal telefono al piano del tassello centrale)`);
console.log(`dopo la correzione: errore quadratico medio ${rms(res1).toFixed(2)} punti; fuori di più di 4: ${res1.filter((x) => Math.abs(x) > 4).length}`);
for (const p of pts.filter((p, i) => Math.abs(res0[i]) > 4)) console.log(`   ${p.f} ${p.nm}.${p.d}: vera ${p.w}, altezza ${p.z}, misurata ${(p.w + p.e).toFixed(1)} → corretta ${(p.w + p.e - k * p.w * p.z).toFixed(1)}`);
// 2. relief of every face: views between its first reading and the next face
const order = Object.keys(firstConf).map(Number).sort((a, b) => firstConf[a] - firstConf[b]);
for (const j of order) {
  const t0 = firstConf[j], t1 = Math.min(...order.map((q) => firstConf[q]).filter((t) => t > t0), Infinity);
  const near = L.log.filter((e) => e.t >= t0 - 1500 && e.t < Math.min(t1, t0 + 30000) && e.edges && e.edges.C && e.g);
  const fr = near.filter((e) => e.ok && Object.keys(e.edges).length >= 9 && Math.abs((e.g[1] - e.g[0]) / (e.h[1] - e.h[0]) - 1) < 0.03);
  const vw = near.filter((e) => { const a = Math.abs((e.g[1] - e.g[0]) / (e.h[1] - e.h[0]) - 1); return a > 0.05 && a < 0.35 && Object.keys(e.edges).length >= 6; }).map((e) => ({ edges: e.edges }));
  if (!fr.length) { console.log(`${NAMES[j]}: nessuna vista di fronte`); continue; }
  const rel = reliefFromViews({ edges: fr[fr.length >> 1].edges }, vw);
  const tz = TR[FACE[j]].z;
  if (!rel) { console.log(`${NAMES[j]}: viste inclinate ${vw.length}, rilievo non calcolabile`); continue; }
  const nm = Object.keys(rel.z).filter((x) => tz[x] != null);
  const corr = nm.reduce((s, x) => s + rel.z[x] * tz[x], 0) / Math.sqrt(nm.reduce((s, x) => s + rel.z[x] ** 2, 0) * nm.reduce((s, x) => s + tz[x] ** 2, 0));
  console.log(`${NAMES[j]}: viste inclinate ${vw.length} (in ${((Math.min(t1, t0 + 30000) - t0) / 1000).toFixed(0)} s), verso ${rel.margin >= 0.2 ? "sicuro" : "incerto"} (${rel.margin}), somiglianza con il rilievo vero ${corr.toFixed(2)} ${corr > 0.8 ? "(buona)" : corr > 0.5 ? "(discreta)" : corr > 0 ? "(scarsa)" : "(verso sbagliato)"}`);
}
// 3. the cube from the widths, against the true state
const c = new Cube(); c.move("R U F' L2 D B' R2 U'");
const truth = c.asString();
const list = [0, 1, 2, 3, 4, 5].map((j) => ({ ext: ext[j], ok: true, sizes: {} }));
const CLS = { A: [16.8, 49.6], B: [22.7, 43.7], C: [29.4, 36.9] };
const d = MV.decodeFree(list, CLS, { placed: { R: 1, B: 2, L: 3, U: 4, D: 5 }, turned: { R: 0, B: 0, L: 0, U: 0, D: 0 } });
let wrong = 0; for (let i = 0; i < 54; i++) if (d.facelets[i] !== truth[i]) wrong++;
console.log(`cubo ricostruito dalle larghezze: tasselli sbagliati ${wrong} su 54 (peggiore ${d.worst.toFixed(1)})`);
````

## k7check.js

````javascript
// The first check of the page (slots as they were at a given time), widths only, perspective corrected.
const fs = require("fs");
const MV = require("./core.js");
const Cube = require("./cubejs_node.js");
const L = JSON.parse(fs.readFileSync(process.argv[2], "utf8").replace(/-?Infinity/g, "1e999").replace(/NaN/g, "null"));
const tCheck = +process.argv[3];
const ext = {}, conf = {};
let prev = [null, null, null, null, null, null];
for (const ev of L.events) { if (ev.t > tCheck) break; ev.slots.forEach((s, j) => { if (s && (!prev[j] || JSON.stringify(prev[j].ext) !== JSON.stringify(s.ext))) { ext[j] = s.ext; conf[j] = ev.t; } }); prev = ev.slots; }
const list = [0, 1, 2, 3, 4, 5].map((j) => { const fr = L.log.filter((e) => e.ok && e.sizes && Math.abs(e.t - conf[j]) < 1200).sort((a, b) => Math.abs(a.t - conf[j]) - Math.abs(b.t - conf[j]))[0]; return { ext: ext[j], ok: true, sizes: fr ? fr.sizes : {} }; });
const CLS = { A: [16.8, 49.6], B: [22.7, 43.7], C: [29.4, 36.9] };
const d = MV.decodeFree(list, CLS, { placed: { R: 1, B: 2, L: 3, U: 4, D: 5 }, turned: { R: 0, B: 0, L: 0, U: 0, D: 0 } });
const c = new Cube(); c.move("R U F' L2 D B' R2 U'"); const truth = c.asString();
let w = 0; for (let i = 0; i < 54; i++) if (d.facelets[i] !== truth[i]) w++;
const PIECES = [[8, 9, 20], [6, 18, 38], [0, 36, 47], [2, 45, 11], [29, 26, 15], [27, 44, 24], [33, 53, 42], [35, 17, 51], [5, 10], [7, 19], [3, 37], [1, 46], [32, 16], [28, 25], [30, 43], [34, 52], [23, 12], [21, 41], [50, 39], [48, 14]];
const onFace = (i, P) => { const pc = PIECES.find((x) => x.includes(i)); const f = pc && pc.find((x) => "URFDLB"[Math.floor(x / 9)] === P); return f != null ? f : i; };
const obs = [];
for (let i = 0; i < 54; i++) for (const o of d.meas[i] || []) { if ((o.w && o.w < 1) || o.key) continue; obs.push({ i, o, z: d.thick[d.facelets[onFace(i, o.P)]] - d.thick[o.P], t: d.thick[d.facelets[i]] }); }
let best = null;
for (const D of [Infinity, 600, 400, 280, 200, 150]) {
  const val = (b) => (isFinite(D) ? b.o.x / (1 + b.z / D) : b.o.x) - b.t;
  const cst = obs.reduce((s, b) => s + Math.min(64, val(b) ** 2), 0), out = obs.filter((b) => Math.abs(val(b)) > 6);
  if (!best || cst < best.cst) best = { D, cst, out };
}
console.log(`letture a ${tCheck / 1000} s: stato ${w} tasselli sbagliati; larghezze ${obs.length}; distanza scelta ${best.D}; rossi dalle larghezze: ${best.out.length} ${best.out.map((b) => b.o.P + ":" + ((isFinite(best.D) ? b.o.x / (1 + b.z / best.D) : b.o.x) - b.t).toFixed(1)).join(" ")}`);
````

## k7walls.js

````javascript
// Step 2 offline: the black band between two neighbouring blocks, in the turned views of the known-
// scramble scan, against the band expected from a cube (heights from its stickers, turn of the view
// fitted on the tiles). True cube: no alarms wanted; wrong cubes (pieces swapped on purpose): seen?
const fs = require("fs");
const Cube = require("./cubejs_node.js");
const L = JSON.parse(fs.readFileSync(process.argv[2], "utf8").replace(/-?Infinity/g, "1e999").replace(/NaN/g, "null"));
const FACE = "FRBLUD", NAMES = ["Davanti", "Destra", "Dietro", "Sinistra", "Sopra", "Sotto"];
const TH = { U: 16.8, D: 49.6, F: 22.7, B: 43.7, R: 29.4, L: 36.9 };
const POS = { TL: 0, T: 1, TR: 2, L: 3, C: 4, R: 5, BL: 6, B: 7, BR: 8 }, G = { TL: [0, 0], T: [0, 1], TR: [0, 2], L: [1, 0], C: [1, 1], R: [1, 2], BL: [2, 0], B: [2, 1], BR: [2, 2] };
const AT = {}; for (const k in G) AT[G[k].join(",")] = k;
const heights = (fac, P) => { const fi = "URFDLB".indexOf(P), z = {}; for (const nm in POS) z[nm] = TH[fac[fi * 9 + POS[nm]]] - TH[P]; return z; };
// windows of every face (first reading to the next face)
const firstConf = {};
let prev = [null, null, null, null, null, null];
for (const ev of L.events) { ev.slots.forEach((s, j) => { if (s && !prev[j] && firstConf[j] == null) firstConf[j] = ev.t; }); prev = ev.slots; }
const order = Object.keys(firstConf).map(Number).sort((a, b) => firstConf[a] - firstConf[b]);
const faceViews = {};
for (const j of order) {
  const t0 = firstConf[j], t1 = Math.min(...order.map((q) => firstConf[q]).filter((t) => t > t0), Infinity);
  const near = L.log.filter((e) => e.t >= t0 - 1500 && e.t < Math.min(t1, t0 + 30000) && e.edges && e.edges.C && e.g && Object.keys(e.edges).length >= 9);
  const fr = near.filter((e) => e.ok && Math.abs((e.g[1] - e.g[0]) / (e.h[1] - e.h[0]) - 1) < 0.03);
  const vw = near.filter((e) => { const a = Math.abs((e.g[1] - e.g[0]) / (e.h[1] - e.h[0]) - 1); return a > 0.05 && a < 0.35; });
  if (fr.length && vw.length >= 6) faceViews[j] = { fr: fr[fr.length >> 1], vw };
}
const GOLD = 28.9;   // width of the gold of the centre tile, % of the side
// for one face, one cube hypothesis: residual of every band in every view
function bands(j, z) {
  const { fr, vw } = faceViews[j];
  const C0 = fr.edges.C, s0u = GOLD / (C0.u1 - C0.u0), s0v = GOLD / (C0.v1 - C0.v0);
  const out = [];
  for (const v of vw) {
    const e = v.edges, C = e.C, su = GOLD / (C.u1 - C.u0), sv = GOLD / (C.v1 - C.v0);
    const nu = (x, c, s) => (x - c) * s;
    // turn of the view from the tiles, with these heights (r = z * a)
    let nU = 0, dU = 0, nV = 0, dV = 0;
    for (const nm in G) {
      if (nm === "C" || !e[nm] || !fr.edges[nm]) continue;
      const [r, c] = G[nm], zz = z[nm];
      const ru = (k) => nu(e[nm][k], C.u0, su) - nu(fr.edges[nm][k], C0.u0, s0u), rv = (k) => nu(e[nm][k], C.v0, sv) - nu(fr.edges[nm][k], C0.v0, s0v);
      if (c > 0) { nU += zz * ru("u0"); dU += zz * zz; } if (c < 2) { nU += zz * ru("u1"); dU += zz * zz; }
      if (r > 0) { nV += zz * rv("v0"); dV += zz * zz; } if (r < 2) { nV += zz * rv("v1"); dV += zz * zz; }
    }
    const au = dU ? nU / dU : 0, av = dV ? nV / dV : 0;
    // bands between neighbours
    for (const nm in G) {
      const [r, c] = G[nm];
      for (const [dr, dc, ax] of [[0, 1, "u"], [1, 0, "v"]]) {
        const nb = AT[(r + dr) + "," + (c + dc)];
        if (!nb || !e[nm] || !e[nb] || !fr.edges[nm] || !fr.edges[nb]) continue;
        const gap = (E, s) => (ax === "u" ? (E[nb].u0 - E[nm].u1) * s : (E[nb].v0 - E[nm].v1) * s);
        const g0 = gap(fr.edges, ax === "u" ? s0u : s0v), g1 = gap(e, ax === "u" ? su : sv);
        let pred = (z[nb] - z[nm]) * (ax === "u" ? au : av);
        pred = Math.max(pred, -g0);                       // a higher block covers the band, it cannot go below nothing
        out.push({ pair: nm + "-" + nb, dz: z[nb] - z[nm], obs: g1 - g0, pred, res: g1 - g0 - pred });
      }
    }
  }
  return out;
}
const score = (arr) => { const big = arr.filter((x) => Math.abs(x.pred) > 1.5 || Math.abs(x.obs) > 1.5); const r = big.map((x) => Math.abs(x.res)).sort((a, b) => a - b); return { n: big.length, med: r.length ? r[r.length >> 1] : 0, out: big.filter((x) => Math.abs(x.res) > 2.5).length / Math.max(1, big.length) }; };
const c = new Cube(); c.move("R U F' L2 D B' R2 U'"); const truth = c.asString();
// wrong cubes: two corners swapped, two edges swapped, one corner twisted
const swapF = (f, A, B) => { const a = f.split(""); for (let k = 0; k < A.length; k++) { const t = a[A[k]]; a[A[k]] = a[B[k]]; a[B[k]] = t; } return a.join(""); };
const twist = (f, A) => { const a = f.split(""); const t = a[A[0]]; a[A[0]] = a[A[1]]; a[A[1]] = a[A[2]]; a[A[2]] = t; return a.join(""); };
const HYP = { "cubo vero": truth, "due angoli scambiati (UFL/URF)": swapF(truth, [6, 18, 38], [8, 20, 9]), "due spigoli scambiati (UF/FR)": swapF(truth, [7, 19], [23, 12]), "un angolo girato (DFR)": twist(truth, [29, 26, 15]) };
console.log("faccia      | " + Object.keys(HYP).join(" | "));
for (const j of Object.keys(faceViews).map(Number)) {
  const P = FACE[j];
  const cells = Object.values(HYP).map((fac) => { const s = score(bands(j, heights(fac, P))); return `${s.med.toFixed(2)} (${Math.round(s.out * 100)}% fuori, ${s.n})`; });
  console.log(`${NAMES[j].padEnd(11)} | ${cells.join(" | ")}`);
}
// robust: one value per pair of neighbours (median over the views where the band should change)
console.log("\nmediana per coppia di blocchi (coppie fuori di più di 2,5 punti / coppie controllate):");
console.log("faccia      | " + Object.keys(HYP).join(" | "));
for (const j of Object.keys(faceViews).map(Number)) {
  const P = FACE[j];
  const cells = Object.values(HYP).map((fac) => {
    const b = bands(j, heights(fac, P)).filter((x) => Math.abs(x.pred) > 1.5 || Math.abs(x.obs) > 1.5);
    const by = {}; for (const x of b) (by[x.pair] = by[x.pair] || []).push(x.res);
    const pairs = Object.values(by).filter((v) => v.length >= 5).map((v) => { const s = v.slice().sort((p, q) => p - q); return s[s.length >> 1]; });
    return `${pairs.filter((m) => Math.abs(m) > 2.5).length}/${pairs.length}`;
  });
  console.log(`${NAMES[j].padEnd(11)} | ${cells.join(" | ")}`);
}
````
