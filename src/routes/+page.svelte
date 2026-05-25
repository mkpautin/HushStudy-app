<script>
	import { form } from "$app/server";
    import { PUBLIC_API_IP, PUBLIC_API_PORT } from '$env/static/public';
    import logo from "$lib/assets/logo.svg";
    import infoIcon from "$lib/assets/blue-information-button-18667.svg";
	import { normalizeUrl } from "@sveltejs/kit";

    let timerStart;
    let elapsedSeconds = $state(0);
    let elapsedIntervalId = null;
    let isAboutOpen = $state(false);

    let isRecording = $state(false);
    let mediaRecorder = null;
    let audioChunks = [];
    let audioUrl = $state(null);
    let isChatter = $state(null);
    let durationSecondsLeft = $state(null);
    let durationMinsLeft = $derived(Math.ceil(durationSecondsLeft / 60))
    let underThreshold = $state(false);
    let isAnalyzing = $state(false);
    let elapsedDisplay = $derived(formatElapsed(elapsedSeconds));
    let { chatterMessage, chatterSubMessage, bgColor} = $derived.by(() => {
        if (isRecording) {
            return {
                chatterMessage: "Recording...",
                chatterSubMessage: `Click the button to stop recording. Elapsed: ${elapsedDisplay}`,
                bgColor: "bg-blue-400"
            };
        } else if (isAnalyzing) {
            return {
                chatterMessage: "Processing...",
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

    function formatElapsed(seconds) {
        const mins = Math.floor(seconds / 60);
        const secs = seconds % 60;
        return `${String(mins).padStart(2, "0")}:${String(secs).padStart(2, "0")}`;
    }

    function startElapsedTimer() {
        stopElapsedTimer();
        elapsedSeconds = 0;
        elapsedIntervalId = setInterval(() => {
            elapsedSeconds += 1;
        }, 1000);
    }

    function stopElapsedTimer() {
        if (elapsedIntervalId) {
            clearInterval(elapsedIntervalId);
            elapsedIntervalId = null;
        }
    }

    function openAbout() {
        isAboutOpen = true;
    }

    function closeAbout() {
        isAboutOpen = false;
    }

    function handleKeydown(event) {
        if (isAboutOpen && event.key === "Escape") {
            closeAbout();
        }
    }

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
                    stopElapsedTimer();
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
            startElapsedTimer();
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
                stopElapsedTimer();
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

<svelte:window on:keydown={handleKeydown} />

<div class="flex flex-col justify-center content-center min-h-screen w-screen {bgColor} font-base">
    <header class="flex items-center rounded-b-3xl shadow-md shadow-blue-300 pl-5 h-25 bg-white">
        <img src={logo} alt="HushStudy Logo" class="h-3/4">
        <h1 class="flex-1 text-left text-5xl text-blue-400">HushStudy</h1>
        <button class="mr-5 flex h-10 w-10 items-center justify-center rounded-full border-2 border-blue-400 bg-white text-blue-400 shadow-md active:scale-95 sm:h-auto sm:w-auto sm:rounded-xl sm:px-4 sm:py-2" aria-label="About" onclick={openAbout}>
            <img class="sm:hidden h-6 w-6" src={infoIcon} alt="" aria-hidden="true">
            <span class="hidden sm:inline">About</span>
        </button>
    </header>
    {#if isAboutOpen}
        <div class="fixed inset-0 z-50 flex items-center justify-center bg-black/60" onclick={closeAbout}>
            <div class="w-11/12 max-w-xl rounded-2xl bg-white p-6 text-blue-900 shadow-xl" onclick={(event) => event.stopPropagation}>
                <div class="flex items-start justify-between gap-4">
                    <h2 class="text-3xl text-blue-400">About HushStudy</h2>
                    <button class="rounded-xl border-2 border-blue-400 bg-white px-3 py-1 text-blue-400 shadow-md active:scale-95" onclick={closeAbout}>Close</button>
                </div>
                <div class="mt-4">
                    <h3 class="text-xl text-blue-500">About</h3>
                    <p class="mt-2 text-base">HushStudy helps you evaluate whether a space is quiet enough to study by analyzing short audio samples.</p>
                </div>
                <div class="mt-4">
                    <h3 class="text-xl text-blue-500">Instructions</h3>
                    <p class="mt-2 text-base">Press Start Recording, wait a few seconds, then press Stop to analyze the audio and see results.</p>
                </div>
            </div>
        </div>
    {/if}
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