<template>
    <div>
        <vue-p5 @setup="setup" @draw="draw"/>
    </div>
</template>

<script>
    import VueP5 from 'vue-p5'

    export default {
        name: "FFT",
        components: {
            VueP5
        },
        props: {
            analyser: {
                required: true
            }
        },
        data: () => ({
            canvasHeight: 300,
            canvasWidth: 1000,
            fftSize: 4410,
            zeroPadding: 1,
            outputSize: 4410,
            sampleOutCount: 256,
            lowerHz: 20,
            higherHz: 20000
        }),
        computed: {
            audioCtx: function () {
                return this.analyser.context
            }
        },
        methods: {
            setup(sketch) {
                sketch.createCanvas(this.canvasWidth, this.canvasHeight)
            },
            draw(sketch) {
                sketch.background(0, 0, 0);
                sketch.noFill();
                sketch.beginShape();
                sketch.stroke(255);

                let waveform = new Float32Array(this.analyser._analyser.fftSize);
                this.analyser._analyser.getFloatTimeDomainData(waveform);

                let audioBuffer = new Array(this.outputSize).fill(0);
                let norm = 0;
                for (let i = 0; i < this.fftSize; i++) {
                    // Apply window function
                    let x = i * 2 / (this.fftSize-1) - 1;
                    let w = Math.cos(x*Math.PI/2) ** 2;
                    audioBuffer[i] = waveform[i+(waveform.length-this.fftSize)] * w;
                    norm += w;
                }
                audioBuffer = audioBuffer.map((x, _, y) => x * (y.length/norm));
                let spectrum = this.calcFFT(audioBuffer);

                this.calcSpectrum(spectrum).map((amp, i, arr) => {
                    sketch.line(this.canvasWidth / arr.length / 2 + i * this.canvasWidth / arr.length, this.map(amp, 0, 0.5, this.canvasHeight/2, 0),
                        this.canvasWidth / arr.length / 2 + i * this.canvasWidth / arr.length, this.map(amp, 0, 0.5, this.canvasHeight/2, this.canvasHeight));
                });

                sketch.endShape();
            },

            // p5.js map function
            map(x, min, max, targetMin, targetMax) {
                return (x - min) / (max - min) * (targetMax - targetMin) + targetMin;
            },
            clamp(x, min, max) {
                return Math.min(Math.max(x, min), max);
            },
            // Linear interpolation (used for FFT bin interpolation)
            lerp(x, y, z) {
                return x*(1-z)+y*z;
            },
            fscale(x, freqScale = 'logarithmic') {
                switch(freqScale.toLowerCase()) {
                    default:
                        return x;
                    case 'log':
                    case 'logarithmic':
                        return Math.log2(x);
                    case 'mel':
                        return Math.log2(1+x/700);
                    case 'critical bands':
                    case 'bark':
                        return (26.81*x)/(1960+x)-0.53;
                    case 'equivalent rectangular bandwidth':
                    case 'erb':
                        return Math.log2(1+0.00437*x);
                }
            },
            invFscale(x, freqScale = 'logarithmic') {
                switch(freqScale.toLowerCase()) {
                    default:
                        return x;
                    case 'log':
                    case 'logarithmic':
                        return 2 ** x;
                    case 'mel':
                        return 700 * ((2 ** x) - 1);
                    case 'critical bands':
                    case 'bark':
                        return 1960 / (26.81/(x+0.53)-1);
                    case 'equivalent rectangular bandwidth':
                    case 'erb':
                        return (1/0.00437) * ((2 ** x) - 1);
                }
            },
            // Hz and FFT bin conversion
            hertzToFFTBin(x, y = 'round', fftSize = this.outputSize, sampleRate = this.audioCtx.sampleRate) {
                const bin = x * fftSize / sampleRate;
                let func = y;

                if (!['floor','ceil','trunc'].includes(func))
                    func = 'round'; // always use round if you specify an invalid/undefined value

                return Math[func](bin);
            },
            // Calculate the FFT
            calcFFT(input) {
                let fft = input.map(x => x);
                let fft2 = input.map(x => x);
                transform(fft, fft2);
                let output = new Array(Math.round(fft.length/2)).fill(0);
                for (let i = 0; i < output.length; i++) {
                    output[i] = Math.hypot(fft[i], fft2[i])/(fft.length);
                }
                return output;
            },
            // Frequency bands generator
            generateFreqBands(N = this.sampleOutCount, low = this.lowerHz, high = this.higherHz, freqScale, freqSkew) {
                let freqArray = [];
                for (let i = 0; i < N; i++) {
                    freqArray.push({
                        lo: this.invFscale( this.map(i-0.5, 0, N-1, this.fscale(low, freqScale, freqSkew), this.fscale(high, freqScale, freqSkew)), freqScale, freqSkew),
                        ctr: this.invFscale( this.map(i, 0, N-1, this.fscale(low, freqScale, freqSkew), this.fscale(high, freqScale, freqSkew)), freqScale, freqSkew),
                        hi: this.invFscale( this.map(i+0.5, 0, N-1, this.fscale(low, freqScale, freqSkew), this.fscale(high, freqScale, freqSkew)), freqScale, freqSkew)
                    });
                }
                return freqArray;
            },
            // Calculate frequency bands power spectrum from FFT
            calcSpectrum(fftCoeffs, mode = 'interpolate', summingMode = 'rms average', hzArray = this.generateFreqBands()) {
                let boundArr = [];
                let nBars = 0;
                for (let i = 0; i < hzArray.length; i++) {
                    let minIdx = this.hertzToFFTBin(hzArray[i].lo, mode === 'bandpower' ? 'round' : 'ceil');
                    let maxIdx = this.hertzToFFTBin(hzArray[i].hi, mode === 'bandpower' ? 'round' : 'floor');
                    let minIdx2 = this.hertzToFFTBin(hzArray[i].lo, 'floor');
                    if (mode === 'interpolate') {
                        if (minIdx > maxIdx)
                            nBars++;
                        else {
                            if (nBars > 1) {
                                for (let j = 1; j <= nBars; j++)
                                    boundArr[boundArr.length - j].factor = (nBars - j) / nBars;
                            }
                            nBars = 1;
                        }
                    }
                    boundArr.push({
                        dataIdx: minIdx <= maxIdx || mode !== 'interpolate' ? minIdx : minIdx2,
                        endIdx: maxIdx,
                        factor: 0,
                        interpolate: minIdx > maxIdx && mode === 'interpolate'
                    });
                }
                return boundArr.map((x) => {
                    let amp = 0;
                    if (x.interpolate) {
                        let xFloat = this.clamp(x.dataIdx+x.factor, 0, fftCoeffs.length-1);
                        let xInt = Math.trunc(xFloat);
                        if (xFloat - xInt <= 0)
                            amp = fftCoeffs[x.dataIdx];
                        else
                            amp = this.lerp(fftCoeffs[this.clamp(x.dataIdx, 0, fftCoeffs.length-1)], fftCoeffs[this.clamp(x.dataIdx+1, 0, fftCoeffs.length-1)], xFloat - xInt);
                    }
                    else {
                        let low = this.clamp(x.dataIdx, 0, fftCoeffs.length-1);
                        let high = this.clamp(x.endIdx, 0, fftCoeffs.length-1);
                        for (let i = low; i <= high; i++) {
                            if (summingMode === 'max')
                                amp = Math.max(amp, fftCoeffs[i]);
                            else
                                amp += fftCoeffs[i] ** ((summingMode === 'rms average' || summingMode === 'rms sum') ? 2 : 1);
                        }
                        amp = (amp/((summingMode === 'rms sum' || summingMode === 'sum' || summingMode === 'max') ? 1 : (x.endIdx - x.dataIdx + 1))) ** ((summingMode === 'rms average' || summingMode === 'rms sum') ? 0.5 : 1);
                    }
                    return amp;
                });
            }
        }
    }

    /*
 * Free FFT and convolution (JavaScript)
 *
 * Copyright (c) 2017 Project Nayuki. (MIT License)
 * https://www.nayuki.io/page/free-small-fft-in-multiple-languages
 *
 * Permission is hereby granted, free of charge, to any person obtaining a copy of
 * this software and associated documentation files (the "Software"), to deal in
 * the Software without restriction, including without limitation the rights to
 * use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
 * the Software, and to permit persons to whom the Software is furnished to do so,
 * subject to the following conditions:
 * - The above copyright notice and this permission notice shall be included in
 *   all copies or substantial portions of the Software.
 * - The Software is provided "as is", without warranty of any kind, express or
 *   implied, including but not limited to the warranties of merchantability,
 *   fitness for a particular purpose and noninfringement. In no event shall the
 *   authors or copyright holders be liable for any claim, damages or other
 *   liability, whether in an action of contract, tort or otherwise, arising from,
 *   out of or in connection with the Software or the use or other dealings in the
 *   Software.
 */


    /*
     * Computes the discrete Fourier transform (DFT) of the given complex vector, storing the result back into the vector.
     * The vector can have any length. This is a wrapper function.
     */
    function transform(real, imag) {
        const n = real.length;
        if (n != imag.length)
            throw "Mismatched lengths";
        if (n <= 0)
            return;
        else if ((2 ** Math.trunc(Math.log2(n))) === n)  // Is power of 2
            transformRadix2(real, imag);
        else  // More complicated algorithm for arbitrary sizes
            transformBluestein(real, imag);
    }


    /*
     * Computes the inverse discrete Fourier transform (IDFT) of the given complex vector, storing the result back into the vector.
     * The vector can have any length. This is a wrapper function. This transform does not perform scaling, so the inverse is not a true inverse.
     */
    function inverseTransform(real, imag) {
        transform(imag, real);
    }


    /*
     * Computes the discrete Fourier transform (DFT) of the given complex vector, storing the result back into the vector.
     * The vector's length must be a power of 2. Uses the Cooley-Tukey decimation-in-time radix-2 algorithm.
     */
    function transformRadix2(real, imag) {
        // Length variables
        const n = real.length;
        if (n != imag.length)
            throw "Mismatched lengths";
        if (n <= 1)  // Trivial transform
            return;
        const logN = Math.log2(n);
        if ((2 ** Math.trunc(logN)) !== n)
            throw "Length is not a power of 2";

        // Trigonometric tables
        let cosTable = new Array(n / 2);
        let sinTable = new Array(n / 2);
        for (let i = 0; i < n / 2; i++) {
            cosTable[i] = Math.cos(2 * Math.PI * i / n);
            sinTable[i] = Math.sin(2 * Math.PI * i / n);
        }

        // Bit-reversed addressing permutation
        for (let i = 0; i < n; i++) {
            let j = reverseBits(i, logN);
            if (j > i) {
                let temp = real[i];
                real[i] = real[j];
                real[j] = temp;
                temp = imag[i];
                imag[i] = imag[j];
                imag[j] = temp;
            }
        }

        // Cooley-Tukey decimation-in-time radix-2 FFT
        for (let size = 2; size <= n; size *= 2) {
            let halfsize = size / 2;
            let tablestep = n / size;
            for (let i = 0; i < n; i += size) {
                for (let j = i, k = 0; j < i + halfsize; j++, k += tablestep) {
                    const l = j + halfsize;
                    const tpre =  real[l] * cosTable[k] + imag[l] * sinTable[k];
                    const tpim = -real[l] * sinTable[k] + imag[l] * cosTable[k];
                    real[l] = real[j] - tpre;
                    imag[l] = imag[j] - tpim;
                    real[j] += tpre;
                    imag[j] += tpim;
                }
            }
        }

        // Returns the integer whose value is the reverse of the lowest 'bits' bits of the integer 'x'.
        function reverseBits(x, bits) {
            let y = 0;
            for (let i = 0; i < bits; i++) {
                y = (y << 1) | (x & 1);
                x >>>= 1;
            }
            return y;
        }
    }


    /*
     * Computes the discrete Fourier transform (DFT) of the given complex vector, storing the result back into the vector.
     * The vector can have any length. This requires the convolution function, which in turn requires the radix-2 FFT function.
     * Uses Bluestein's chirp z-transform algorithm.
     */
    function transformBluestein(real, imag) {
        // Find a power-of-2 convolution length m such that m >= n * 2 + 1
        const n = real.length;
        if (n != imag.length)
            throw "Mismatched lengths";
        const m = 2 ** Math.trunc(Math.log2(n*2)+1);

        // Trignometric tables
        let cosTable = new Array(n);
        let sinTable = new Array(n);
        for (let i = 0; i < n; i++) {
            let j = i * i % (n * 2);  // This is more accurate than j = i * i
            cosTable[i] = Math.cos(Math.PI * j / n);
            sinTable[i] = Math.sin(Math.PI * j / n);
        }

        // Temporary vectors and preprocessing
        let areal = newArrayOfZeros(m);
        let aimag = newArrayOfZeros(m);
        for (let i = 0; i < n; i++) {
            areal[i] =  real[i] * cosTable[i] + imag[i] * sinTable[i];
            aimag[i] = -real[i] * sinTable[i] + imag[i] * cosTable[i];
        }
        let breal = newArrayOfZeros(m);
        let bimag = newArrayOfZeros(m);
        breal[0] = cosTable[0];
        bimag[0] = sinTable[0];
        for (let i = 1; i < n; i++) {
            breal[i] = breal[m - i] = cosTable[i];
            bimag[i] = bimag[m - i] = sinTable[i];
        }

        // Convolution
        let creal = new Array(m);
        let cimag = new Array(m);
        convolveComplex(areal, aimag, breal, bimag, creal, cimag);

        // Postprocessing
        for (let i = 0; i < n; i++) {
            real[i] =  creal[i] * cosTable[i] + cimag[i] * sinTable[i];
            imag[i] = -creal[i] * sinTable[i] + cimag[i] * cosTable[i];
        }
    }


    /*
     * Computes the circular convolution of the given real vectors. Each vector's length must be the same.
     */
    /*function convolveReal(x, y, out) {
        const n = x.length;
        if (n != y.length || n != out.length)
            throw "Mismatched lengths";
        convolveComplex(x, newArrayOfZeros(n), y, newArrayOfZeros(n), out, newArrayOfZeros(n));
    }*/


    /*
     * Computes the circular convolution of the given complex vectors. Each vector's length must be the same.
     */
    function convolveComplex(xreal, ximag, yreal, yimag, outreal, outimag) {
        const n = xreal.length;
        if (n != ximag.length || n != yreal.length || n != yimag.length
            || n != outreal.length || n != outimag.length)
            throw "Mismatched lengths";

        xreal = xreal.slice();
        ximag = ximag.slice();
        yreal = yreal.slice();
        yimag = yimag.slice();
        transform(xreal, ximag);
        transform(yreal, yimag);

        for (let i = 0; i < n; i++) {
            const temp = xreal[i] * yreal[i] - ximag[i] * yimag[i];
            ximag[i] = ximag[i] * yreal[i] + xreal[i] * yimag[i];
            xreal[i] = temp;
        }
        inverseTransform(xreal, ximag);

        for (let i = 0; i < n; i++) {  // Scaling (because this FFT implementation omits it)
            outreal[i] = xreal[i] / n;
            outimag[i] = ximag[i] / n;
        }
    }


    function newArrayOfZeros(n) {
        let result = new Array(n).fill(0);
        return result;
    }
</script>

<style scoped>

</style>
