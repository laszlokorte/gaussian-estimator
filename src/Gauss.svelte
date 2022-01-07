<script>
  import { onMount, tick  } from "svelte";
  import createRegl from "regl";
 
  let className
  export {className as class};

  export let show;
  export let view;
  export let onclick = null;
  export let mean;
  export let variance;
  export let correlation;
  export let samples = [];

  let gl;
  let disableClick = false

  let clickHandler = function(evt) {
  	if(!disableClick) {
  		onclick(evt)
  	}
  }

  function vectorMultiplyMatrix([x,y,z,w], matrix) {
        return [
          // wir verstehen ein Element mit 12 Elementen als 3x4 Matrix
          // ein 3D Punkt/Vektor mal eine 3x4 Matrix ergibt wieder einen Punkt
          /*neues X:*/ x * matrix[0] + y * matrix[1] + z * matrix[2] + matrix[3] * w,
          /*neues Y:*/ x * matrix[4] + y * matrix[5] + z * matrix[6] + matrix[7] * w,
          /*neues Z:*/ x * matrix[8] + y * matrix[9] + z * matrix[10] + matrix[11] * w,
          /*neues Z:*/ x * matrix[12] + y * matrix[13] + z * matrix[14] + matrix[15] * w,
        ]
      }

      function matrixMultiplyMatrix([
        _x1,  _x2,  _x3,  _x4,
        _y1,  _y2,  _y3,  _y4,
        _z1,  _z2,  _z3,  _z4,
        _w1,  _w2,  _w3,  _w4,
      ], [
        x1,  x2,  x3,  x4,
        y1,  y2,  y3,  y4,
        z1,  z2,  z3,  z4,
        w1,  w2,  w3,  w4,
      ]) {
        return [
          x1 * _x1 + x2 * _y1 + x3 * _z1 + x4 * _w1,
          x1 * _x2 + x2 * _y2 + x3 * _z2 + x4 * _w2,
          x1 * _x3 + x2 * _y3 + x3 * _z3 + x4 * _w3,
          x1 * _x4 + x2 * _y4 + x3 * _z4 + x4 * _w4,

          y1 * _x1 + y2 * _y1 + y3 * _z1 + y4 * _w1,
          y1 * _x2 + y2 * _y2 + y3 * _z2 + y4 * _w2,
          y1 * _x3 + y2 * _y3 + y3 * _z3 + y4 * _w3,
          y1 * _x4 + y2 * _y4 + y3 * _z4 + y4 * _w4,

          z1 * _x1 + z2 * _y1 + z3 * _z1 + z4 * _w1,
          z1 * _x2 + z2 * _y2 + z3 * _z2 + z4 * _w2,
          z1 * _x3 + z2 * _y3 + z3 * _z3 + z4 * _w3,
          z1 * _x4 + z2 * _y4 + z3 * _z4 + z4 * _w4,

          w1 * _x1 + w2 * _y1 + w3 * _z1 + w4 * _w1,
          w1 * _x2 + w2 * _y2 + w3 * _z2 + w4 * _w2,
          w1 * _x3 + w2 * _y3 + w3 * _z3 + w4 * _w3,
          w1 * _x4 + w2 * _y4 + w3 * _z4 + w4 * _w4,
        ]
      }

  function deg2rad(deg) {
        return deg/180 * Math.PI
      }

      function makeMatrixPerspective(fovDeg, aspect, near, far) {
        const f = 1.0 / Math.tan(deg2rad(fovDeg) / 2)
        const nf = 1 / (near - far)

        return [
          f / aspect, 0.0, 0.0, 0.0,
          0.0, f, 0.0, 0.0,
          0.0, 0.0, (far + near) * nf, -1.0,
          0.0, 0.0, (2 * far * near) * nf, 0.0
        ]
      }

      function makeMatrixIdentity() {
        return [
          1, 0, 0, 0,
          0, 1, 0, 0,
          0, 0, 1, 0,
          0, 0, 0, 1,
        ]
      }

      function makeMatrixScale(x,y,z) {
        return [
          x, 0, 0, 0,
          0, y, 0, 0,
          0, 0, z, 0,
          0, 0, 0, 1,
        ]
      }

      function makeMatrixTranslate(x,y,z) {
        return [
          1, 0, 0, 0,
          0, 1, 0, 0,
          0, 0, 1, 0,
          x, y, z, 1,
        ]
      }

      function makeMatrixRotateZ(angle) {
        const c = Math.cos(angle)
        const s = Math.sin(angle)
        return [
          c, -s, 0, 0,
          s,  c, 0, 0,
          0,  0, 1, 0,
          0,  0, 0, 1,
        ]
      }

      function makeMatrixRotateX(angle) {
        const c = Math.cos(angle)
        const s = Math.sin(angle)
        return [
          1, 0,  0, 0,
          0, c, -s, 0,
          0, s,  c, 0,
          0, 0,  0, 1,
        ]
      }

      function makeMatrixRotateY(angle) {
        const c = Math.cos(angle)
        const s = Math.sin(angle)
        return [
           c, 0, s,  0,
           0, 1, 0,  0,
          -s, 0, c,  0,
           0, 0, 0,  1,
        ]
      }

  function makeDragControl(regl, el) {
    const state = {
      isDragging: false,
      zoom: 0.1,
      rotationX: 1,
      rotationY: 0,
      isTouching: false,
      touchX: 0,
      touchY: 0,
      setup: regl({
        context: {
          view: ({tick}) => {
            const w = el.width/2
            const t = 0.01 * tick
            return [
              makeMatrixTranslate(0,0,-30+8*state.zoom),
              makeMatrixRotateX(-state.rotationX),
              makeMatrixRotateY(-state.rotationY),
              makeMatrixScale(8,8, 8),
            ].reduce(matrixMultiplyMatrix)
          },
          projection: ({viewportWidth, viewportHeight}) =>
            makeMatrixPerspective(45, viewportWidth/viewportHeight, 0.01, 128),

          viewport: () => ({ x: 0, y: 0, width: el.width, height: el.height }),
        },

        uniforms: {
          view: regl.context('view'),
          projection: regl.context('projection'),
          viewport: regl.context('viewport'),
        }
      }),
    }

    function pan(dx, dy) {
      state.rotationX += 0.01 * dy * window.devicePixelRatio
      state.rotationY += 0.01 * dx * window.devicePixelRatio

      state.rotationX = Math.max(0, Math.min(Math.PI/2-0.05, state.rotationX))
    }

    function zoom(z) {
      state.zoom *= Math.pow(2, z)
      state.zoom = Math.max(0.5, Math.min(2, state.zoom))
    }

    el.addEventListener('mousedown', function(evt) {
      evt.preventDefault()
      disableClick = false
      state.isDragging = true
    })

    el.addEventListener('click', function(evt) {
      evt.preventDefault()
      evt.stopPropagation()
    }, true)


    el.ownerDocument.addEventListener('mousemove', function(evt) {
      if(state.isDragging) {
        disableClick = true
        pan(evt.movementX, evt.movementY)
      }
    })

    el.ownerDocument.addEventListener('mouseup', function() {
      state.isDragging = false
    })

    el.addEventListener('wheel', function(evt) {
      if(evt.shiftKey || evt.altKey) {
        return
      }
      evt.preventDefault()
      zoom(evt.wheelDelta/400)
    })

    var touchState = {
      touchIds: [],
      prevDistance: null,
      prevRotation: null,
    }

    el.addEventListener('touchstart', function touchStart(evt) {
      var newIds = Array.prototype.map.call(evt.changedTouches, function(t) {
        return t.identifier;
      })

      touchState.touchIds = touchState.touchIds.concat(newIds).filter(onlyUnique)

      var touches = filterTouches(touchState.touchIds, evt.touches)

      if(touchState.touchIds.length >= 1) {
        touchState.panBase = touchCenter(touches)
      } else {
        touchState.panBase = null
      }

      if (touchState.touchIds.length >= 1) {
        touchState.prevDistance = touchRadius(touches);
        touchState.prevRotation = touchRotation(touches);
        touchState.pinchPivot = touchCenter(touches)
      } else {
        touchState.prevDistance = null;
        touchState.prevRotation = null;
      }

      if(evt.touches.length >= 1) {
        event.preventDefault()
      }
    });

    el.ownerDocument.addEventListener('touchmove', function touchMove(evt) {
      if(touchState.touchIds.length === 0) {
        return;
      }

      var touches = filterTouches(touchState.touchIds, evt.touches)
      var preventDefault = false;

      if(touchState.panBase) {
        var pos = touchCenter(touches)
        pan((pos.x - touchState.panBase.x)/3, (pos.y - touchState.panBase.y)/3)
        touchState.panBase.x = pos.x
        touchState.panBase.y = pos.y
        preventDefault = true;
      }

      if(touchState.prevDistance) {
        var touches = filterTouches(touchState.touchIds, evt.touches)
        var distance = touchRadius(touches);

        zoom((distance - touchState.prevDistance) / 100, touchState.pinchPivot);
        touchState.prevDistance = distance;
        preventDefault = true;
      }

      if(touches.length >= 2) {
        var rotation = touchRotation(touches)
        if(rotation && touchState.prevRotation !== null) {
          touchState.prevRotation = rotation;
        } else if(rotation) {
          touchState.prevRotation = rotation;
        }
        preventDefault = true;
      }

      if(false && preventDefault) {
        evt.preventDefault()
      }
    });

    el.ownerDocument.addEventListener('touchend', function touchEnd(evt) {
      if(touchState.touchIds.length === 0) {
        return
      }
      var oldIds = Array.prototype.map.call(evt.changedTouches, function(t) {
        return t.identifier;
      })

      touchState.touchIds = touchState.touchIds.filter(function diffId(id) {
        return oldIds.indexOf(id) < 0;
      })

      var touches = filterTouches(touchState.touchIds, evt.touches)

      if(touchState.touchIds.length >= 2) {
        touchState.panBase = touchCenter(touches)
      } else {
        touchState.panBase = null
      }

      if(touchState.touchIds.length >= 2) {
        touchState.prevDistance = touchRadius(touches);
        touchState.prevRotation = null;
        touchState.pinchPivot = touchCenter(touches)
      } else {
        touchState.prevDistance = null;
        touchState.prevRotation = null;
      }
    });

    function touchCenter(touches) {
      var length = touches.length
      return Array.prototype.reduce.call(touches, function(sum, touch) {
        return {
          x: sum.x + touch.clientX / length,
          y: sum.y + touch.clientY / length,
        };
      }, {x: 0, y: 0});
    }

    function touchRadius (touches) {
      var length = touches.length
      var center = touchCenter(touches)

      return Array.prototype.reduce.call(touches, function(sum, touch) {
          return sum +
          Math.sqrt(
            Math.pow(touch.clientX - center.x, 2) +
            Math.pow(touch.clientY - center.y, 2)
          );
      }, 0) / length
    }

    function touchRotation(touches) {
      var center = touchCenter(touches)

      var angles = Array.prototype.map.call(touches, function(touch) {
        return Math.atan2(touch.clientY - center.y, touch.clientX - center.x) + Math.PI;
      }, 0);

      var sum = (Array.prototype.reduce.call(angles, function(a,b){
        return a + b;
      }) * 180 / Math.PI);

      return (sum + 360 % 360);
    }


    function filterTouches (ids, touches) {
      return Array.prototype.filter.call(touches, function(t) {
        return ids.indexOf(t.identifier) > -1;
      })
    }

    function onlyUnique(value, index, self) {
      return self.indexOf(value) === index;
    }

    return state
  }

  function makeArrowShader(regl) {
      var roundCapJoin = {
        buffer: regl.buffer([
          [0, -0.5, 0],
          [0, -0.5, 1],
          [0, 0.5, 1],
          [0, -0.5, 0],
          [0, 0.5, 1],
          [0, 0.5, 0],

          [8,0,1],
          [0.5,3,1],
          [2,0,1],

          [8,0,1],
          [2,0,1],
          [0.5,-3,1],
        ]),
        count: 12
      }

      return regl({
        vert: `
          precision highp float;
          attribute vec3 position;
          attribute vec3 pointA, pointB;
          uniform float width;
          uniform vec2 resolution;
          uniform mat4 model, view, projection;

          void main() {
            vec4 clip0 = projection * view * model * vec4(pointA, 1.0);
            vec4 clip1 = projection * view * model * vec4(pointB, 1.0);
            vec2 screen0 = resolution * (0.5 * clip0.xy/clip0.w + 0.5);
            vec2 screen1 = resolution * (0.5 * clip1.xy/clip1.w + 0.5);
            vec2 xBasis = normalize(screen1 - screen0);
            vec2 yBasis = vec2(-xBasis.y, xBasis.x);
            vec2 pt0 = screen0 + width * (position.x * xBasis + position.y * yBasis);
            vec2 pt1 = screen1 + width * (position.x * xBasis + position.y * yBasis);
            vec2 pt = mix(pt0, pt1, position.z);
            vec4 clip = mix(clip0, clip1, position.z);
            gl_Position = vec4(clip.w * (2.0 * pt/resolution - 1.0), clip.z, clip.w);
          }`,

        frag: `
          precision highp float;
          uniform vec4 color;
          void main() {
            gl_FragColor = color;
          }`,

        attributes: {
          position: {
            buffer: roundCapJoin.buffer,
            divisor: 0
          },
          pointA: {
            buffer: regl.prop("points"),
            divisor: 1,
            offset: Float32Array.BYTES_PER_ELEMENT * 0,
            stride: Float32Array.BYTES_PER_ELEMENT * 6
          },
          pointB: {
            buffer: regl.prop("points"),
            divisor: 1,
            offset: Float32Array.BYTES_PER_ELEMENT * 3,
            stride: Float32Array.BYTES_PER_ELEMENT * 6
          }
        },

        uniforms: {
          width: regl.prop("width"),
          color: regl.prop("color"),
          model: regl.prop("model"),
          resolution: regl.prop("resolution")
        },

        depth: {
          enable: regl.prop("depth")
        },

        cull: {
          enable: true,
          face: "back"
        },

        blend: {
          enable: true,
          func: {
            srcRGB: 'src alpha',
            srcAlpha: 1,
            dstRGB: 'one minus src alpha',
            dstAlpha: 1
          },
          equation: {
            rgb: 'add',
            alpha: 'add'
          },
          color: [0, 0, 0, 0]
        },

        stencil: {
          enable: (_, props) => props.stencilId >= 0,
          func: {
            cmp: 'equal',
            ref: 0xff,
            mask: (_, props) => 1 << props.stencilId,
          },
          op: {
            fail: 'keep',
            zfail: 'keep',
            zpass: 'keep'
          },
        },

        polygonOffset: {
          enable: true,
          offset: {
            factor: 0,
            units: 32
          }
        },


        count: roundCapJoin.count,
        instances: regl.prop("segments")
      });
    }

    function makePointShader(regl) {
      var roundCapJoin = {
        buffer: regl.buffer([
          [-1,-1],
          [-1,1],
          [1,1],
          [-1,-1],
          [1,1],
          [1,-1],
        ]),
        count: 6
      }

      return regl({
        vert: `
          precision highp float;
          attribute vec2 position;
          attribute vec2 point;
          uniform float width;
          uniform vec2 resolution;
          uniform mat4 model, view, projection;

          void main() {
            vec4 clip = projection * view * model * vec4(point.x, 0.0, point.y, 1.0);
            gl_Position = vec4(clip.x + width*clip.w*position.x/resolution.x, clip.y + width*clip.w*position.y/resolution.y, clip.z, clip.w);
          }`,

        frag: `
          precision highp float;
          uniform vec4 color;
          void main() {
            gl_FragColor = color;
          }`,

        attributes: {
          position: {
            buffer: roundCapJoin.buffer,
            divisor: 0
          },
          point: {
            buffer: regl.prop("points"),
            divisor: 1,
            offset: Float32Array.BYTES_PER_ELEMENT * 0,
            stride: Float32Array.BYTES_PER_ELEMENT * 2
          },
        },

        uniforms: {
          width: regl.prop("width"),
          color: regl.prop("color"),
          model: regl.prop("model"),
          resolution: regl.prop("resolution")
        },

        depth: {
          enable: regl.prop("depth")
        },

        cull: {
          enable: false,
          face: "back"
        },

        blend: {
          enable: true,
          func: {
            srcRGB: 'src alpha',
            srcAlpha: 1,
            dstRGB: 'one minus src alpha',
            dstAlpha: 1
          },
          equation: {
            rgb: 'add',
            alpha: 'add'
          },
          color: [0, 0, 0, 0]
        },

        stencil: {
          enable: (_, props) => props.stencilId >= 0,
          func: {
            cmp: 'equal',
            ref: 0xff,
            mask: (_, props) => 1 << props.stencilId,
          },
          op: {
            fail: 'keep',
            zfail: 'keep',
            zpass: 'keep'
          },
        },

        polygonOffset: {
          enable: true,
          offset: {
            factor: 0,
            units: 32
          }
        },


        count: roundCapJoin.count,
        instances: regl.prop("segments")
      });
    }

  function makeColorGridShader(regl) {
  	const planeBuffer = regl.buffer([
      -1,0,-1,
      -1,0,+1,
      +1,0,+1,
      -1,0,-1,
      +1,0,+1,
      +1,0,-1,
    ])

    const weightBuffer = regl.buffer([
       1,0,0,0,//
       0,1,0,0,//
       0,0,0,1,
       1,0,0,0,//
       0,0,0,1,
       0,0,1,0,
    ])

    return regl({
      frag: `
      precision highp float;
      varying vec3 vColor;
      void main () {
        gl_FragColor = vec4(vColor,1);
      }`,
      vert: `
      attribute float pointA, pointB, pointC, pointD;
      attribute float instanceId;
      attribute vec3 position;
      attribute vec4 weight;
      uniform mat4 projection, view, model;
      uniform float sidelength;
      uniform float maxheight;
      varying vec3 vColor;
      void main() {
        float row = ceil(instanceId/sidelength);
        float col = ceil(mod(instanceId, sidelength));
        float noedge = float(col < (sidelength - 1.0));
        float height = noedge * dot(vec4(pointA,pointB, pointC, pointD), weight) + (1.0 - noedge) * (pointA * (weight.x+weight.y) + pointC * (weight.w+weight.z));

        

        vColor = vec3(0.5*sqrt(height),(height/maxheight),0.3+0.5*height*height/maxheight/maxheight);
        gl_Position = projection * view * model * vec4(
        position/sidelength - vec3(1.0,0.0,1.0) + 
        vec3(2.0*row/sidelength,
        20.0 * height / sidelength,
        2.0*col/sidelength), 1);
      }`,
      attributes: {
        position: planeBuffer,
        weight: weightBuffer,
        pointA: {
          buffer: regl.prop("points"),
          divisor: 1,
          offset: Float32Array.BYTES_PER_ELEMENT * 0,
          stride: Float32Array.BYTES_PER_ELEMENT * 1
        },
        pointB: {
          buffer: regl.prop("points"),
          divisor: 1,
          offset: Float32Array.BYTES_PER_ELEMENT * 1,
          stride: Float32Array.BYTES_PER_ELEMENT * 1
        },
        pointC: {
          buffer: regl.prop("points"),
          divisor: 1,
          offset: Float32Array.BYTES_PER_ELEMENT * (0 + 256),
          stride: Float32Array.BYTES_PER_ELEMENT * 1
        },
        pointD: {
          buffer: regl.prop("points"),
          divisor: 1,
          offset: Float32Array.BYTES_PER_ELEMENT * (1 + 256),
          stride: Float32Array.BYTES_PER_ELEMENT * 1
        },
        instanceId: {
          buffer: regl.prop("instanceIds"),
          divisor: 1,
          offset: Float32Array.BYTES_PER_ELEMENT * 0,
          stride: Float32Array.BYTES_PER_ELEMENT * 1
        },
      },
      uniforms: {
      	sidelength: regl.prop("sidelength"),
        model: regl.prop("model"),
        maxheight: regl.prop("maxheight"),
      },
      cull: {
        enable: false,
        face: 'back'
      },
      depth: {
        enable: true,
      },
      instances: regl.prop("instances"),
      count: 6,
    })
  }

  const sidelength = 256
  let gridPoints
  let pointBuffer
  let sampleCount = 0
  let maxheight = 1

  function loadHeights(heights) {
      maxheight = 1
      for(let i=0;i<heights.length;i++) {
        if(heights[i] > maxheight) {
          maxheight = heights[i]
        }
      }
      gridPoints(heights) 
  }

   
    $:if(samples && pointBuffer) {
      pointBuffer(samples.flat(1))
      sampleCount = samples.length
    } else {
      sampleCount = 0
    }

    $:if(gridPoints && variance.x && variance.y && Math.abs(correlation) <0.9999) {

      const A = 1/(2*Math.PI*Math.sqrt(variance.x*variance.y)*(Math.sqrt(1-correlation*correlation)||1))
      const density = Array(sidelength*(sidelength+1)+1).fill(null).map((_,i)=> {
          const [x,y] = [
            Math.floor(i/sidelength)/sidelength, 
            (i % sidelength)/sidelength
          ]

            return A * Math.exp(
              -0.5/((1-correlation*correlation)||1) * (
                Math.pow(x-mean.x, 2)/variance.x+
                Math.pow(y-mean.y, 2)/variance.y-
                2*correlation*(x-mean.x)*(y-mean.y)/Math.sqrt(variance.x*variance.y)
              ))
        })


      if(view == 'pdf') {
        loadHeights(density)
      } else {
          const cumulation = Array(sidelength*(sidelength+1)+1).fill(0)

          cumulation[0] = density[0]
          for (let i = 1; i <= sidelength; i++) {
              cumulation[(0)*(sidelength)+(i)] = cumulation[(0)*(sidelength)+(i - 1)] + density[(0)*(sidelength)+(i)];
          }
          for (let i = 0; i <= sidelength; i++) {
              cumulation[(i)*(sidelength)+(0)] = cumulation[(i - 1)*(sidelength)+(0)] + density[(i)*(sidelength)+(0)];
          }
       
          for (let i = 1; i <= sidelength; i++)
          {
              for (let j = 2; j <= sidelength; j++) {
                  cumulation[(i)*(sidelength)+(j)] = cumulation[(i - 1)*(sidelength)+(j)] + cumulation[(i)*(sidelength)+(j - 1)] - cumulation[(i - 1)*(sidelength)+(j - 1)] + density[(i)*(sidelength)+(j)] / 256;
              }
          }

          const max  = 8/cumulation[cumulation.length - 2]
          const l = cumulation.length
          for (let i = 0; i <= l; i++)
          {
              cumulation[i] *= max
          }
        loadHeights(cumulation)
      }

    } else if(gridPoints) {
      loadHeights(Array(sidelength*(sidelength+1)+1).fill(0))
    }
  

  

  onMount(async () => {
    await tick()
    window.onerror = function(msg) {
      document.body.innerHTML = (msg)
    }


	  gl.width = gl.clientWidth * window.devicePixelRatio
  	gl.height = gl.clientHeight * window.devicePixelRatio
   
    const re = createRegl({
    	canvas: gl, 
      extensions: ["ANGLE_instanced_arrays"]
    })

    const camera = makeDragControl(re, gl)
    const drawGauss = makeColorGridShader(re)
    const drawAxis = makeArrowShader(re)
    const drawPoints = makePointShader(re)

    const modelMatrix = makeMatrixScale(1,1,1)
    const axisMatrix = matrixMultiplyMatrix(
      makeMatrixScale(1.05,1.05,1.05),
      makeMatrixTranslate(-1,0.01,-1)
    )

    const axisBuffer = re.buffer([
      //x
      0,0,0,
      1,0,0,

      //z
      0,0,0,
      0,0,1,

      //y
      0,0,0,
      0,0.8,0,

    ]);

    pointBuffer = re.buffer();
    const instanceBuffer = re.buffer(Array(sidelength*sidelength).fill(null).map((_, i) => i));

    gridPoints = re.buffer(Array(sidelength*(sidelength+1)+1).fill(null).map((_,i)=>0))

  	re.frame(({time}) => {

		  gl.width = gl.clientWidth * window.devicePixelRatio
	  	gl.height = gl.clientHeight * window.devicePixelRatio
  	  // clear contents of the drawing buffer
  	  re.clear({
  	    color: [0, 0, 0, 0],
  	    depth: 1
  	  })


  	  camera.setup(() => {
        drawGauss({
          points: gridPoints,
          sidelength: sidelength,
          maxheight: maxheight,
          model: modelMatrix,
          instanceIds: instanceBuffer,
          instances: sidelength*sidelength,
        })
        drawAxis({
            points: axisBuffer,
            model: axisMatrix,
            color: [0,0,0, 1],
            width: 2 * window.devicePixelRatio,
            segments: 3,
            resolution: [gl.width,gl.height],
            depth: false,
          })

        drawPoints({
            points: pointBuffer,
            model: axisMatrix,
            color: [0.5,1,1, 1],
            width: 4 * window.devicePixelRatio,
            segments: sampleCount,
            resolution: [gl.width,gl.height],
            depth: false,
          })
  	  })
  	})

    return () => re.destroy()
  });
</script>

<style>
	.hide {
		display: none;
	}
</style>

<canvas on:click={clickHandler} class:hide={!show} bind:this={gl} class="{className}"></canvas>
