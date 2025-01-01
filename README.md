# Youtube-Video-Ads-Blocker-FF-chrome-extension
- Our app blocks some Youtube ads, but not well enough yet (it still occasionally shows pop-ups, and has various errors on Youtube)
- We want to make sure the extension is working via manifest v3 (since V2 will be phased out)
- Clean up the code where possible, and make the extension perform faster
- Check filterlists to make sure they're being dynamically loaded properly
- Clean up various typos and small errors associated with the app
---------
To create a Firefox/Chrome extension that blocks YouTube ads, we can follow a few steps to improve and clean up the code based on your requirements. Here's an approach to handle manifest v3, optimize performance, and address your concerns.
1. Update manifest.json for Manifest v3

Since Manifest v2 will be phased out, we need to ensure the extension is compatible with manifest v3. This includes using service workers and properly declaring permissions, actions, and content scripts.

{
  "manifest_version": 3,
  "name": "YouTube Ad Blocker",
  "description": "Block YouTube ads more efficiently and clean up errors.",
  "version": "1.0",
  "permissions": [
    "activeTab",
    "storage",
    "webNavigation",
    "webRequest",
    "webRequestBlocking",
    "tabs"
  ],
  "background": {
    "service_worker": "background.js"
  },
  "content_scripts": [
    {
      "matches": ["https://www.youtube.com/*"],
      "js": ["content.js"],
      "run_at": "document_start"
    }
  ],
  "host_permissions": [
    "https://www.youtube.com/*"
  ],
  "action": {
    "default_popup": "popup.html",
    "default_icon": {
      "16": "assets/icon-16.png",
      "48": "assets/icon-48.png",
      "128": "assets/icon-128.png"
    }
  },
  "icons": {
    "16": "assets/icon-16.png",
    "48": "assets/icon-48.png",
    "128": "assets/icon-128.png"
  }
}

2. Background Script (background.js)

In Manifest v3, we use service workers instead of background pages. We’ll dynamically load filter lists and improve performance by managing ad-blocking rules more efficiently.

// background.js (Service Worker)
let blockList = [];

const loadFilterList = () => {
  fetch('https://example.com/ads-filterlist.txt') // URL of your filter list
    .then(response => response.text())
    .then(data => {
      blockList = data.split("\n").filter(line => line.trim() !== "");
      console.log('Filter list loaded:', blockList.length, 'rules.');
    })
    .catch(error => console.error('Failed to load filter list:', error));
};

// Run filter list load when the service worker starts
loadFilterList();

// Listen for request to block ads
chrome.webRequest.onBeforeRequest.addListener(
  (details) => {
    const url = details.url;
    // Block URLs based on filter list
    if (blockList.some(filter => url.includes(filter))) {
      return { cancel: true };
    }
  },
  { urls: ["*://*.youtube.com/*"] },
  ["blocking"]
);

3. Content Script (content.js)

We will enhance the content script to block pop-ups and ensure ad blocking happens as soon as possible. Here we will listen for network requests and prevent pop-ups by manipulating the DOM.

// content.js
const blockAds = () => {
  // Find and hide any elements that are ads
  const adElements = document.querySelectorAll(".ad-container, .ytp-ad-player-overlay");
  adElements.forEach(ad => {
    ad.style.display = "none";
  });

  // Block iframe or embedded ad elements
  const iframeAds = document.querySelectorAll("iframe[src*='ads']");
  iframeAds.forEach(iframe => iframe.style.display = "none");
};

// Wait for the page to load and block any ads
window.addEventListener('load', () => {
  blockAds();
  // Additional cleanup after a few seconds if necessary
  setTimeout(blockAds, 5000);
});

// Listen for URL changes (e.g., when YouTube videos switch)
window.addEventListener('yt-navigate-finish', () => {
  blockAds();
});

4. Optimizing Performance

    Reduce Redundant Operations: Make sure that filtering and ad-blocking tasks are only performed when necessary.
    Lazy Loading: Use dynamic loading for your filter lists, so they aren’t loaded unnecessarily on each page load.
    Efficient DOM Manipulation: Avoid constantly checking or modifying the DOM unless something changes (e.g., URL change, video change).
    Throttle Requests: Don't repeatedly load and filter URLs more than needed. Block them once and cache the result.

5. Popup HTML (popup.html)

In the popup, the user can see whether the extension is working, turn it on or off, or view logs. This helps in debugging and improving user experience.

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>YouTube Ad Blocker</title>
  <style>
    body { font-family: Arial, sans-serif; margin: 10px; }
    h1 { font-size: 16px; }
    button { padding: 8px 16px; margin: 10px 0; cursor: pointer; }
  </style>
</head>
<body>
  <h1>YouTube Ad Blocker</h1>
  <button id="toggleBlock">Disable Blocking</button>
  <p>Status: <span id="status">Active</span></p>
  <script src="popup.js"></script>
</body>
</html>

6. Popup Script (popup.js)

Here, we handle the logic for enabling or disabling ad-blocking from the popup.

// popup.js
const statusText = document.getElementById('status');
const toggleButton = document.getElementById('toggleBlock');

// Initially set status
chrome.storage.local.get(['blockAdsEnabled'], (result) => {
  if (result.blockAdsEnabled === false) {
    statusText.textContent = 'Inactive';
    toggleButton.textContent = 'Enable Blocking';
  } else {
    statusText.textContent = 'Active';
    toggleButton.textContent = 'Disable Blocking';
  }
});

// Toggle block ads status
toggleButton.addEventListener('click', () => {
  chrome.storage.local.get(['blockAdsEnabled'], (result) => {
    const newStatus = !(result.blockAdsEnabled === false);
    chrome.storage.local.set({ blockAdsEnabled: newStatus });
    if (newStatus) {
      statusText.textContent = 'Active';
      toggleButton.textContent = 'Disable Blocking';
    } else {
      statusText.textContent = 'Inactive';
      toggleButton.textContent = 'Enable Blocking';
    }
  });
});

7. Handling Filter Lists Dynamically

To ensure the filter lists are being loaded properly, you can monitor and verify them with simple logging in the background.js file.

// background.js - logging filter list load success
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.action === 'checkFilterList') {
    if (blockList.length > 0) {
      sendResponse({ status: 'Filter list loaded successfully.' });
    } else {
      sendResponse({ status: 'Filter list not loaded.' });
    }
  }
});

8. Improving Speed

    Caching Filters: Use chrome.storage.local to cache filter lists, so they aren’t fetched every time the extension starts.

// background.js - Cache the filter list
chrome.storage.local.get(['adsFilterList'], (result) => {
  if (result.adsFilterList) {
    blockList = result.adsFilterList;
    console.log('Using cached filter list');
  } else {
    loadFilterList();
  }
});

9. Fixing Typos and Errors

Ensure that all variable names are consistent and there are no misplaced or forgotten semicolons, incorrect function calls, or unused variables. Use a linter like ESLint to automatically catch most common issues.
Conclusion

By following the above code snippets and principles, you should have a cleaner, faster, and more reliable YouTube ad-blocking extension. The use of Manifest v3, service workers, content scripts, and background processes ensures that the extension performs well and is ready for future updates to Chrome and Firefox.

If you're building the extension for both Chrome and Firefox, you should test both browsers since there may be minor differences in behavior (especially around permissions, background workers, etc.).
