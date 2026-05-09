<script>
	import { form } from "$app/server";
    import { PUBLIC_API_IP, PUBLIC_API_PORT } from '$env/static/public';
    let isRecording = $state(false);
    let mediaRecorder = null;
    let audioChunks = [];
    let audioUrl = $state(null);

    async function startRecording() {
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
                const audioBlob = new Blob(audioChunks, {type: mediaRecorder.mimeType});
                audioUrl = URL.createObjectURL(audioBlob);
                audioChunks = [];

                const formData = new FormData();
                formData.append("file", audioBlob, "recording.webm");

                await fetch(`http://${PUBLIC_API_IP}:${PUBLIC_API_PORT}/upload-audio`, {
                    method: "POST",
                    body: formData
                });

                stream.getTracks().forEach(track => track.stop());
            }

            mediaRecorder.start();
            isRecording = true;
        } catch (err) {
            console.error("Error recording: ", err);
        }
    }

    function stopRecording() {
        if (mediaRecorder && isRecording) {
            mediaRecorder.stop();
            isRecording = false;
        }
    }
</script>
<figure class="flex flex-col items-center gap-4 mb-8">
    <figcaption>Recorded Audio:</figcaption>
    <audio controls src="{audioUrl}"></audio>
</figure>
<div id="buttons" class="flex justify-center gap-2">
    {#if !isRecording}
        <button class="rounded-md border border-gray-300 shadow-lg p-1 h-fit" onclick={startRecording} >Start Recording</button>
    {:else}
        <button class="rounded-md border border-gray-300 shadow-lg p-1 h-fit" onclick={stopRecording} >Stop Recording</button>
    {/if}
</div>