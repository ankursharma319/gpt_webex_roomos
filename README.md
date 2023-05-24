# GPT Webex RoomOs

A playtime (internal hackathon) project for Cisco Webex RoomOs devices. Its a proof of concept hacked together, so do not expect it work in all cases.

Web app is hosted at: https://gpt_webex.surge.sh

## Developing

```bash
nix develop
npm install
npm run dev
```

## Building

To create a production version of your app:

```bash
npm run build
```

You can preview the production build with `npm run preview`.

> To deploy your app, you may need to install an [adapter](https://kit.svelte.dev/docs/adapters) for your target environment.

## Deploying on surge

```bash
nix develop
npm run build
surge ./build/ gpt_webex.surge.sh
```
