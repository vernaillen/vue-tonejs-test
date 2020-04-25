<template>
    <div>
        <vue-p5 @setup="setup" @draw="draw"/>
    </div>
</template>

<script>
    import VueP5 from "vue-p5";
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
            canvasWidth: 50
        }),
        methods: {
            setup(sketch) {
                sketch.createCanvas(this.canvasWidth, this.canvasHeight);
            },
            draw(sketch) {
                sketch.background(240, 240, 250);

                // logo should be in the middle of the screen and rotated
                sketch.translate(sketch.width / 2, sketch.height / 2);
                sketch.resetMatrix();

                for (let line of this.lines) {
                    sketch.stroke(line.color);
                    sketch.line(line.pmouseX, line.pmouseY, line.mouseX, line.mouseY);
                }
                sketch.stroke(0);
                sketch.line(0, 127 - this.input, this.canvasWidth, 127 - this.input);
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
