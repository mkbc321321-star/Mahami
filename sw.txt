var VERSION = "mahami-v1";
var SHELL = [
  "./",
  "index.html",
  "manifest.webmanifest",
  "icon-192.png",
  "icon-512.png",
  "badge-96.png"
];

self.addEventListener("install", function (e) {
  e.waitUntil(caches.open(VERSION).then(function (c) { return c.addAll(SHELL); }).then(function () { return self.skipWaiting(); }));
});

self.addEventListener("activate", function (e) {
  e.waitUntil(
    caches.keys().then(function (keys) {
      return Promise.all(keys.filter(function (k) { return k !== VERSION; }).map(function (k) { return caches.delete(k); }));
    }).then(function () { return self.clients.claim(); })
  );
});

self.addEventListener("fetch", function (e) {
  var req = e.request;
  if (req.method !== "GET") return;
  var url = new URL(req.url);

  if (url.origin === "https://fonts.googleapis.com" || url.origin === "https://fonts.gstatic.com") {
    e.respondWith(
      caches.open(VERSION + "-fonts").then(function (c) {
        return c.match(req).then(function (hit) {
          var net = fetch(req).then(function (res) { if (res && (res.ok || res.type === "opaque")) c.put(req, res.clone()); return res; }).catch(function () { return hit; });
          return hit || net;
        });
      })
    );
    return;
  }

  if (url.origin !== self.location.origin) return;

  if (req.mode === "navigate") {
    e.respondWith(
      fetch(req).then(function (res) {
        var copy = res.clone();
        caches.open(VERSION).then(function (c) { c.put("index.html", copy); });
        return res;
      }).catch(function () { return caches.match("index.html"); })
    );
    return;
  }

  e.respondWith(
    caches.match(req, { ignoreSearch: true }).then(function (hit) {
      var net = fetch(req).then(function (res) {
        if (res && res.ok) { var copy = res.clone(); caches.open(VERSION).then(function (c) { c.put(req, copy); }); }
        return res;
      }).catch(function () { return hit; });
      return hit || net;
    })
  );
});

self.addEventListener("notificationclick", function (e) {
  var n = e.notification;
  var id = n.data && n.data.id;
  var action = e.action || "open";
  n.close();
  e.waitUntil(
    self.clients.matchAll({ type: "window", includeUncontrolled: true }).then(function (list) {
      var client = list.find(function (c) { return "focus" in c; });
      if (client) {
        if (id) client.postMessage({ type: "act", action: action, id: id });
        return client.focus();
      }
      var target = "./" + (id ? "?act=" + encodeURIComponent(action) + "&id=" + encodeURIComponent(id) : "");
      return self.clients.openWindow(target);
    })
  );
});
