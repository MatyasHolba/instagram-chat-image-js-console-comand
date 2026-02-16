# instagram-chat-image-js-console-comand
It will show you download buttons above the pusctures in chat plus it has also button for download all iif is are images in one message 


____________________________________________ 
<script>------CODE for copy ---------->

(function(){
  if (window.igdl_msg_v3) { console.info('IGDL v3 už běží. Použij igdl.cleanup() nebo igdl.stop()'); return; }
  window.igdl_msg_v3 = true;

  const LEFT_COL_RATIO = 0.33;
  const GROUP_MIN = 2;
  const DOWNLOAD_DELAY = 220;

  function cleanupButtons(){
    document.querySelectorAll('.igdl-btn, .igdl-all-btn').forEach(n=>n.remove());
    document.querySelectorAll('[data-igdl-added]').forEach(n=>n.removeAttribute('data-igdl-added'));
    document.querySelectorAll('[data-igdl-all-added]').forEach(n=>n.removeAttribute('data-igdl-all-added'));
  }

  function filenameFromUrl(url, idx=0){
    try{
      const u = new URL(url);
      const path = (u.pathname.split('/').filter(Boolean).pop() || 'image').split('?')[0];
      const ext = path.includes('.') ? path.split('.').pop() : 'jpg';
      const base = path.includes('.') ? path.split('.').slice(0,-1).join('.') : path;
      return `${base || 'instagram'}_${Date.now()}_${idx}.${ext}`;
    }catch(e){ return `instagram_${Date.now()}_${idx}.jpg`; }
  }

  function downloadImage(url, filename){
    if(!url) return;
    fetch(url, {mode:'cors'}).then(r=>r.blob()).then(blob=>{
      const a=document.createElement('a');
      a.href=URL.createObjectURL(blob);
      a.download=filename;
      document.body.appendChild(a);
      a.click();
      a.remove();
      setTimeout(()=>URL.revokeObjectURL(a.href),5000);
    }).catch(err=>{
      // fallback: otevřít obrázek v novém okně
      console.warn('IGDL fetch failed, opening in new tab', err);
      window.open(url, '_blank');
    });
  }

  function makeBtn(text, opts={}){
    const b = document.createElement('button');
    b.className = opts.all ? 'igdl-all-btn' : 'igdl-btn';
    b.textContent = text;
    Object.assign(b.style, {
      position:'absolute',
      zIndex:999999,
      padding:'6px 10px',
      fontSize:'12px',
      borderRadius:'8px',
      border:'none',
      cursor:'pointer',
      pointerEvents:'auto',
      boxShadow:'0 2px 8px rgba(0,0,0,0.45)',
      ...opts.style
    });
    if(opts.bg) b.style.background = opts.bg;
    if(opts.color) b.style.color = opts.color;
    // zabránit otevření IG modal po kliknutí
    b.addEventListener('click', e=>{ e.stopPropagation(); e.preventDefault(); });
    return b;
  }

  // Exclude: profilovky / avatar links (odkazy na profil mají href="/" nebo aria-label "Open the profile")
  function isProfileImage(img){
    try{
      if(!img) return false;
      const alt = (img.alt||'').toLowerCase();
      if(alt.includes('profile') || alt.includes('user-profile')) return true;
      // pokud je uvnitř linku na profil
      const a = img.closest('a');
      if(a && a.getAttribute('href') && a.getAttribute('href').startsWith('/')) return true;
      // specifická třída pro avatar v tvém dumpu: x15mokao / x1ga7v0g -> kontrola třídy
      const cls = img.className || '';
      if(cls.includes('x15mokao') || cls.includes('x1ga7v0g')) return true;
      return false;
    }catch(e){ return false; }
  }

  function isVisible(el){
    try{
      const r = el.getBoundingClientRect();
      return r.width>8 && r.height>8 && r.bottom>0 && r.top < window.innerHeight;
    }catch(e){ return false; }
  }

  function isOnRight(el){
    try{
      const r = el.getBoundingClientRect();
      return r.left > window.innerWidth * LEFT_COL_RATIO;
    }catch(e){ return true; }
  }

  function findWrapper(img){
    if(!img) return null;
    let w = img.closest('[role="button"]') || img.closest('[style*="--x-paddingTop"]') || img.parentElement;
    return w;
  }

  function addButtons(){
    cleanupButtons();

    // všechny kandidátní obrázky v chatu podle tvého dumpu
    const allImgs = Array.from(document.querySelectorAll('img.x1rg5ohu, img[src*="scontent"], img[alt*="Otevřít fotku"]'));

    // filtrujeme: vynech profilovky a neviditelné / levý panel
    const imgs = allImgs.filter(img=>{
      if(!isVisible(img)) return false;
      if(isProfileImage(img)) return false;
      if(!isOnRight(img)) return false;
      return true;
    });

    // přidej tlačítko ke každému obrázku
    imgs.forEach((img, idx)=>{
      if(img.dataset.igdlAdded) return;
      const wrap = findWrapper(img);
      if(!wrap) return;
      // only on right side wrappers
      if(!isOnRight(wrap)) return;

      const prevPos = window.getComputedStyle(wrap).position;
      if(prevPos === 'static' || !prevPos) wrap.style.position = 'relative';

      const btn = makeBtn('Download', { style: { bottom: '8px', right: '8px', background:'rgba(0,0,0,0.75)', color:'#fff' }});
      btn.addEventListener('click', e=>{
        e.stopPropagation(); e.preventDefault();
        const url = img.src || img.getAttribute('src') || img.dataset.src || img.dataset.lazy;
        downloadImage(url, filenameFromUrl(url));
      });

      wrap.appendChild(btn);
      img.dataset.igdlAdded = '1';
    });

    // Seskupení: najdeme pro každý obrázek "message ancestor", kde se nachází >= GROUP_MIN cílových obrázků.
    // Heuristika: pro každé img posuneme se nahoru až 6 levelů a hledáme ancestor, který obsahuje >=2 cílových img.
    const doneContainers = new Set();
    imgs.forEach(img=>{
      if(!img) return;
      let candidate = img;
      for(let level=0; level<6 && candidate; level++){
        candidate = candidate.parentElement;
        if(!candidate) break;
        // spočítáme cílové obrázky uvnitř tohoto ancestoru
        const contained = Array.from(candidate.querySelectorAll('img.x1rg5ohu, img[src*="scontent"], img[alt*="Otevřít fotku"]'))
                            .filter(i=> isVisible(i) && !isProfileImage(i) && isOnRight(i));
        if(contained.length >= GROUP_MIN){
          // přidej "Download All" pokud ještě nebylo přidáno a pokud ještě nebyl použit tento container
          if(!candidate.dataset.igdlAllAdded && !doneContainers.has(candidate)){
            // zamezíme dávání tlačítka na velké kontejnery (levý panel) pomocí pozice
            if(!isOnRight(candidate)) break;
            const prevPos = window.getComputedStyle(candidate).position;
            if(prevPos === 'static' || !prevPos) candidate.style.position = 'relative';

            const btnAll = makeBtn('Download All', { all: true, style:{ top: '8px', right: '8px' }, bg:'#ff2d7f', color:'#fff' });
            btnAll.addEventListener('click', ev=>{
              ev.stopPropagation(); ev.preventDefault();
              contained.forEach((ci, i)=>{
                const url = ci.src || ci.getAttribute('src') || ci.dataset.src || ci.dataset.lazy;
                setTimeout(()=> downloadImage(url, filenameFromUrl(url, i)), i * DOWNLOAD_DELAY);
              });
            });

            candidate.appendChild(btnAll);
            candidate.dataset.igdlAllAdded = '1';
            doneContainers.add(candidate);
          }
          break; // našli jsme vhodný ancestor pro tento obrázek -> pokračovat dalším obrázkem
        }
      }
    });
  }

  // inicialní průchod
  addButtons();

  // observer
  const mo = new MutationObserver(muts=>{
    // pokud jsou přidané/změněné uzly, debounce volání
    if(window.__igdl_debounce) clearTimeout(window.__igdl_debounce);
    window.__igdl_debounce = setTimeout(()=>{ addButtons(); }, 180);
  });
  mo.observe(document.body, { childList:true, subtree:true });

  // resize handler (layout může změnit levý panel / pravou část)
  const onResize = ()=> { if(window.__igdl_resize) clearTimeout(window.__igdl_resize); window.__igdl_resize = setTimeout(()=>addButtons(), 200); };
  window.addEventListener('resize', onResize);

  // export ovládání
  window.igdl = {
    addButtons,
    cleanup: cleanupButtons,
    stop: ()=>{ mo.disconnect(); window.removeEventListener('resize', onResize); cleanupButtons(); delete window.igdl_msg_v3; delete window.igdl; console.info('IGDL v3 zastaveno'); }
  };

  console.info('IGDL v3 aktivní — igdl.cleanup() smaže tlačítka, igdl.stop() vypne observer.');
})();
