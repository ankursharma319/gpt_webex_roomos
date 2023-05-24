<script lang="ts">
    /* global globalThis, RequestInit */
    import * as jsxapi from 'jsxapi';

	let input_text = 'Open a webview containing google.com';
	let device_ip = '192.168.10.184';
	let device_username = 'admin';
	let device_password = '';
    let last_cmd_error = "TBD";
    let last_cmd_js_code = "TBD";
    let open_api_auth_key = "";
    let open_api_llm_model = "gpt-3.5-turbo";
    let mediaRecorder: MediaRecorder;

	async function askChatGpt(): Promise<void> {
        last_cmd_error = "Asking ChatGPT to interpret command into code";
		console.log('askChatGpt');
        const headers1 = new Headers();
        headers1.append("Content-Type", "application/json");
        headers1.append("Authorization", "Bearer " + open_api_auth_key);
        const options1: RequestInit = {
            method: "GET", // *GET, POST, PUT, DELETE, etc.
            mode: "cors", // no-cors, *cors, same-origin
            cache: "default", // *default, no-cache, reload, force-cache, only-if-cached
            // credentials: "same-origin", // include, *same-origin, omit
            headers: headers1,
            // redirect: "follow", // manual, *follow, error
            // referrerPolicy: "no-referrer", // no-referrer, *no-referrer-when-downgrade, origin, origin-when-cross-origin, same-origin, strict-origin, strict-origin-when-cross-origin, unsafe-url
            // body: JSON.stringify(data), // body data type must match "Content-Type" header
        };
		const res1 = await fetch(new Request(`https://api.openai.com/v1/models/gpt-4`, options1)).catch((err) => {
            console.warn("Failed to list info about gpt-4 model");
            last_cmd_error = "Failed to list info about gpt-4 model: " + err;
        });
        console.log("received res1", res1);

        const template = `
const jsxapi = require('jsxapi');

// Connect to the device
const xapi = jsxapi.connect('ssh://10.10.10.10', {
  username: 'admin',
  password: 'password',
});

xapi.on('error', (err) => {
  console.error("connection failed", err);
  process.exit(1);
});

xapi.on('ready', async () => {
  console.log('connection successful');

  // TODO rest of the code goes here
});
`;
        const headers2 = new Headers();
        headers2.append("Content-Type", "application/json");
        headers2.append("Authorization", "Bearer " + open_api_auth_key);
        const options2: RequestInit = {
            method: "POST", // *GET, POST, PUT, DELETE, etc.
            mode: "cors", // no-cors, *cors, same-origin
            cache: "default", // *default, no-cache, reload, force-cache, only-if-cached
            // credentials: "same-origin", // include, *same-origin, omit
            headers: headers2,
            // redirect: "follow", // manual, *follow, error
            // referrerPolicy: "no-referrer", // no-referrer, *no-referrer-when-downgrade, origin, origin-when-cross-origin, same-origin, strict-origin, strict-origin-when-cross-origin, unsafe-url
            body: JSON.stringify({
                model: open_api_llm_model,
                messages: [
                    { role: "system", content: "Process the following queries in the context of xapi for webex devices "
                        + "and only return javascript code which uses jsxapi. Make sure to include await keyword when executing xapi requests. "
                        + "Assume that the following code is already written and return only the code that is "
                        + "supposed to go in the xapi connection success callback. I repeat, do not return rest of the template code. ```javascript\n" + template + "```\n",
                        name: "my_app_frontend"
                    },
                    { role: "user", content: input_text, name: "my_app_frontend"},
                ],
            }),
        };
        const url2 = "https://api.openai.com/v1/chat/completions";
        console.log("making request req2");
		const res2 = await fetch(new Request(url2, options2)).catch((err) => {
            console.warn("Failed to successfully get completion from openai chatgpt model");
            last_cmd_error = "Failed to get completion from chatgpt model: " + err;
        });
        console.log("received res2", res2);
        if (!res2) {
            throw new Error("res2 was void");
        }
        const res2_str = await res2.clone().text();
        const res2_json = await res2.json();
        let completion = "";
        try {
            completion = res2_json["choices"].length >= 1 ? res2_json["choices"][0].message.content : "";
            if (!completion) {
                throw Error("bad parse");
            }
            console.log("Got completion: ", completion);

        } catch (ex: any) {
            last_cmd_error = "Failed to parse completion from chatgpt response: " + res2_str;
            throw new Error("Wasnt able to parse the completion from chatgpt properly");
        }
        let my_regex = /```(?:javascript)?\n([\s\S]*?)\n```/g;

        let m;
        let codes = [];

        while ((m = my_regex.exec(completion)) !== null) {
            // This is necessary to avoid infinite loops with zero-width matches
            if (m.index === my_regex.lastIndex) {
                my_regex.lastIndex++;
            }

            // Push the matched text (group 1) to the codes array
            codes.push(m[1]);
        }

        console.log(codes);
        // let matches = completion.match(/```(javascript)?(.*?)```/);  //returns array
        // console.log("matches = ", matches)
        do_execute_js(codes[0]);
	}

    async function do_execute_js(js_to_exec: string) {

        last_cmd_error = "Executing javascript code on device";
        // Connecting using WebSockets
        jsxapi
            .connect('wss://' + device_ip, {
                username: device_username,
                password: device_password,
            })
            .on('error', (e) => {
                console.error("Caught error myself");
                console.info(e);
            })
            .on('ready', async (xapi: jsxapi.XAPI) => {
                // const volume = await xapi.status.get('Audio Volume');
                // console.log(`volume is: ${volume}`);
                console.log("Evalling ctxScript");
                globalThis["xapi"] = xapi;
                last_cmd_js_code = js_to_exec;
                try {
                    await Object.getPrototypeOf(async function() { return; }).constructor(js_to_exec)();
                    console.log("Finished evalling ctxScript");
                    last_cmd_error = "";
                } catch (err: any) {
                    console.warn("failed to execute code, got error", err);
                    last_cmd_error = JSON.stringify(err);
                }
                xapi.close();
            });
    }
    async function executeJs(e: MouseEvent) {
        console.log("clicked executeJs", e);
        let test_js_code = `
console.log("volume is something");
const volume = await xapi.status.get('Audio Volume');
console.log("volume is: " + volume);
// throw new Error("some error");
`;
        test_js_code = input_text;
        await do_execute_js(test_js_code);

    }

	async function sendAudioToServer(audioBlob: any) {
        last_cmd_error = "Transcribing audio";
        console.log("sendAudioDataToServer of size", audioBlob.size);
        let formdata = new FormData();
        formdata.append('file', audioBlob, "audio.webm");
        formdata.append("model", "whisper-1");
        formdata.append("language", "en");
        const headers2 = new Headers();
        // headers2.append("Content-Type", "application/json");
        headers2.append("Authorization", "Bearer " + open_api_auth_key);
        const options2: RequestInit = {
            method: "POST", // *GET, POST, PUT, DELETE, etc.
            mode: "cors", // no-cors, *cors, same-origin
            cache: "default", // *default, no-cache, reload, force-cache, only-if-cached
            // credentials: "same-origin", // include, *same-origin, omit
            headers: headers2,
            // redirect: "follow", // manual, *follow, error
            // referrerPolicy: "no-referrer", // no-referrer, *no-referrer-when-downgrade, origin, origin-when-cross-origin, same-origin, strict-origin, strict-origin-when-cross-origin, unsafe-url
            body: formdata,
        };
        const url2 = "https://api.openai.com/v1/audio/transcriptions";
        console.log("making transcibe request");
		const res2 = await fetch(new Request(url2, options2)).catch((err) => {
            console.warn("Failed to successfully get transcript from openai whisper model");
            last_cmd_error = "Failed to get transcript from openai transcript model: " + err;
        });
        console.log("received res2", res2);
        if (!res2) {
            throw new Error("res2 was void");
        }
        const res2_str = await res2.clone().text();
        const res2_json = await res2.json();
        console.log("received res2_str", res2_str);
        input_text = res2_json["text"];
        askChatGpt();
    }

    async function askChatGptVoice(): Promise<void> {

        last_cmd_error = "Recording voice";
        console.log("askChatGptVoice clicked");
        if (mediaRecorder && mediaRecorder.state == "recording") {
            console.log("Stopping mediaRecorder");
            mediaRecorder.stop();
            return;
        }
        navigator.mediaDevices.getUserMedia({ audio: true, video: false })
            .then(stream => {
                const options = {
                    audioBitsPerSecond: 128000,
                    mimeType: "audio/webm",
                };
                mediaRecorder = new MediaRecorder(stream, options);
                mediaRecorder.start();

                let audioChunks : any[] = [];
                mediaRecorder.addEventListener("dataavailable", event => {
                    audioChunks.push(event.data);
                });

                mediaRecorder.addEventListener("stop", async () => {
                    console.log("Stopped mediaRecorder, mime type =", mediaRecorder.mimeType);
                    const audioBlob = new Blob(audioChunks, { type: "audio/webm" });
                    await sendAudioToServer(audioBlob);
                });

            });
    }

