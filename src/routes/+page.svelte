<script>
    let isRecording = $state(false);
    let mediaRecorder = null;
    let audioChunks = [];
    let audioUrl = $state(null);

    async function startRecording() {
        try {
            const stream = await navigator.mediaDevices.getUserMedia({audio: true});
            mediaRecorder = new MediaRecorder(stream);
            audioChunks = [];

            mediaRecorder.ondataavailable = (event) => {
                audioChunks.push(event.data);
            };

            mediaRecorder.onstop = () => {
                const audioBlob = new Blob(audioChunks, {type: 'audio/wav'});
                audioUrl = URL.createObjectURL(audioBlob);
                audioChunks = [];

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
    <button class="rounded-md border border-gray-300 shadow-lg p-1 h-fit" onclick={startRecording} disabled={isRecording}>Start Recording</button>
    <button class="rounded-md border border-gray-300 shadow-lg p-1 h-fit" onclick={stopRecording} disabled={!isRecording}>Stop Recording</button>
</div>