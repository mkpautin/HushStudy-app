<script>
	// import { form } from "$app/server";
    // import { PUBLIC_API_IP, PUBLIC_API_PORT } from '$env/static/public';
    import logo from "$lib/assets/logo.svg"
	import { normalizeUrl } from "@sveltejs/kit";

    let timerStart;

    let isRecording = $state(false);
    let mediaRecorder = null;
    let audioChunks = [];
    let audioUrl = $state(null);
    let isChatter = $state(null);
    let durationSecondsLeft = $state(null)
    let bgColor = $derived.by(() => {
        if (isChatter === true) {
            return "bg-red-700"
        } else if (isChatter === false) {
            return "bg-green-500"
        } else {
            return "bg-blue-400"
        }
    })
    let { chatterMessage, chatterSubMessage} = $derived.by(() => {
        if (isRecording) {
            return {
                chatterMessage: "Recording...",
                chatterSubMessage: "Click the button to stop recording"
            } 
        } else if (isChatter === false) {
            return {
                chatterMessage: "Non-Chatter",
                chatterSubMessage: "You could study here."
            }
        } else if (isChatter === true && durationSecondsLeft) {
            return {
                chatterMessage: "Chatter",
                chatterSubMessage: `This may last for ${durationSecondsLeft} ${durationSecondsLeft <= 1 ? "min" : "mins"}.`
            }
        } else {
            return {
                chatterMessage: "Welcome",
                chatterSubMessage: "Click the button to start recording."
            }
        }
    })
    let errorMessage = $state("")

    async function startRecording() {
        errorMessage = ""
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

                    // const response = await fetch(`http://${PUBLIC_API_IP}:${PUBLIC_API_PORT}/upload-audio`, {
                    //     method: "POST",
                    //     body: formData
                    // });
                    // const data = response.json()
                    isChatter = Math.random() >= 0.5 // data.is_chatter
                    durationSecondsLeft = Math.floor(Math.random() * 5) + 1 // data.duration_seconds_left
                } catch (e) {
                    errorMessage = e.message
                }

                stream.getTracks().forEach(track => track.stop());
            }
            audioUrl = null
            isChatter = null
            durationSecondsLeft = null
            timerStart = Date.now();
            mediaRecorder.start();
            isRecording = true;
        } catch (err) {
            errorMessage = `Error recording: ${err.message}`
        }
    }

    function stopRecording() {
        errorMessage = ''
        if (mediaRecorder && isRecording) {
            if ((Date.now()-timerStart)/1000 >= 5) {
                mediaRecorder.stop();
                isRecording = false;
            } else {
                errorMessage = "Recording should be at least 5 seconds"
            }
        } 
    }
</script>

<div class="flex flex-col justify-center content-center min-h-screen w-screen {bgColor} font-base">
    <header class="flex items-center rounded-b-3xl shadow-md shadow-blue-300 pl-5 h-25 bg-white">
        <img src={logo} alt="HushStudy Logo" class="h-3/4">
        <h1 class="flex-1 text-left text-5xl text-blue-400">HushStudy</h1>
    </header>
    <div class="text-white mt-10">
        <h2 class="text-5xl text-center">{chatterMessage}</h2>
        <h3 class="mt-20 text-xl text-center">{chatterSubMessage}</h3>
    </div>
    <div class="flex-1 flex flex-col items-center mt-30 text-white">
        <button class="rounded-2xl outline-2 outline-offset-1 outline-white border-2 bg-white active:scale-95 shadow-2xl pr-4 pl-4 p-2 text-blue-400 h-fit w-fit" onclick={isRecording? stopRecording : startRecording} >{ isRecording ? "Stop" : "Start" } Recording</button>
        {#if errorMessage}
            <h5 class="m-3 pl-2 pr-2 rounded-lg text-sm font-sans shadow-md bg-red-500">{errorMessage}</h5>
        {/if}
        {#if audioUrl}            
            <figure class="flex flex-col items-center gap-4 mt-8">
                <figcaption>Recorded Audio:</figcaption>
                <audio controls src="{audioUrl}"></audio>
            </figure>
        {/if}
    </div>
</div>