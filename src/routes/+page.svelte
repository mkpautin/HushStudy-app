<script>
	import { form } from "$app/server";
    import { PUBLIC_API_IP, PUBLIC_API_PORT } from '$env/static/public';
    import logo from "$lib/assets/logo.svg";
	import { normalizeUrl } from "@sveltejs/kit";

    let timerStart;

    let isRecording = $state(false);
    let mediaRecorder = null;
    let audioChunks = [];
    let audioUrl = $state(null);
    let isChatter = $state(null);
    let durationSecondsLeft = $state(null);
    let durationMinsLeft = $derived(Math.ceil(durationSecondsLeft / 60))
    let underThreshold = $state(false);
    let isAnalyzing = $state(false);
    let { chatterMessage, chatterSubMessage, bgColor} = $derived.by(() => {
        if (isRecording) {
            return {
                chatterMessage: "Recording...",
                chatterSubMessage: "Click the button to stop recording",
                bgColor: "bg-blue-400"
            };
        } else if (isAnalyzing) {
            return {
                chatterMessage: "Analyzing...",
                chatterSubMessage: "Processing your recording",
                bgColor: "bg-blue-400"
            };
        } else if (isChatter === false) {
            return {
                chatterMessage: "Non-Chatter",
                chatterSubMessage: "You could study here.",
                bgColor: "bg-green-500"
            };
        } else if (isChatter === true && durationSecondsLeft) {
            return {
                chatterMessage: "Chatter",
                chatterSubMessage: `This may last for ${durationMinsLeft} ${durationMinsLeft <= 1 ? "min" : "mins"}. ${underThreshold ? "You could study here." : "You may want to study somewhere else."}`,
                bgColor: underThreshold ? "bg-orange-500" : "bg-red-700"
            };
        } else {
            return {
                chatterMessage: "Welcome",
                chatterSubMessage: "Click the button to start recording.",
                bgColor: "bg-blue-400"
            };
        }
    })
    let recordErrorMessage = $state("")
    let adjustMessage = $state("")

    async function startRecording() {
        recordErrorMessage = "";
        adjustMessage = "";
        try {
            const stream = await navigator.mediaDevices.getUserMedia({audio: true});
            mediaRecorder = new MediaRecorder(stream, {
                mimeType: "audio/webm"
            });
            audioChunks = [];

            mediaRecorder.ondataavailable = (event) => {
                audioChunks.push(event.data);
            };
            
            console.log(mediaRecorder.mimeType);
            mediaRecorder.onstop = async () => {
                try {
                    const audioBlob = new Blob(audioChunks, {type: mediaRecorder.mimeType});
                    audioUrl = URL.createObjectURL(audioBlob);
                    audioChunks = [];

                    const formData = new FormData();
                    formData.append("file", audioBlob, "recording.webm");

                    const response = await fetch(`http://${PUBLIC_API_IP}:${PUBLIC_API_PORT}/upload-audio`, {
                        credentials: "include",
                        method: "POST",
                        body: formData
                    });
                    const data = await response.json();
                    console.log(data);
                    isChatter = data.is_chatter;
                    durationSecondsLeft = data.duration_seconds_left;
                    underThreshold = data.studyable;

                } catch (e) {
                    recordErrorMessage = e.message;
                } finally {
                    isAnalyzing = false;
                }

                stream.getTracks().forEach(track => track.stop());
            }
            audioUrl = null;
            isChatter = null;
            durationSecondsLeft = null;
            isAnalyzing = false;
            timerStart = Date.now();
            mediaRecorder.start();
            isRecording = true;
        } catch (err) {
            recordErrorMessage = `Error recording: ${err.message}`;
        }
    }

    async function stopRecording() {
        recordErrorMessage = '';
        if (mediaRecorder && isRecording) {
            if ((Date.now()-timerStart)/1000 >= 5) {
                isRecording = false;
                isAnalyzing = true;
                mediaRecorder.stop();
            } else {
                recordErrorMessage = "Recording should be at least 5 seconds";
            }
        } 
    }

    async function adjustThreshold() {
        try {
            adjustMessage = ""
            const response = await fetch(`http://${PUBLIC_API_IP}:${PUBLIC_API_PORT}/adjust-threshold?duration_prediction=${durationSecondsLeft}`, {
                credentials: "include",
                method: "POST"
            })
            const data = await response.json()
            adjustMessage = `${data.message}. New threshold: ${data.new_threshold}`
        } catch (err) {
            adjustMessage = `Error adjusting threshold: ${err.message}`
        }
    }
</script>

<div class="flex flex-col justify-center content-center min-h-screen w-screen {bgColor} font-base">
    <header class="flex items-center rounded-b-3xl shadow-md shadow-blue-300 pl-5 h-25 bg-white">
        <img src={logo} alt="HushStudy Logo" class="h-3/4">
        <h1 class="flex-1 text-left text-5xl text-blue-400">HushStudy</h1>
    </header>
    <div class="text-white mt-10 pl-10 pr-10 h-46">
        <h2 class="text-5xl text-center">{chatterMessage}</h2>
        <h3 class="mt-20 text-xl text-center">{chatterSubMessage}</h3>
    </div>
    <div class="flex-1 flex flex-col items-center mt-25 text-white">
        <button class="rounded-2xl outline-2 outline-offset-1 outline-white border-2 bg-white active:scale-95 shadow-2xl pr-4 pl-4 p-2 text-blue-400 h-fit w-fit" onclick={isRecording? stopRecording : startRecording} >{ isRecording ? "Stop" : "Start" } Recording</button>
        {#if recordErrorMessage}
            <h5 class="m-3 pl-2 pr-2 rounded-lg text-sm font-sans shadow-md bg-red-500">{recordErrorMessage}</h5>
        {/if}
        {#if isChatter}
        <button class="mt-3 rounded-2xl outline-2 outline-offset-1 outline-white border-2 bg-white active:scale-95 shadow-2xl pr-4 pl-4 p-2 text-blue-400 h-fit w-fit" onclick={adjustThreshold}>Adjust Threshold</button>
            {#if adjustMessage}
                <h5 class="m-3 pl-2 pr-2 rounded-lg text-white text-sm font-sans shadow-md bg-blue-400">{adjustMessage}</h5>
            {/if}
        {/if}
        {#if audioUrl}     
            <figure class="flex flex-col items-center gap-4 mt-8">
                <figcaption>Recorded Audio:</figcaption>
                <audio controls src="{audioUrl}"></audio>
            </figure>
        {/if}
    </div>
</div>