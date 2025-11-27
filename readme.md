# MIJSR — Mini In-Browser JavaScript Runner

Mini In-Browser JavaScript Runner (MIJSR)

### Note: this is an archived version of MIJSRv1

---

This repo provides a tiny popup UI (`ui.html`) that you can open from a **console command** or a **bookmarklet**. Use the UI to paste JavaScript and run it in the page you opened the launcher from, or type the name of an app (see `apps/`).

---

## Open the UI

### Option A — Console (paste & run)

Paste the block below into DevTools → **Console** and press Enter:

```js
// MIJSR launcher (console) — installs direct exec on the page + opens popup
(async ()=>{
  const JS_PATH = 'https://cdn.jsdelivr.net/gh/yxorp69/MIJSR@main/ui.html';
  const SECRET = Math.random().toString(36).slice(2);

  // install only once
  if(!window.__mijsr_installed){
    window.__mijsr_installed = true;

    // Direct executor callable by popup: runs code in page context.
    // We intentionally use Function to get top-level scope (not arrow).
    window.__mijsr_exec = function(code){
      try{
        // optional: limit visible errors and return result
        return (new Function(code))();
      }catch(e){
        // surface error to page console
        console.error('[MIJSR] exec error:', e);
        throw e;
      }
    };

    // postMessage handler (keeps compatibility)
    window.addEventListener('message', function mijsr_msg(e){
      try{
        const d = e.data;
        if(!d || d.type !== 'mini-run' || d.secret !== SECRET) return;
        try{
          // use the same executor to run
          window.__mijsr_exec(d.code);
        }catch(err){
          console.error('[MIJSR] mini-run eval error:', err);
          try{ alert('mini-run eval error: '+err); }catch(e){}
        }
      }catch(ex){ console.error(ex); }
    }, false);

    console.log('[MIJSR] installer: __mijsr_exec added and message listener installed.');
  } else {
    console.log('[MIJSR] already installed.');
  }

  // open popup
  const w = window.open('about:blank', 'mini-run-popup', 'width=900,height=560');
  if(!w){ alert('Popup blocked — allow popups for this site'); return; }

  try{
    const res = await fetch(JS_PATH, {cache:'no-store'});
    if(!res.ok) throw new Error(res.status + ' ' + res.statusText);
    let txt = await res.text();
    txt = txt.replace(/___MINI_RUN_SECRET___/g, SECRET);
    // ensure ui focuses app input by name — ui handles that
    txt = txt.replace(/__DEFAULT_REPO__/g, 'yxorp69/MIJSR@main');
    w.document.open(); w.document.write(txt); w.document.close(); w.focus();
  }catch(err){
    console.error('[MIJSR] Failed to load UI:', err);
    alert('Failed to load UI: ' + err);
  }
})();
```

### Option B — Bookmarklet

Create a bookmark and paste this entire string into the bookmark’s **URL** field. Click the bookmark to open the UI.

```
javascript:(function(){const JS_URL='https://cdn.jsdelivr.net/gh/yxorp69/MIJSR@main/ui.html';const SECRET=Math.random().toString(36).slice(2);if(!window.__mijsr_installed){window.__mijsr_installed=true;window.__mijsr_exec=function(code){try{return (new Function(code))();}catch(e){console.error('[MIJSR] exec error:',e);throw e;}};window.addEventListener('message',function(e){try{const d=e.data;if(!d||d.type!=='mini-run'||d.secret!==SECRET)return;window.__mijsr_exec(d.code);}catch(err){console.error(err);}},false);console.log('[MIJSR] installer: __mijsr_exec added.');}else console.log('[MIJSR] already installed.');var w=window.open('about:blank','mini-run-popup','width=900,height=560');if(!w){alert('Popup blocked — allow popups');return;}fetch(JS_URL,{cache:'no-store'}).then(r=>{if(!r.ok)throw new Error(r.status+' '+r.statusText);return r.text()}).then(t=>{t=t.replace(/___MINI_RUN_SECRET___/g,SECRET);t=t.replace(/__DEFAULT_REPO__/g,'yxorp69/MIJSR@main');w.document.open();w.document.write(t);w.document.close();w.focus();}).catch(e=>{console.error('load UI failed',e);alert('load UI failed: '+e);});})();

```

---

## Run code through the UI

1. Open the popup using **Console** or **Bookmarklet** (above).
2. In the popup UI:

   * Paste JavaScript into the textarea and click **Run** (or press **Ctrl+Enter**) — this executes the code in the original page.
   * Or type an **app name** (e.g. `example`) into the **Path** field and click **Fetch**. The UI will automatically fetch `apps/<name>.js` from this repo (so `example` → `apps/example.js`). Then click **Run**.
3. The popup sends the code to the opener page, which verifies the popup’s secret token and executes the code.
