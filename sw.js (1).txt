
const APP_SHELL_CACHE = 'quran-app-shell-v2';
const AUDIO_CACHE = 'quran-audio-cache';
const APP_SHELL_URLS = [
  './',
  './index.html',
  './manifest.json'
];

self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(APP_SHELL_CACHE).then(cache => cache.addAll(APP_SHELL_URLS)).then(() => self.skipWaiting())
  );
});

self.addEventListener('activate', event => {
  event.waitUntil(
    caches.keys().then(keys => Promise.all(
      keys.filter(k => k !== APP_SHELL_CACHE && k !== AUDIO_CACHE).map(k => caches.delete(k))
    )).then(() => self.clients.claim())
  );
});

self.addEventListener('fetch', event => {
  const req = event.request;
  const url = new URL(req.url);

  // صوتيات القرآن - Cache First مع تخزين
  if (url.hostname.includes('mp3quran.net')) {
    event.respondWith(
      caches.open(AUDIO_CACHE).then(cache => 
        cache.match(req).then(cached => {
          if (cached) return cached;
          return fetch(req).then(res => {
            if (res.ok) cache.put(req, res.clone());
            return res;
          }).catch(() => cached);
        })
      )
    );
    return;
  }

  // ملفات التطبيق - Stale While Revalidate
  if (req.method === 'GET') {
    event.respondWith(
      caches.match(req).then(cached => {
        const fetched = fetch(req).then(networkRes => {
          if (networkRes.ok && url.origin === self.location.origin) {
            caches.open(APP_SHELL_CACHE).then(cache => cache.put(req, networkRes.clone()));
          }
          return networkRes;
        }).catch(() => cached);
        return cached || fetched;
      })
    );
  }
});

// للتحميل المسبق من التطبيق
self.addEventListener('message', event => {
  if (event.data && event.data.type === 'CACHE_URLS') {
    const urls = event.data.urls;
    event.waitUntil(
      caches.open(AUDIO_CACHE).then(cache => {
        return Promise.all(urls.map(u => fetch(u).then(r => r.ok && cache.put(u, r))));
      })
    );
  }
});
