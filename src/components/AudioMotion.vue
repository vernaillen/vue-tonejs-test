<template>
    <div id="audioMotionContainer"></div>
</template>

<script>
    import AudioMotionAnalyzer from 'audiomotion-analyzer';

    export default {
        name: "AudioMotion",
        props: {
            audioContext: {
                required: true
            },
            audioNode: {
                required: true
            }
        },
        computed: {
            audioMotion: function () {
                return new AudioMotionAnalyzer(
                    document.getElementById('audioMotionContainer'),
                    {
                        audioCtx: this.audioContext
                    }
                )
            }
        },
        created() {
            this.audioNode.fan(this.audioMotion.analyzer)
            console.log("audioMotion.fftSize: " + this.audioMotion.fftSize)
        }
    }
</script>

<style scoped>

</style>