</script>

<svelte:head>
	<title>Home</title>
	<meta name="description" content="Svelte demo app" />
</svelte:head>

<section class="flex flex-col h-[600px] justify-start items-center w-screen gap-3 m-4">
    <div class="flex flex-row gap-4">
	    <label>
            <span>Device: </span>
            <input type="text" bind:value={device_ip} class={"border-2 p-1 " + (device_ip ? "border-slate-600" : "border-red-600")}/>
        </label>
	    <label>
            <span>Username: </span>
            <input type="text" bind:value={device_username} class={"border-2 p-1 " + (device_username ? "border-slate-600" : "border-red-600")}/>
        </label>
	    <label>
            <span>Password: </span>
            <input type="password" bind:value={device_password} class={"border-2 p-1 " + (device_password ? "border-slate-600" : "border-red-600")}/>
        </label>
    </div>
    <div class="flex flex-row gap-4">
	    <label class="">
            <span>OpenAI Api Key: </span>
            <input type="password" bind:value={open_api_auth_key}
                class={"border-2 p-1 " + (open_api_auth_key ? "border-slate-700" : "border-red-500")}
            />
        </label>
	    <label class="">
            <span>OpenAI LLM Model: </span>
            <input type="text" bind:value={open_api_llm_model} class="border-2 border-slate-700 p-1"/>
        </label>
    </div>
    <br/>
	<label class="w-full text-center">
		<textarea
			bind:value={input_text}
			class="text-md w-4/5 border-2 border-slate-500"
			cols="100"
			rows="5"
		/>
	</label>
	<div class="flex flex-row gap-2">
		<button on:click={askChatGptVoice} class="p-2 border-slate-600 border-2 w-48 text-center bg-slate-200">
            { mediaRecorder && mediaRecorder.state == "recording" ? "Stop recording" : "Ask ChatGPT voice"}
		</button>
		<button on:click={askChatGpt} class="p-2 border-slate-600 border-2 w-48 text-center bg-slate-200">
			Ask ChatGPT text
		</button>
		<button on:click={executeJs} class="p-2 border-slate-600 border-2 w-48 text-center bg-slate-200">
			Execute JS on device
		</button>
	</div>
    <div>
        Last cmd status:
        <span class={"p-2 " + (last_cmd_error ? "text-red-700" : "text-green-700")}>{ last_cmd_error ? last_cmd_error: "OK"}</span>
    </div>
    <div class="p-2 whitespace-pre-line">
        Last cmd js code:<br>
        <div class="p-2 text-md border-2 border-slate-300">
            <code>{last_cmd_js_code}</code>
        </div>
    </div>
</section>
