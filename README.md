# GPT Webex RoomOs

A playtime (internal hackathon) project for Cisco Webex RoomOs devices. Its a proof of concept hacked together, so do not expect it work in all cases.

Web app is hosted at: https://gpt_webex.surge.sh
To use, simply add it as a web app on your device. ([See here](https://help.webex.com/en-us/article/7ypsyc/Enable-Web-App-Management-on-Boards-and-Desk-Series-devices) for info on how to do that).

## Demo

https://github.com/ankursharma319/gpt_webex_roomos/assets/20162355/75ad5ea7-9feb-4075-8f1f-c4eeccf9de1c

Recorded on a Cisco Webex Desk Pro. Shows a simple command to show that it can work. It does not work in all cases mostly because the chatgpt model is not finetuned well enough on xapi schemas and jsxapi code.

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
