# Product: Restore Electron Devtron Extension

## Preamble

I am deeply grateful to my mentors [@David Sanders](https://github.com/dsanders11), [@Erick Zhao](https://github.com/erickzhao), [@Samuel Maddock](https://github.com/samuelmaddock), [@Anny Yang](https://github.com/yangannyx) and the entire Electron team for their patience, guidance, and thoughtful feedback on my pull requests.

Your encouragement and support throughout Google Summer of Code has been invaluable, and I truly appreciate the time and care you’ve invested in helping me grow.

## Contact Information

- **Name:** Hitarth Singh Rajput
- **Email:** [goforhitarth@gmail.com](mailto:goforhitarth@gmail.com)
- **Phone:** [+91 6264658180](tel:+916264658180)
- **Location:** Lucknow, Uttar Pradesh, India

### Social Media

- **GitHub:** https://github.com/hitarth-gg
- **LinkedIn:** https://www.linkedin.com/in/hitarth-rajput/
- **Discord:** hitarth.

## Overview

Electron is an open-source framework that lets developers build desktop applications with web technologies. By embedding **Chromium** and **Node.js** into its binary, Electron allows you to maintain one JavaScript codebase and create cross-platform apps that work on Windows, macOS, and Linux.

Many widely used desktop applications such as [VS Code](https://code.visualstudio.com/), [Slack](https://slack.com/), [Discord](https://discord.com/), and [Notion](https://www.notion.com/), are built with Electron.

During [Google Summer of Code 2025](https://summerofcode.withgoogle.com/), I rewrote [Devtron](https://github.com/electron/devtron), a decade old developer tool for Electron, from scratch using modern frameworks and libraries, while accounting for the significant changes in Electron APIs over the years and the transition to Chrome’s [Manifest Version 3](https://developer.chrome.com/docs/extensions/develop/migrate/what-is-mv3).

The [original Devtron](https://github.com/electron/devtron/tree/legacy) extension had become non-functional over the past ten years due to evolving APIs in both Chrome and Electron, as well as the release of newer manifest versions.

This work was especially meaningful to me because it provides newcomers with a clearer understanding of how IPC events flow between different processes in an Electron app. At the same time, it serves as a developer tool that helps developers keep track of all the IPC events occurring within the applications they are building.

An issue tracking progress, known issues and next steps can be found here: [electron/devtron#272](https://github.com/electron/devtron/issues/272)

## Details

This section highlights the development process and some of the challenges encountered.

I had submitted my GSoC proposal with a prototype that tracked IPC events sent and received by `ipcRenderer` and `ipcMain` . This served as a good starting point, and the prototype was later refined and converted into a pull request.

However, the prototype was initially written entirely in JavaScript and later converted to TypeScript in a follow-up pull request to improve long-term maintainability.

### How IPC events are tracked

My initial approach to tracking IPC events was to patch the `ipcMain` and `ipcRenderer` methods, attaching a tracker that intercepted the requests they received.

However, my mentor [@Samuel Maddock](https://github.com/samuelmaddock) pointed out in a [comment](https://github.com/electron/devtron/pull/265#discussion_r2141206726) that this approach would track `ipcMain` but miss several other channels, such as `webContents.ipc`, `webFrameMain.ipc`, and `serviceWorkerMain.ipc`.

The solution was to attach listeners to the `Session` object, which allowed capturing IPC messages from multiple sources. For example:

```ts
// ses = defaultSession or a newly created session
ses.on(
  "-ipc-message",
  (
    event: Electron.IpcMainEvent | Electron.IpcMainServiceWorkerEvent,
    channel: Channel,
    args: any[]
  ) => {
    if (event.type === "frame")
      trackIpcEvent({
        direction: "renderer-to-main",
        channel,
        args,
        devtronSW,
      });
    else if (event.type === "service-worker")
      trackIpcEvent({
        direction: "service-worker-to-main",
        channel,
        args,
        devtronSW,
      });
  }
);
```

These `.on` listeners are attached to three channels: `-ipc-message`, `-ipc-invoke`, and `-ipc-message-sync`. This approach also makes it possible to track IPC events sent by service workers to Electron’s main process.

To track IPC events flowing through `ipcRenderer`, Devtron still patches its various methods to attach trackers.

Initially, this was achieved by manually importing and executing `monitorRenderer()` from Devtron in the renderer preload scripts where IPC tracking was needed. With `sandbox: false`, this worked fine. However, when `sandbox: true` was set in the `webPreferences` of an Electron app (the default setting), preload scripts were restricted in what they could `import` or `require`, which caused issues when trying to load the Devtron module.

This issue was resolved by registering the file that patched `ipcRenderer` as a preload script in the main process of electron using [`ses.registerPreloadScript()`](https://www.electronjs.org/docs/latest/api/session#sesregisterpreloadscriptscript).

With this change, Devtron can now be installed with just a few lines of code:

```ts
// main.js
const { devtron } = require("@electron/devtron");
// or import { devtron } from '@electron/devtron'

devtron.install(); // call this function at the top of your file
```

Devtron can also be installed conditionally. For example:

```ts
const { app } = require("electron");

const isDev = !app.isPackaged;

async function installDevtron() {
  const { devtron } = await import("@electron/devtron");
  await devtron.install();
}

if (isDev) {
  installDevtron().catch((error) => {
    console.error("Failed to install Devtron:", error);
  });
}
```

Messages sent from the main process to service workers using [`serviceWorker.send(channel, ...args)`](https://www.electronjs.org/docs/latest/api/service-worker-main#serviceworkersendchannel-args-experimental) had to be tracked separately by patching the `.send` method.

In one of the weekly Slack huddles, [@Samuel Maddock](https://github.com/samuelmaddock/) suggested that it’d be nice to track the time it takes for the [`ipcRenderer.invoke(channel, ...args)`](https://www.electronjs.org/docs/latest/api/ipc-renderer#ipcrendererinvokechannel-args) and [`ipcRenderer.sendSync(channel, ...args)`](https://www.electronjs.org/docs/latest/api/ipc-renderer#ipcrenderersendsyncchannel-args) to receive back a response from the main process.

This was a great addition to Devtron, implemented using UUIDs. The `renderer` process generates a UUID and attaches it to `.invoke` and `.sendSync` requests. When these requests reach the main process, the UUID is extracted from the arguments. Since both the renderer and main processes are now using the same identifier, events can be reliably linked together.

On the frontend, Devtron links `request` and `response` events using the same UUID, allowing users to easily navigate back and forth between them.

<img width="1283" height="423" alt="devtron-1" src="https://gist.github.com/user-attachments/assets/5cd0ccde-94ec-44e9-bdac-602dab36d338" />

### Devtron Extension

The Devtron extension is built with [Manifest Version 3](https://developer.chrome.com/docs/extensions/develop/migrate/what-is-mv3) in mind. The frontend for Devtron’s DevTools Panel is built using [React](https://react.dev/) and [TailwindCSS](https://tailwindcss.com/). To display the tracked events in the panel, I decided to use the community version of [AG Grid](https://www.ag-grid.com/). Since it comes with virtualization, thousands of rows can be stored and displayed efficiently by rendering only the rows visible to the user.

Some additional features were also added, such as light and dark theme support, the ability to dock the Details Panel to the bottom or the right, and a “lock to bottom” option that automatically scrolls to newly added events.

<img width="1284" height="581" alt="devtron-2" src="https://gist.github.com/user-attachments/assets/961f3a7c-f6e9-458e-8b95-558ea4b1017c" />

<img width="1035" height="545" alt="devtron-4" src="https://gist.github.com/user-attachments/assets/d2138453-108c-4bf6-8f21-ca70c0d7f745" />

One challenge encountered during development was that Devtron’s Service Worker would terminate after a period of inactivity. This was resolved by keeping it alive using [`serviceWorker.startTask()`](https://www.electronjs.org/docs/latest/api/service-worker-main#serviceworkerstarttask-experimental) .

### Building the package

For packaging Devtron, I chose [Webpack](https://webpack.js.org/), enabling bundling into both `ESM` and `CommonJS` formats to ensure compatibility across different module systems. <br/>
Webpack’s [`DefinePlugin()`](https://webpack.js.org/plugins/define-plugin/) is also used to replace `__dirname` with `import.meta.url`, allowing correct resolution of file paths in the ESM context.

```ts
// src/index.ts
const dirname = __dirname; // __dirname is replaced with import.meta.url in ESM builds using webpack
const serviceWorkerPreloadPath = createRequire(dirname).resolve(
  // resolves the path as defined in Devtron's package.json "exports"
  "@electron/devtron/service-worker-preload"
);
```

## Contributions

Below is a list of all contributions made to [electron/devtron](https://github.com/electron/devtron):

### Merged Pull Requests

[#265](https://github.com/electron/devtron/pull/265): chore: initial setup with prototype <br/>
[#266](https://github.com/electron/devtron/pull/266): chore: migrate to TypeScript <br/>
[#268](https://github.com/electron/devtron/pull/268): feat: use Session to track IPCs and improve UI <br/>
[#269](https://github.com/electron/devtron/pull/269): fix: commonjs compatibility <br/>
[#270](https://github.com/electron/devtron/pull/270): feat: raise event display limit to 20k <br/>
[#271](https://github.com/electron/devtron/pull/271): feat: remove need to manually call `monitorRenderer` <br/>
[#273](https://github.com/electron/devtron/pull/273): feat: track IPC events sent from main to service worker <br/>
[#274](https://github.com/electron/devtron/pull/274): feat: add tooltips to header buttons <br/>
[#275](https://github.com/electron/devtron/pull/275): feat: track response time of `invoke` and `sendSync` methods on `ipcRenderer` <br/>
[#276](https://github.com/electron/devtron/pull/276): fix: type declarations not found and invalid `require` usage in ESM build <br/>
[#277](https://github.com/electron/devtron/pull/277): feat: default to system color scheme in UI <br/>
[#279](https://github.com/electron/devtron/pull/279): test: set up testing environment <br/>
[#280](https://github.com/electron/devtron/pull/280): feat: add `getEvents` API to retrieve tracked IPC events in main process <br/>
[#281](https://github.com/electron/devtron/pull/281): fix: correct return type of `getEvents()` function <br/>
[#284](https://github.com/electron/devtron/pull/284): feat: add filtering support to event grid <br/>
[#285](https://github.com/electron/devtron/pull/285): fix: move "types" field above "import" in package exports <br/>

### Open Pull Requests (at the time of writing)

[#282](https://github.com/electron/devtron/pull/282): test: set up Vitest and add tests for `src/index.ts`

- Initially, I opened two separate PRs to explore different testing approaches for Devtron. One used `mocha` and `chai` with a real Electron app to verify IPC events with Devtron’s Service Worker. This one ([#282](https://github.com/electron/devtron/pull/282)) uses `vitest`, mocking various Electron APIs to enable testing in a lighter environment.
- During a weekly Slack huddle, we decided to move forward with [#279](https://github.com/electron/devtron/pull/279). The `vitest` PR may now be closed.

[#283](https://github.com/electron/devtron/pull/283): feat: add configurable `logLevel` to install options

## Future Work

Add more tests to make sure everything’s covered and all features work reliably.

Since Devtron is primarily a developer tool for Electron, some features from the legacy Devtron, such as an Event Inspector for core Electron objects and tracking require calls for different modules could also be reintroduced in future updates.

## Closing Remarks

I’m really grateful for the chance to be a GSoC Contributor and work closely with the Electron team. Huge thanks to my mentors for their guidance, code reviews, and for patiently answering all my questions about the codebase and design approaches.

The last few months have been super productive. Building Devtron from scratch gave me a ton of hands-on experience and a much deeper understanding of Electron’s architecture.

I’ve also improved a lot as a developer, especially when it comes to writing clean, maintainable code.

Overall, this has been an amazing learning experience :)

Now that the GSoC coding phase is over, I’ll keep contributing to Devtron _unofficially_, squashing bugs and making it better as more people start using it.
