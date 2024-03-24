<template>
    <VCard tag='b' style="margin: 60px; padding: 40px" color="#fbfbfb">
        <h1 style="font-size: 25px;">5S / SOSSP Database</h1><br>
        <VTextField
            class="mb-4"
            label="Speed / Period"
            placeholder="Enter Spaceship Speed / Oscillator Period"
            v-model="speed"
            :rules="[value => /^(P?[0-9]+|(\([0-9]+,[0-9]+\)|[0-9]*)c(\/[0-9]+[od]?)?)$/.test(value)]"
        />
        <v-btn @click="read5SDatabase"> Confirm </v-btn><br><br><br><br>
        <code style="background: #f0f0f0; padding: 10px; display: block; white-space: pre-wrap; font-weight: normal">
            {{ output }}
        </code>

        <v-dialog width="500" v-model="showModal">
            <v-card>
                <v-card-title>{{ title }}</v-card-title>
                <v-card-text>{{ message }}</v-card-text>

                <v-card-actions>
                <v-spacer></v-spacer>

                <v-btn
                    text="Close"
                    @click="showModal = false"
                ></v-btn>
                </v-card-actions>
            </v-card>
        </v-dialog>
    </VCard>
    <VCard tag='b' style="margin: 60px; padding: 40px" color="#fbfbfb">
        <h1 style="font-size: 25px;">GliderDB Database</h1><br>
        <VTextField
            class="mb-4"
            label="Speed / Period"
            placeholder="Enter Spaceship Speed / Oscillator Period"
            v-model="speed"
            :rules="[value => /^(P?[0-9]+|(\([0-9]+,[ ]*[0-9]+\)|[0-9]*)c(\/[0-9]+[od]?)?)$/.test(value)]"
        />
        <VCheckbox v-model="ignoreVelocity" :label=label1 /><br><br>
        <VTextField
            class="mb-4"
            label="Rule"
            placeholder="Enter the rule to search in"
            v-model="rule"
        />
        <VCheckbox v-model="ignoreRule" :label=label2 /><br><br>
        <v-btn @click="readGliderDB"> Confirm </v-btn><br><br><br><br>

        <VList v-for="(glider, index) in gliders" :key="index" style="width:100%">
            <code
                style="background: #f0f0f0; padding: 10px; display: block;
                white-space: pre-wrap; font-weight: normal; width:100%"
            >
                {{ glider.rle }}
            </code>
        </VList>
    </VCard>
</template>

<script lang="ts" setup>
import { ref } from 'vue'

import oscillators from '/src/assets/sossp.sss.txt?raw'
import oblique from '/src/assets/oblique.sss.txt?raw'
import diagonal from '/src/assets/diagonal.sss.txt?raw'
import orthogonal from '/src/assets/orthogonal.sss.txt?raw'

import r1_moore_gliders from '/src/assets/R1-C2-NM-gliders.db.txt?raw'
import r1_moore_oscillators from '/src/assets/R1-C2-NM-oscillators.db.txt?raw'
import r1_c3_moore_gliders from '/src/assets/R1-C3-NM-gliders.db.txt?raw'
import r2_moore_gliders from '/src/assets/R2-C2-NM-gliders.db.txt?raw'
import r2_moore_oscillators from '/src/assets/R2-C2-NM-oscillators.db.txt?raw'

import r1_neumann_gliders from '/src/assets/R1-C2-NN-gliders.db.txt?raw'
import r2_neumann_gliders from '/src/assets/R2-C2-NN-gliders.db.txt?raw'

const speed = ref("")
const rule = ref("")
const output = ref("Awaiting user input...")  // For 5S
const gliders = ref(
    [  // For GliderDB
        { rle: "Awaiting user input..." }
    ]
)
const showModal = ref(false)
const title = ref("")
const message = ref("")
const label1 = ref("Ignore Velocity")
const label2 = ref("Ignore Rule")
const ignoreVelocity = ref(false)
const ignoreRule = ref(false)

const insertNewLine = function(string: string, maxLength: number): string {
    let newString = []
    for (let i = 0; i < string.length; i++) {
        newString.push(string[i])
        if (i % maxLength === 0 && i !== 0) {
            newString.push("\n")
        }
    }

    return newString.join("")
}

