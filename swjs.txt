self.addEventListener('install', e => {
  e.waitUntil(
    caches.open('cogs-v1').then(cache => cache.addAll(['/', 'style.css', 'script.js']))
  );
});
self.addEventListener('fetch', e => {
  e.respondWith(
    caches.match(e.request).then(res => res || fetch(e.request))
  );
});