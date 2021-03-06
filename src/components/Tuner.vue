<template>
    <div class="effectContainer">
        <img class="buttonIcon close" src="/static/wat/x_white.svg" @click="close" />
        <h1 ref="title" class="title">Tuner</h1>
        <span ref="toggleButton" class="toggleButton" @click="toggle">
            <img class="buttonIcon" src="/static/wat/power.svg" />
        </span>
        <div ref="tolerance"></div>
        <canvas ref="canvas" :width="width" :height="height"></canvas>
        <div ref="sensitivity"></div>
    </div>
</template>

<script>
import {mapGetters} from 'vuex'
import {PitchDetector} from '../pitchDetector'
import {utils} from '../utils'
import {knob} from '../knob'

const effectName = 'Tuner'

export default {
    name: effectName,
    methods: {
        ...mapGetters(['stream']),
        toggle() {
            if (this.isActive) {
                this.$refs.toggleButton.classList.remove('activeButton')
                this.$refs.title.classList.remove('active')
            } else {
                this.$refs.toggleButton.classList.add('activeButton')
                this.$refs.title.classList.add('active')

                if (this.$store.getters.stream) {
                    this.node.setStream(this.$store.getters.stream)
                    window.requestAnimationFrame(this.draw)
                } else {
                    console.error('Tuner error: No mediaStreamSource set')
                }
            }
            this.isActive = !this.isActive

            Object.values(this.knobs).forEach((knob) => {
                knob.setActive(this.isActive)
            })
        },
        close() {
            this.$emit('closeEffect', effectName)
        },
        draw() {
            this.ctx.clearRect(0, 0, this.width, this.height)

            if (this.isActive) {
                const pitchData = this.node.getPitchData()
                // console.log(pitchData)
                this.recentPitchData.push(pitchData)

                this.ctx.strokeStyle = '#444'
                this.drawArc(250, 268)
                this.drawArc(272, 290)

                if (this.recentPitchData.length > this.tunSensitivity) {
                    this.recentPitchData.shift()

                    const recentNotes = []
                    this.recentPitchData.forEach((pd) => {
                        recentNotes.push(pd.note)
                    })

                    const dominantNote = recentNotes
                        .sort(
                            (a, b) =>
                                recentNotes.filter((v) => v === a).length - recentNotes.filter((v) => v === b).length
                        )
                        .pop()

                    if (dominantNote !== '') {
                        let detuneSum = 0
                        let relaventNoteCount = 0
                        this.recentPitchData.forEach((pd) => {
                            if (pd.note === dominantNote) {
                                detuneSum += pd.detune
                                relaventNoteCount++
                            }
                        })
                        const averageDetune = detuneSum / relaventNoteCount

                        this.ctx.strokeStyle = '#ff9c33'
                        this.ctx.fillStyle = '#ff9c33'
                        if (Math.abs(averageDetune) < 2) {
                            this.ctx.strokeStyle = '#00ff5e'
                            this.ctx.fillStyle = '#00ff5e'
                        }

                        const baseNote = dominantNote[0]
                        const isSharp = dominantNote[1] === '#'

                        this.ctx.font = '60px Graphik Regular'
                        this.ctx.fillText(baseNote, 200, 85)

                        if (isSharp) {
                            this.ctx.font = '30px Graphik Regular'
                            this.ctx.fillText('#', 232, 62)
                        }

                        const angle = utils.map(averageDetune, -50, 50, 250, 290)
                        this.drawArc(angle - 1, angle + 1)

                        this.ctx.font = '30px Graphik Regular'
                        if (Math.abs(averageDetune) < 2) {
                            this.ctx.fillText('♭', 20, 38)
                            this.ctx.fillText('♯', 380, 34)
                            this.drawFlatTriangle()
                            this.drawSharpTriangle()
                        } else if (averageDetune < 0) {
                            this.ctx.fillText('♭', 20, 38)
                            this.drawFlatTriangle()
                            this.ctx.fillStyle = '#444'
                            this.ctx.fillText('♯', 380, 34)
                            this.drawSharpTriangle()
                        } else {
                            this.ctx.fillText('♯', 380, 34)
                            this.drawSharpTriangle()
                            this.ctx.fillStyle = '#444'
                            this.ctx.fillText('♭', 20, 38)
                            this.drawFlatTriangle()
                        }
                    } else {
                        this.drawOffState()
                    }
                } else {
                    this.drawOffState()
                }

                if (this.isActive) {
                    window.requestAnimationFrame(this.draw)
                }
            } else {
                this.drawOffState()
            }
        },
        drawArc(start, stop) {
            this.ctx.beginPath()
            this.ctx.arc(200, 520, 500, (start * Math.PI) / 180, (stop * Math.PI) / 180)
            this.ctx.stroke()
        },
        drawFlatTriangle() {
            this.ctx.beginPath()
            this.ctx.moveTo(100, 60)
            this.ctx.lineTo(100, 80)
            this.ctx.lineTo(130, 70)
            this.ctx.fill()
        },
        drawSharpTriangle() {
            this.ctx.beginPath()
            this.ctx.moveTo(300, 60)
            this.ctx.lineTo(300, 80)
            this.ctx.lineTo(270, 70)
            this.ctx.fill()
        },
        drawOffState() {
            this.ctx.fillStyle = '#444'
            this.ctx.strokeStyle = '#444'

            this.drawArc(250, 268)
            this.drawArc(272, 290)

            this.ctx.font = '30px Graphik Regular'
            this.ctx.fillText('♯', 380, 34)
            this.ctx.fillText('♭', 20, 38)

            this.ctx.font = '60px Graphik Regular'
            this.ctx.fillText('--', 200, 85)

            this.drawSharpTriangle()
            this.drawFlatTriangle()
        },
    },
    data() {
        return {
            isActive: false,
            tunWasActive: true,
            width: 400,
            height: 100,
            tunTolerance: 90,
            tunSensitivity: 25,
        }
    },
    mounted() {
        this.node = new PitchDetector()
        this.recentPitchData = []

        this.tunWasActive = localStorage.tunWasActive === 'false' ? false : true
        this.tunTolerance = localStorage.tunTolerance || this.tunTolerance
        this.tunSensitivity = localStorage.tunSensitivity || this.tunSensitivity

        this.knobs = {
            tolerance: knob.create(this.$refs.tolerance, 'Tolerance', this.tunTolerance, 50, 99, false, (_, v) => {
                this.node.tolerance = v / 100
                this.tunTolerance = v
            }),
            sensitivity: knob.create(
                this.$refs.sensitivity,
                'Sample Size',
                this.tunSensitivity,
                1,
                100,
                false,
                (_, v) => {
                    this.tunSensitivity = v
                }
            ),
        }

        this.canvas = this.$refs.canvas
        this.ctx = this.canvas.getContext('2d')
        this.canvas.style.backgroundColor = '#111'

        this.ctx.lineWidth = 8
        this.ctx.lineCap = 'round'
        this.ctx.textAlign = 'center'

        this.drawOffState()

        if (this.tunWasActive) {
            this.toggle()
        }
    },
    watch: {
        isActive(value) {
            localStorage.tunWasActive = value
        },
        tunTolerance(value) {
            localStorage.tunTolerance = value
        },
        tunSensitivity(value) {
            localStorage.tunSensitivity = value
        },
    },
    beforeDestroy() {
        if (this.isActive) {
            this.toggle()
        }
    },
}
</script>

<style scoped>
canvas {
    border-radius: 6px;
    margin: 0 8px;
}
</style>