const isSupersetOf = function(setB: Set<object>, setA: Set<object>): boolean {
    if (typeof(setB) === 'undefined') return true
    if (typeof(setA) === 'undefined') return false

    let result = true
    setA.forEach(ele => {
        if (!(setB.has(ele))) result = false
    })

    return result
}

const parseSpeed = function(): Array<any> {
    // Error Handling
    if (!/^(P?[0-9]+|(\([0-9]+,[ ]*[0-9]+\)|[0-9]*)c(\/[0-9]+[od]?)?)$/.test(speed.value)) {
        title.value = "Error"
        message.value = `"${speed.value}" is not a valid speed`
        showModal.value = !showModal.value
        return []
    }

    let dx = 0, dy = 0, p = 1
    let tokens = speed.value.split("c")

    if (tokens.length === 1) {
        p = parseInt(tokens[0].replace("P", ""))
    } else if (tokens[1].length === 0) {
        dx = tokens[0].length > 0 ? parseInt(tokens[0]) : 1
        p = 1
    } else {
        const tokens2 = tokens[0].replace(/[()]/, "").split(",")
        if (tokens2.length === 1) dx = tokens[0].length > 0 ? parseInt(tokens[0]) : 1
        else {
            dx = parseInt(tokens2[0])
            dy = parseInt(tokens2[1])

            // Ensure dx > dy
            if (dx < dy) {
                const temp = dx
                dx = dy
                dy = temp
            }
        }

        p = parseInt(tokens[1].replace(/[/od]/, ""))
    }

    // Check for diagonal speeds
    if (speed.value[speed.value.length - 1] === 'd') dy = dx
    return [dx, dy, p]
}

