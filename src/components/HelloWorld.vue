<template>
  <div class="hello">
    <canvas ref="glcanvas" width="400" height="300">Ваш браузер не поддерживает элемент canvas</canvas>
    <button @click="currentScale = [1.0, aspectRatio]" class="button">По часовой</button>
    <button @click="currentScale = [-1.0, aspectRatio]" class="button">Против часовой</button>
    <!-- switch the direction of color change -->
    <button @click="uColorDirection = uColorDirection === 'ltr' ? 'rtl' : 'ltr'" class="button">Поменять цвет</button>
    <script id="vertex-shader" type="x-shader/x-vertex">
      attribute vec2 aVertexPosition;

      uniform vec2 uScalingFactor;
      uniform vec2 uRotationVector;

      void main() {
        vec2 rotatedPosition = vec2(
          aVertexPosition.x * uRotationVector.y +
                aVertexPosition.y * uRotationVector.x,
          aVertexPosition.y * uRotationVector.y -
                aVertexPosition.x * uRotationVector.x
        );

        gl_Position = vec4(rotatedPosition * uScalingFactor, 0.0, 1.0);
      }
    </script>
    <script id="fragment-shader" type="x-shader/x-fragment">
      #ifdef GL_ES
        precision highp float;
      #endif

      uniform vec4 uGlobalColor;

      void main() {
        gl_FragColor = uGlobalColor;
      }
    </script>
  </div>
</template>

<script>
var uScalingFactor = '',
    uGlobalColor = '',
    uRotationVector = '',
    aVertexPosition = '';
