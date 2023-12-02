<script setup>
import { ref } from 'vue'

const imageFile = ref(null)

function chooseFile() {
    document.getElementById('inputImage').click()
}

function processImageFile(evt) {
    const img = new Image
    img.src = URL.createObjectURL(evt.target.files[0])
    img.onload = function () {
        drawOnCanvas(img)
    }
}

function drawOnCanvas(img) {
    const canvas = document.getElementById('canvas')
    canvas.width = 640
    canvas.height = canvas.width * img.height / img.width
    const ctx = canvas.getContext('2d', { willReadFrequently: true })
    ctx.drawImage(img, 0, 0, canvas.width, canvas.height)
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height).data
    const rgbValues = []

    for (let index = 0; index < imageData.length; index+= 4) {
        rgbValues.push({
            red: imageData[index],
            green: imageData[index + 1],
            blue: imageData[index + 2],
        })
    }

    const quantizedColor = quantization(rgbValues, 0)
    buildPallete(quantizedColor)
}

function quantization(rgbValues, depth) {
    const MAX_DEPTH = 4

    if (depth === MAX_DEPTH || rgbValues.length === 0) {
        const color = rgbValues.reduce((prev, cur) => {
            prev.red += cur.red
            prev.green += cur.green
            prev.blue += cur.blue

            return prev
        },{ red: 0, green: 0, blue: 0 })

        color.red = Math.round(color.red / rgbValues.length)
        color.green = Math.round(color.green / rgbValues.length)
        color.blue = Math.round(color.blue / rgbValues.length)
    
        return [color]
    }

    const sortBy = largestColorRange(rgbValues)
    rgbValues.sort((a, b) => {
        return a[sortBy] - b[sortBy]
    })
    const mid = rgbValues.length / 2

    return [
        ...quantization(rgbValues.slice(0, mid), depth + 1),
        ...quantization(rgbValues.slice(mid + 1), depth + 1),
    ]
}

function largestColorRange(rgbValues) {
    let redMin = Number.MAX_VALUE
    let blueMin = Number.MAX_VALUE
    let greenMin = Number.MAX_VALUE

    let redMax = Number.MIN_VALUE
    let blueMax = Number.MIN_VALUE
    let greenMax = Number.MIN_VALUE

    for (let index = 0; index < rgbValues.length; index++) {
        const pixel = rgbValues[index]

        redMin = Math.min(redMin, pixel.red)
        greenMin = Math.min(greenMin, pixel.green)
        blueMin = Math.min(blueMin, pixel.blue)

        redMax = Math.max(redMax, pixel.red)
        greenMax = Math.max(greenMax, pixel.green)
        blueMax = Math.max(blueMax, pixel.blue)
    }

    const redRange = redMax - redMin
    const greenRange = greenMax - greenMin
    const blueRange = blueMax - blueMin
    const largestRange = Math.max(redRange, greenRange, blueRange)

    if (largestRange === redRange) {
        return 'red'
    } else if (largestRange === greenRange) {
        return 'green'
    } else {
        return 'blue'
    }
}

function buildPallete(quantizedColor) {
    const palleteContainer = document.getElementById('palette')
    palleteContainer.innerHTML = ''
    const orderedColor = orderByLuminance(quantizedColor)

    for (let index = 0; index < orderedColor.length; index++) {
        if (index > 0 && index + 1 !== orderedColor.length) {
            if (calcColorDiff(orderedColor[index], orderedColor[index + 1]) < 120) continue
        }

        const hexColor = rgbToHex(orderedColor[index])
        const colorContainer = document.createElement('div')
        colorContainer.style.display = 'flex'
        colorContainer.style['justify-content'] = 'space-between'
        colorContainer.style['align-items'] = 'center'
        const colorText = document.createElement('p')
        colorText.innerHTML = hexColor
        colorText.addEventListener('click', function () {
            navigator.clipboard.writeText(hexColor)
            .then(() => alert('RGB Value Copied!'))
            .catch(() => alert('Error Copying RGB Value!'))
        })
        colorText.style.cursor = 'pointer'
        colorContainer.appendChild(colorText)
        const colorBox = document.createElement('div')
        colorBox.addEventListener('click', function () {
            const q = encodeURIComponent(`${hexColor}`)
            window.open('https://www.google.com/search?q=' + q, '_blank').focus()
        })
        colorBox.style.cursor = 'pointer'
        colorBox.style.width = '50%'
        colorBox.style.height = '50%'
        colorBox.style.margin = '0.5rem'
        colorBox.style.background = hexColor
        colorBox.style.padding = '1rem'
        colorContainer.appendChild(colorBox)
        palleteContainer.appendChild(colorContainer)
    }
}

function orderByLuminance(rgbValues) {
    const calcLuminance = (pixel) => {
        return 0.2126 * pixel.red + 0.7152 * pixel.green + 0.0722 * pixel.blue
    }

    return rgbValues.sort((a, b) => {
        return calcLuminance(a) - calcLuminance(b)
    })
}

function rgbToHex(pixel) {
  const toHex = (c) => {
    const hex = c.toString(16)
    return hex.length == 1 ? '0' + hex : hex
  }

  return ('#' + toHex(pixel.red) + toHex(pixel.green) + toHex(pixel.blue)).toUpperCase()
}

function calcColorDiff(color1, color2) {
    return (Math.pow(color2.red - color1.red, 2)
        + Math.pow(color2.green - color1.green, 2)
        + Math.pow(color2.blue - color1.blue, 2))
}
</script>

<template>
    <div class="input-wrapper">
        <input ref="imageFile" type="file" id="inputImage" hidden @change="processImageFile">
        <button class="button" @click="chooseFile()">Choose File</button>
        <div class="canvas-wrapper">
            <canvas id="canvas" />
        </div>
        <div class="output">
            <div id="palette" />
        </div>
    </div>
</template>

<style scoped>
.input-wrapper {
    display: flex;
    flex-direction: column;
    margin: 1rem;
}

.input-wrapper .canvas-wrapper {
    max-width: 80vw;
    max-height: 80vh;
    overflow: auto;
    text-align: center;
}

.input-wrapper .button {
    margin: 1rem;
    padding: .5rem;
    font-size: 16px;
    border-radius: 8px;
    cursor: pointer;
}

.output {
    margin: 1rem 2rem;
}
</style>