const parseRule = function(rule: string): Array<any> {
    let range = 1, numStates = 2, neighbourhood = "M", birth: Set<number>, survival: Set<number>
    if (/^[Bb][0-8]*\/[Ss][0-8]*[VH]?$/.test(rule)) {
        let tokens = rule.split("/")
        birth = new Set(Array.from(tokens[0].replace(/[BbSs]/, "")).map(x => parseInt(x)))
        survival = new Set(Array.from(tokens[1].replace(/[BbSs]/, "")).map(x => parseInt(x)))

        switch (rule[rule.length - 1]) {
            case "V":
                neighbourhood = "N"
                break
            case "H":
                neighbourhood = "H"
                break
            default: neighbourhood = "M"
        }
    } else if (/^[0-8]*\/[0-8]*\/[0-9]+[VH]?$/.test(rule)) {
        let tokens = rule.split("/")
        survival = new Set(Array.from(tokens[0]).map(x => parseInt(x)))
        birth = new Set(Array.from(tokens[1]).map(x => parseInt(x)))
        numStates = parseInt(tokens[2])

        switch (rule[rule.length - 1]) {
            case "V":
                neighbourhood = "N"
                break
            case "H":
                neighbourhood = "H"
                break
            default: neighbourhood = "M"
        }
    } else if (/^R[0-9]+,C[0-9]+,S(((\d,(?=\d))|(\d-(?=\d))|\d)+)?,B(((\d,(?=\d))|(\d-(?=\d))|\d)+)?,N[ABbCGHLMNX23*+#]$/.test(rule)) {
        let tokens = rule.split(",")
        range = parseInt(tokens[0].replace("R", ""))
        numStates = parseInt(tokens[1].replace("C", ""))
        neighbourhood = tokens[tokens.length - 1].replace("N", "")

        survival = new Set()
        rule.match(/S(((\d,(?=\d))|(\d-(?=\d))|\d)+)?/)![0].replace("S", "").split(",")
            .forEach(ele => {
                if (/^\d+$/.test(ele)) {
                    survival.add(parseInt(ele))
                } else {
                    for (let i = parseInt(ele.split("-")[0]); i <= parseInt(ele.split("-")[1]); i++)
                        survival.add(i)
                }
            })

        birth = new Set()
        rule.match(/B(((\d,(?=\d))|(\d-(?=\d))|\d)+)?/)![0].replace("B", "").split(",")
            .forEach(ele => {
                if (/^\d+$/.test(ele)) {
                    birth.add(parseInt(ele))
                } else {
                    for (let i = parseInt(ele.split("-")[0]); i <= parseInt(ele.split("-")[1]); i++)
                        birth.add(i)
                }
            })
    } else {
        title.value = "Error"
        message.value = `"${rule}" is not a valid rule`
        showModal.value = !showModal.value
        return []
    }

    return [range, numStates, neighbourhood, birth, survival]
}

const read5SDatabase = function() {
    const speedList = parseSpeed()
    const dx = speedList[0], dy = speedList[1], p = speedList[2]

    let db = ""
    if (dx === 0 && dy === 0) db = oscillators
    else if (dy === 0) db = orthogonal
    else if (dx === dy) db = diagonal
    else db = oblique

    // Iterate through
    let tokens
    for (let x of db.split("\n")) {
        tokens = x.split(", ")
        if (parseInt(tokens[2]) === dx && parseInt(tokens[3]) === dy && parseInt(tokens[4]) === p) {
            output.value = `#C Population: ${tokens[0]}\nx = 0, y = 0, rule = ${tokens[1]}\n${tokens[5]}`
            return
        }
    }

    if (dx === 0 && dy === 0) output.value = `Oscillator of period ${p} not found in database!`
    else output.value = `Spaceship of speed ${speed.value} not found in database!`
}

const readGliderDB = function() {
    const parsedSpeed = parseSpeed()
    const dx = parsedSpeed[0], dy = parsedSpeed[1], p = parsedSpeed[2]

    const parsedRule = parseRule(rule.value)
    const range = parsedRule[0], numStates = parsedRule[1], neighbourhood = parsedRule[2],
        birth = parsedRule[3], survival = parsedRule[4]

    // Choose the correct database
    let database_name = ""
    let db_lookup: any = {
        "R1-C2-NM-gliders": r1_moore_gliders,
        "R1-C2-NM-oscillators": r1_moore_oscillators,
        "R1-C3-NM-gliders": r1_c3_moore_gliders,
        "R2-C2-NM-gliders": r2_moore_gliders,
        "R2-C2-NM-oscillators": r2_moore_oscillators,
        "R1-C2-NN-gliders": r1_neumann_gliders,
        "R2-C2-NN-gliders": r2_neumann_gliders
    }
    if (dx === 0 && dy === 0) database_name = `R${range}-C${numStates}-N${neighbourhood}-oscillators`
    else database_name = `R${range}-C${numStates}-N${neighbourhood}-gliders`

    let db = db_lookup[database_name]

    // Linear Search
    let tokens: Array<any>, minRule, maxRule
    let entries = db.split("\n")

    gliders.value = []
    for (let i = 0; i < entries.length; i++) {
        tokens = entries[i].split(":")

        // Pre-processing
        tokens[5] = Math.abs(parseInt(tokens[5]))
        tokens[6] = Math.abs(parseInt(tokens[6]))
        if (tokens[4].includes("/")) {
            tokens[4] = parseInt(tokens[4].split("/")[0])
        } else tokens[4] = parseInt(tokens[4])

        // Checking speed is correct
        if (((tokens[4] === p && ((tokens[5] === Math.abs(dx) && tokens[6] === Math.abs(dy)) ||
            (tokens[5] === Math.abs(dy) && tokens[6] === Math.abs(dx))))) || ignoreVelocity.value) {
            minRule = parseRule(tokens[2])
            maxRule = parseRule(tokens[3])

            // Checking minimum and maximum rules
            if ((isSupersetOf(birth, minRule[3]) && isSupersetOf(survival, minRule[4]) &&
                    isSupersetOf(maxRule[3], birth) && isSupersetOf(maxRule[4], survival))
                || ignoreRule.value) {
                gliders.value.push({
                    rle: (dx === 0 && dy === 0 ? `#C P${tokens[4]}\n` :
                            `#C (${tokens[5]}, ${tokens[6]})/${tokens[4]}\n`) +
                        (tokens[0] !== "" ? `#C Name: ${tokens[0]}\n` : '') +
                        (tokens[1] !== "" ? `#C Discovered by: ${tokens[1]}\n` : '') +
                        `#C Min Rule: ${tokens[2]}\n` +
                        `#C Max Rule: ${tokens[3]}\n` +
                        `x = ${tokens[7]}, y = ${tokens[8]}, rule = ${ignoreRule.value ? tokens[2] : rule.value}\n` +
                        insertNewLine(tokens[9], 100)
                })
            }
        }
    }

    if (gliders.value.length === 0) {
        if (dx === 0 && dy === 0) gliders.value = [ { rle: `Spaceship not found in database!` } ]
        else gliders.value = [ { rle: `Spaceship not found in database!` } ]
    }
}

</script>