export default {
  data () {
    return {
      gl: null,
      glCanvas: null,
      // Aspect ratio and coordinate system
      aspectRatio: 0,
      currentRotation: [0, 1],
      currentScale: [1.0, 1.0],
      // Vertex information
      vertexArray: [],
      vertexBuffer: '',
      vertexNumComponents: 0,
      vertexCount: 0,

      uCurrentColor: [
        [0.0, 0.0, 1.0, 1.0],
        [0.0, 1.0, 0.0, 1.0],
        [1.0, 0.0, 0.0, 1.0],
      ],
      uColorDirection: 'ltr',
      uScalingFactor: '',
      uGlobalColor: '',
      uRotationVector: '',
      aVertexPosition: '',
      // Animation timing
      previousTime: 0.0,
      degreesPerSecond: 90.0,
      shaderProgram: null,
      currentAngle: null,
      rotationRate: null,
      currentColorIndex: 0
    }
  },
  mounted () {
    this.startup()
  },
  watch: {
    uScalingFactor (to) {
      uScalingFactor = to
    },
    uGlobalColor (to) {
      uGlobalColor = to
    },
    uRotationVector (to) {
      uRotationVector = to
    },
    aVertexPosition (to) {
      aVertexPosition = to
    }
  },
  methods: {
    startup () {
      this.glCanvas = this.$refs.glcanvas
      this.gl = this.glCanvas.getContext('webgl')

      const shaderSet = [
        {
          type: this.gl.VERTEX_SHADER,
          id: 'vertex-shader',
          code: 'attribute vec2 aVertexPosition;uniform vec2 uScalingFactor;uniform vec2 uRotationVector;void main() {vec2 rotatedPosition = vec2(aVertexPosition.x * uRotationVector.y + aVertexPosition.y * uRotationVector.x, aVertexPosition.y * uRotationVector.y - aVertexPosition.x * uRotationVector.x);gl_Position = vec4(rotatedPosition * uScalingFactor, 0.0, 1.0);}'
        },
        {
          type: this.gl.FRAGMENT_SHADER,
          id: 'fragment-shader',
          code: 'precision highp float;uniform vec4 uGlobalColor;void main() {gl_FragColor = uGlobalColor;}'
        }
      ]

      this.shaderProgram = this.buildShaderProgram(shaderSet)

      this.aspectRatio = this.glCanvas.width/this.glCanvas.height
      this.currentRotation = [1, 1]
      // direction of rotation
      this.currentScale = [-1.0, this.aspectRatio] 

      this.vertexArray = new Float32Array([
        -0.5, -0.5, 0.0,
        0.0, 0.65, 0.0,
        0.5, -0.5, 0.0
      ])
      

      this.vertexBuffer = this.gl.createBuffer()
      this.gl.bindBuffer(this.gl.ARRAY_BUFFER, this.vertexBuffer)
      this.gl.bufferData(this.gl.ARRAY_BUFFER, this.vertexArray, this.gl.STATIC_DRAW)

      this.vertexNumComponents = 3
      this.vertexCount = this.vertexArray.length/this.vertexNumComponents

      this.currentAngle = 0.0
      this.rotationRate = 6

      this.animateScene()
      this.switchColor()
    },

    buildShaderProgram (shaderInfo) {
      let program = this.gl.createProgram()

      shaderInfo.forEach(desc => {
        let shader = this.compileShader(desc.id, desc.type, desc.code)

        if (shader) {
          this.gl.attachShader(program, shader)
        }
      })

      this.gl.linkProgram(program)

      if (!this.gl.getProgramParameter(program, this.gl.LINK_STATUS)) {
        console.log('Error linking shader program:')
        console.log(this.gl.getProgramInfoLog(program))
      }

      return program
    },

    compileShader(id, type, code) {

      let shader = this.gl.createShader(type)

      this.gl.shaderSource(shader, code)
      this.gl.compileShader(shader)

      if (!this.gl.getShaderParameter(shader, this.gl.COMPILE_STATUS)) {
        console.log(`Error compiling ${type === this.gl.VERTEX_SHADER ? 'vertex' : 'fragment'} shader:`)
        console.log(this.gl.getShaderInfoLog(shader))
      }
      return shader
    },

    animateScene() {
      this.gl.viewport(0, 0, this.glCanvas.width, this.glCanvas.height)
      this.gl.clearColor(0.8, 0.9, 0.9, 1.0)
      this.gl.clear(this.gl.COLOR_BUFFER_BIT)

      let radians = this.currentAngle * Math.PI / 180.0
      this.currentRotation[0] = Math.sin(radians)
      this.currentRotation[1] = Math.cos(radians)

      this.gl.useProgram(this.shaderProgram)

      this.uScalingFactor = this.gl.getUniformLocation(this.shaderProgram, 'uScalingFactor')
      this.uGlobalColor = this.gl.getUniformLocation(this.shaderProgram, 'uGlobalColor')
      this.uRotationVector = this.gl.getUniformLocation(this.shaderProgram, 'uRotationVector')

      this.gl.uniform2fv(this.uScalingFactor, this.currentScale)
      this.gl.uniform2fv(this.uRotationVector, this.currentRotation)
      // the color is set from the uCurrentColor array at the currentColorIndex
      this.gl.uniform4fv(this.uGlobalColor, this.uCurrentColor[this.currentColorIndex])

      this.gl.bindBuffer(this.gl.ARRAY_BUFFER, this.vertexBuffer)

      this.aVertexPosition =
          this.gl.getAttribLocation(this.shaderProgram, 'aVertexPosition')

      this.gl.enableVertexAttribArray(this.aVertexPosition)
      this.gl.vertexAttribPointer(this.aVertexPosition, this.vertexNumComponents, this.gl.FLOAT, false, 0, 0)

      this.gl.drawArrays(this.gl.TRIANGLES, 0, this.vertexCount)
      
      window.requestAnimationFrame(currentTime => {
        let deltaAngle = ((currentTime - this.previousTime) / 1000.0) * this.degreesPerSecond
        this.currentAngle = (this.currentAngle + deltaAngle) % 360

        this.previousTime = currentTime
        this.animateScene()
      })
    },

    switchColor () {
      // check if an element with an index exists in the array
      const checkUndef = (i) => this.uCurrentColor[i] !== undefined
      if (this.uColorDirection === 'ltr') {
        // if there is an element under the i + 1 intex in the array, we increase the index, otherwise we reset it to 0
        checkUndef(this.currentColorIndex + 1) ? this.currentColorIndex++ : this.currentColorIndex = 0
      } else {
        // If there is an element in the array under the i - 1 intex, we decrease the index, otherwise we set the index = the length of the array (last element)
        checkUndef(this.currentColorIndex - 1) ? this.currentColorIndex-- : this.currentColorIndex = this.uCurrentColor.length - 1
      }
      setTimeout(_ => this.switchColor(), 1000)
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>

.hello {
  margin-left: 30%;
}

.button {
  display: block;
  padding: 10px 45px;
  text-decoration: none;
  color: #ffffff;
  border: 1px solid #ffffff;
  cursor: pointer;
  outline: none;
}

</style>
