+++
title = 'Message passing in Chrome extension (2024)'
description = "Learn how to communicate between parts of a Chrome extension"
image = "message-passing-headline.png"
date = 2024-06-12
categories = ["Software Development"]
tags = ["Chrome Extension", "Message passing", "Web Development"]
draft = false
+++

This article is part of a series on [building a production-ready Chrome extension](../chrome-extension).

In the previous article, we [set up our Chrome extension with TypeScript support and the webpack bundler](../add-webpack-and-typescript-to-chrome-extension). This article will build on that code, dive into the new APIs, and cover message-passing communication between different parts of our Chrome extension.

## Communication between parts of a Chrome extension

As we covered in [the first article](../create-chrome-extension), a Chrome extension consists of three main parts:
- service worker (background script)
- content script
- popup

These parts need to communicate with each other. For example, a popup needs to send a message to a content script to change the appearance of a webpage. Or a background script needs to send a message to a popup to update the user interface based on the page that's being visited.

{{< figure src="message-passing.svg" title="Communication in a Chrome extension" alt="Communication in a Chrome extension" >}}

One way to communicate between these parts is to use the local storage via the `chrome.storage` APIs. We do not recommend this method because it is slow and can cause performance issues. This method is slow because it is not synchronous -- the scripts need to check the storage for changes periodically. A better way to communicate between extension parts is to use message passing.

## What is message passing?

In computer science, message passing is a method for communicating between different processes or threads. A process or thread sends a message to another process or thread, which receives the message and acts on it. This method is often used in distributed systems, where processes run on different machines and need to communicate with each other. The sender sends a message, and the receiver decodes it and executes the appropriate code.

## Message passing in a Chrome extension

Message passing is a way to communicate between different parts of a Chrome extension. Its main advantage is that it's fast and efficient. When a message is sent, the receiver gets it immediately and can respond to it right away.

Message passing is done in Chrome extensions using the `chrome.runtime.sendMessage`, `chrome.tabs.sendMessage` and `chrome.runtime.onMessage` functions. Here's how it works:

1. The sender calls `chrome.runtime.sendMessage` or `chrome.tabs.sendMessage` with the message to send.
2. The receiver listens for messages using `chrome.runtime.onMessage.addListener`
3. The receiver processes the incoming message and, optionally, responds to the message.

## Message passing from a popup to a content script

Let's see how we can use message passing to communicate between a popup and a content script. We will send a message when the user toggles the enable slider in the popup, which will enable or disable the content script's processing. We will use the `chrome.tabs.sendMessage` function to send a message to a specific tab ID.

In the popup script (`popup.ts`), we send a message to all the tabs when we detect a change in the top slider:

```typescript
// Send message to content script in all tabs
const tabs = await chrome.tabs.query({})
for (const tab of tabs) {
    // Note: sensitive tab properties such as tab.title or tab.url can only be accessed for
    // URLs in the host_permissions section of manifest.json
    chrome.tabs.sendMessage(tab.id!, {enabled: event.target.checked})
        .then((response) => {
            console.info("Popup received response from tab with title '%s' and url %s",
                response.title, response.url)
        })
        .catch((error) => {
            console.warn("Popup could not send message to tab %d", tab.id, error)
        })
}
```

In the content script (`content.ts`), we listen for the message and process it:

```typescript
// Listen for messages from popup.
chrome.runtime.onMessage.addListener((request, sender, sendResponse) => {
    if (request.enabled !== undefined) {
        console.log("Received message from sender %s", sender.id, request)
        enabled = request.enabled
        if (enabled) {
            observe()
        } else {
            observer.disconnect()
        }
        sendResponse({title: document.title, url: window.location.href})
    }
})
```

When the user toggles the slider in the popup, the popup sends a message to all tabs. The receiving tab will print this message to the Chrome Developer Tools console.

{{< figure src="content-script-received.png" title="Content script received message" alt="Content script received message" >}}

Then, the popup will receive a response from the content script with the tab's title and URL. This response prints to the `Inspect Popup` console. System tabs like `chrome://extensions/` will not respond to messages.

{{< figure src="popup-response.png" title="Popup received response" alt="Popup received response" >}}

## Message passing from a popup to the service worker (background script)

To send a message to the service worker, we must use the `chrome.runtime.sendMessage` function instead of `chrome.tabs.sendMessage`. The service worker does not have a tab ID, so we cannot use `chrome.tabs.sendMessage`.

```typescript
chrome.runtime.sendMessage({enabled: event.target.checked})
    .then((response) => {
        console.info("Popup received response", response)
    })
    .catch((error) => {
        console.warn("Popup could not send message", error)
    })
```

In the service worker script (`background.ts`), we listen for the message and process it:

```typescript
chrome.runtime.onMessage.addListener((request, sender, sendResponse) => {
    if (request.enabled !== undefined) {
        console.log("Service worker received message from sender %s", sender.id, request)
        sendResponse({message: "Service worker processed the message"})
    }
})
```

## Message passing from a content script to the popup and service worker

To send a message from the content script, use the `chrome.runtime.sendMessage` function. The popup and service worker can listen and receive this message.

## Message passing from the service worker (background script) to a content script and the popup

Use the `chrome.tabs.sendMessage` function to send a message to the content script. Use the `chrome.runtime.sendMessage` function to send a message to the popup.

The code for sending a message from the service worker is the same as the code for sending a message from the popup. The receiving code in the content and popup scripts is also the same.

## Next steps

In the next part of this series, we will add linting and formatting tools to our Chrome extension. These tools increase the quality of our code and increase engineering velocity for projects with multiple developers.

## Chrome extension with webpack and TypeScript code on GitHub

The complete code is available on GitHub at: https://github.com/getvictor/create-chrome-extension/tree/main/3-message-passing

## Message passing in a Chrome extension video

{{< youtube qANlZ5kzxcg >}}

*Note:* If you want to comment on this article, please do so on the YouTube video.
