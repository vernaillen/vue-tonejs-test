<template>
    <div>
        <vue-p5 @setup="setup" @draw="draw"/>
    </div>
</template>

<script>
    import VueP5 from "vue-p5";
    import "p5/lib/addons/p5.sound";

    export default {
        name: "Waveform",
        components: {
            "vue-p5": VueP5
        },
        props: {
            input: {
                required: true
            }
        },
        data: () => ({
            lines: [],
            canvasHeight: 127,
            canvasWidth: 50,
            fft: null,
            osc: null
        }),
        methods: {
            setup(sketch) {
                sketch.createCanvas(this.canvasWidth, this.canvasHeight)
                this.fft = new sketch.fft
                this.osc = new VueP5.p5.Oscillator('Sine')
            },
            draw(sketch) {
                sketch.background(240, 240, 250);

                /*let spectrum = this.fft.Analyser
                sketch.noStroke();
                sketch.fill(255, 0, 255);
                for (let i = 0; i< spectrum.length; i++){
                    let x = sketch.map(i, 0, spectrum.length, 0, this.canvasWidth);
                    let h = -sketch.height + sketch.map(spectrum[i], 0, 255, this.canvasHeight, 0);
                    sketch.rect(x, this.canvasHeight, this.canvasWidth / spectrum.length, h )
                }

                let waveform = this.fft.Waveform
                sketch.noFill();
                sketch.beginShape();
                sketch.stroke(20);
                for (let i = 0; i < waveform.length; i++){
                    let x = sketch.map(i, 0, waveform.length, 0, this.canvasWidth);
                    let y = sketch.map( waveform[i], -1, 1, 0, this.canvasHeight);
                    sketch.vertex(x,y);
                }*/
                sketch.endShape();
            },
            pushLine(line) {
                let lines = this.lines;
                lines.push(line);
                this.lines = lines.slice(-100);
            }
        }
    }
</script>

<style scoped>

</style>
