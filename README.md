# (function () {
  // 🔒 Registry to track all listeners
  const listenerRegistry = new Map();

  // 🧠 Monkey-patch addEventListener to auto-track
  const originalAddEventListener = EventTarget.prototype.addEventListener;
  EventTarget.prototype.addEventListener = function (type, listener, options) {
    const key = `${this}_${type}`;
    if (!listenerRegistry.has(key)) listenerRegistry.set(key, []);
    listenerRegistry.get(key).push({ listener, options });
    originalAddEventListener.call(this, type, listener, options);
  };

  // 🧹 Remove tracked listeners
  function removeTrackedListeners(element, type) {
    const key = `${element}_${type}`;
    const listeners = listenerRegistry.get(key);
    if (listeners) {
      listeners.forEach(({ listener, options }) => {
        element.removeEventListener(type, listener, options);
      });
      listenerRegistry.delete(key);
    }
  }

  // ✅ Restore clipboard behavior
  const forceBrowserDefault = function (e) {
    e.stopImmediatePropagation();
    return true;
  };

  // 🧼 Remove inline handlers
  const inlineHandlers = [
    'onblur', 'onfocus',
    'oncopy', 'oncut', 'onpaste',
    'onvisibilitychange', 'onfullscreenchange'
  ];
  [window, document].forEach(target => {
    inlineHandlers.forEach(handler => {
      if (handler in target) target[handler] = null;
    });
  });

  // 🧽 Clean tracked listeners
  const eventsToClean = [
    'copy', 'cut', 'paste',
    'blur', 'focus',
    'fullscreenchange',
    'visibilitychange'
  ];
  const targets = [window, document];
  eventsToClean.forEach(event => {
    targets.forEach(target => removeTrackedListeners(target, event));
  });

  // 🧼 Fallback: Remove listeners via DevTools API (if available)
  function removeAllListeners(element, eventType) {
    if (typeof getEventListeners === 'function') {
      const eventListeners = getEventListeners(element)[eventType];
      if (eventListeners && eventListeners.length > 0) {
        eventListeners.forEach(({ listener, useCapture }) => {
          element.removeEventListener(eventType, listener, useCapture);
        });
      }
    }
  }

  eventsToClean.forEach(event => {
    removeAllListeners(window, event);
    removeAllListeners(document, event);
  });

  // 🛡️ Prevent tab switch detection
  document.addEventListener('visibilitychange', forceBrowserDefault, true);
  window.addEventListener('blur', forceBrowserDefault, true);
  window.addEventListener('focus', forceBrowserDefault, true);

  // ✅ Restore clipboard behavior
  ['copy', 'cut', 'paste'].forEach(event => {
    document.addEventListener(event, forceBrowserDefault, true);
  });

  // 🕶️ Spoof fullscreen detection
  Object.defineProperty(document, 'fullscreenElement', { get: () => null });
  Object.defineProperty(document, 'fullscreenEnabled', { get: () => false });
  Object.defineProperty(document, 'webkitFullscreenElement', { get: () => null });
  Object.defineProperty(document, 'webkitFullscreenEnabled', { get: () => false });

  // 🚫 Block fullscreen requests
  Element.prototype.requestFullscreen = function () {
    console.log('requestFullscreen blocked');
  };
  document.exitFullscreen = function () {
    console.log('exitFullscreen blocked');
  };
})();
