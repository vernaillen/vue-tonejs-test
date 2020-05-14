<template>
    <div>
        <vue-p5 @setup="setup" @draw="draw"/>
    </div>
</template>

<script>
    import VueP5 from 'vue-p5'

    export default {
        name: "Waveform",
        components: {
            VueP5
        },
        props: {
            wave: {
                required: true
            }
        },
        data: () => ({
            lines: [],
            canvasHeight: 300,
            canvasWidth: 1000
        }),
        methods: {
            setup(sketch) {
                sketch.createCanvas(this.canvasWidth, this.canvasHeight)
            },
            draw(sketch) {
                sketch.background(0, 0, 0);

                sketch.noFill();
                sketch.beginShape();
                sketch.stroke(255);
                for (let i = 0; i < this.wave.size; i++){
                    let x = sketch.map(i, 0, this.wave.size, 0, this.canvasWidth);
                    let y = sketch.map( this.wave.getValue()[i], -1, 1, 0, this.canvasHeight);
                    sketch.vertex(x,y);
                }
                sketch.endShape();
            },
        }
    }
</script>

<style scoped>

</style>
