<style lang="scss">
  .wrapper {
    position: relative;
    height: auto;
    max-height: inherit;
  }
  .inner-content {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    max-height: inherit;
  }
</style>

<template>
  <div class="wrapper" ref="shader">
    <div class="inner-content">
      <slot></slot>
    </div>
  </div>
</template>

<script>

const throwError = (message) => {
  throw new Error(`[vue-flat-surface-shader]: ${message}`);
};

const { FSS } = window;

export default {
  name: 'flat-surface-shader',
  props: {
    filter: {
      type: String,
    },
    type: {
      type: String,
      default: 'svg',
    },
    mesh: {
      type: Object,
      default() {
        return {};
      },
    },
    light: {
      type: Object,
      default() {
        return {};
      },
    },
    width: {
      type: Number,
    },
    height: {
      type: Number,
    },
    shouldAnimate: {
      type: Boolean,
      default: true,
    },
  },

  mounted() {
    this.initialize();
  },
  data() {
    return {
      start: Date.now(),
      scene: null,
      renderer: null,
      meshIns: null,
      MESH: {
        width: 1.2,
        height: 1.2,
        depth: 10,
        segments: 16,
        slices: 8,
        xRange: 0.8,
        yRange: 0.1,
        zRange: 1.0,
        ambient: '#555555',
        diffuse: '#FFFFFF',
        speed: 0.002,
      },
      LIGHT: {
        count: 2,
        xyScalar: 1,
        zOffset: 100,
        ambient: '#880066',
        diffuse: '#FF8800',
        speed: 0.001,
        gravity: 1200,
        dampening: 0.95,
        minLimit: 10,
        maxLimit: null,
        minDistance: 20,
        maxDistance: 400,
        autopilot: false,
        draw: true,
        bounds: FSS.Vector3.create(),
        step: FSS.Vector3.create(
          Math.randomInRange(0.2, 1.0),
          Math.randomInRange(0.2, 1.0),
          Math.randomInRange(0.2, 1.0),
        ),
      },
      geometry: new FSS.Plane(),
      material: new FSS.Material(),
      center: FSS.Vector3.create(),
      attractor: FSS.Vector3.create(),
      animationFrameId: null,
    };
  },

  beforeUnmount() {
    this.clearAll();
  },

  watch: {
    light: {
      handler() {
        // this.LIGHT = Object.assign(this.LIGHT, this.light);
        this.createLights();
        this.animate();
      },
      deep: true,
      immediate: false,
    },
    mesh: {
      handler() {
        // this.MESH = Object.assign(this.MESH, this.mesh);
        this.createMesh();
        this.animate();
      },
      deep: true,
      immediate: false,
    },
    shouldAnimate: {
      handler() {
        // this.MESH = Object.assign(this.MESH, this.mesh);
        this.animate();
      },
      deep: true,
      immediate: false,
    },
  },

  methods: {
    initialize() {
      const container = this.$refs.shader;
      this.setRenderer();
      this.scene = new FSS.Scene();
      this.createMesh();
      this.createLights();
      this.addEventListeners();
      this.resize(container.offsetWidth, container.offsetHeight);
      this.animate();
    },

    clearAll() {
      this.removeEventListeners();
      this.renderer.clear();
      const container = this.$refs.shader;
      container.removeChild(this.renderer.element);
      window.cancelAnimationFrame(this.animationFrameId);
    },

    setRenderer() {
      const container = this.$refs.shader;
      if (this.renderer) {
        container.removeChild(this.renderer.element);
      }
      switch (this.type) {
        case 'webgl':
          this.renderer = new FSS.WebGLRenderer();
          break;
        case 'canvas':
          this.renderer = new FSS.CanvasRenderer();
          break;
        case 'svg':
          this.renderer = new FSS.SVGRenderer();
          break;
        default:
          throwError(`Invalid renderer type - ${this.type}`);
      }
      this.renderer.setSize(container.offsetWidth, container.offsetHeight);
      container.insertBefore(this.renderer.element, container.firstChild);
    },

    createMesh() {
      this.MESH = Object.assign(this.MESH, this.mesh);
      this.scene.remove(this.meshIns);
      this.renderer.clear();
      const { MESH } = this;
      this.geometry = new FSS.Plane(
        MESH.width * this.renderer.width,
        MESH.height * this.renderer.height,
        MESH.segments,
        MESH.slices,
      );
      this.material = new FSS.Material(MESH.ambient, MESH.diffuse);
      this.meshIns = new FSS.Mesh(this.geometry, this.material);
      this.scene.add(this.meshIns);
      // Augment vertices for animation
      for (let v = this.geometry.vertices.length - 1; v >= 0; v -= 1) {
        const vertex = this.geometry.vertices[v];
        vertex.anchor = FSS.Vector3.clone(vertex.position);
        vertex.step = FSS.Vector3.create(
          Math.randomInRange(0.2, 1.0),
          Math.randomInRange(0.2, 1.0),
          Math.randomInRange(0.2, 1.0),
        );
        vertex.time = Math.randomInRange(0, Math.PIM2);
      }
    },

    createLights() {
      this.LIGHT = Object.assign(this.LIGHT, this.light);
      for (let l = this.scene.lights.length - 1; l >= 0; l -= 1) {
        const light = this.scene.lights[l];
        this.scene.remove(light);
      }
      this.renderer.clear();
      for (let l = 0; l < this.LIGHT.count; l += 1) {
        const light = new FSS.Light(this.LIGHT.ambient, this.LIGHT.diffuse);
        light.ambientHex = light.ambient.format();
        light.diffuseHex = light.diffuse.format();
        this.scene.add(light);
        // Augment light for animation
        light.mass = Math.randomInRange(0.5, 1);
        light.velocity = FSS.Vector3.create();
        light.acceleration = FSS.Vector3.create();
        light.force = FSS.Vector3.create();

        // Ring SVG Circle
        // light.ring = document.createElementNS(FSS.SVGNS, 'circle');
        // light.ring.setAttributeNS(null, 'stroke', light.ambientHex);
        // light.ring.setAttributeNS(null, 'stroke-width', '0.5');
        // light.ring.setAttributeNS(null, 'fill', 'none');
        // light.ring.setAttributeNS(null, 'r', '10');

        // Core SVG Circle
        // light.core = document.createElementNS(FSS.SVGNS, 'circle');
        // light.core.setAttributeNS(null, 'fill', light.diffuseHex);
        // light.core.setAttributeNS(null, 'r', '4');
      }
    },

    resize(width, height) {
      this.renderer.setSize(width, height);
      FSS.Vector3.set(this.center, this.renderer.halfWidth, this.renderer.halfHeight);
      this.createMesh();
    },

    animate() {
      window.cancelAnimationFrame(this.animationFrameId);
      this.tick();
      this.shaderRender();
      if (this.shouldAnimate) {
        this.animationFrameId = requestAnimationFrame(this.animate);
      }
    },

    updated() {
      this.animate();
    },

    tick() {
      this.MESH = Object.assign(this.MESH, this.mesh);
      this.LIGHT = Object.assign(this.LIGHT, this.light);
      const now = Date.now() - this.start;
      let ox; let oy; let
        oz;
      const offset = this.MESH.depth / 2;
      // Update Bounds
      FSS.Vector3.copy(this.LIGHT.bounds, this.center);
      FSS.Vector3.multiplyScalar(this.LIGHT.bounds, this.LIGHT.xyScalar);
      // Upadte Attractor
      FSS.Vector3.setZ(this.attractor, this.LIGHT.zOffset);
      // Overwrite the Attractor position
      if (this.LIGHT.autopilot) {
        ox = Math.sin(this.LIGHT.step[0] * now * this.LIGHT.speed);
        oy = Math.cos(this.LIGHT.step[1] * now * this.LIGHT.speed);
        FSS.Vector3.set(
          this.attractor,
          this.LIGHT.bounds[0] * ox,
          this.LIGHT.bounds[1] * oy,
          this.LIGHT.zOffset,
        );
      }
      // Animate Lights
      for (let l = this.scene.lights.length - 1; l >= 0; l -= 1) {
        const light = this.scene.lights[l];
        // Reset the z position of the light
        FSS.Vector3.setZ(light.position, this.LIGHT.zOffset);
        // Calculate the force Luke!
        const D = Math.clamp(
          FSS.Vector3.distanceSquared(light.position, this.attractor),
          this.LIGHT.minDistance,
          this.LIGHT.maxDistance,
        );
        const F = (this.LIGHT.gravity * light.mass) / D;
        FSS.Vector3.subtractVectors(light.force, this.attractor, light.position);
        FSS.Vector3.normalise(light.force);
        FSS.Vector3.multiplyScalar(light.force, F);
        // Update the light position
        FSS.Vector3.set(light.acceleration);
        FSS.Vector3.add(light.acceleration, light.force);
        FSS.Vector3.add(light.velocity, light.acceleration);
        FSS.Vector3.multiplyScalar(light.velocity, this.LIGHT.dampening);
        FSS.Vector3.limit(light.velocity, this.LIGHT.minLimit, this.LIGHT.maxLimit);
        FSS.Vector3.add(light.position, light.velocity);
      }
      // Animate Vertices
      for (let v = this.geometry.vertices.length - 1; v >= 0; v -= 1) {
        const vertex = this.geometry.vertices[v];
        ox = Math.sin(vertex.time + vertex.step[0] * now * this.MESH.speed);
        oy = Math.cos(vertex.time + vertex.step[1] * now * this.MESH.speed);
        oz = Math.sin(vertex.time + vertex.step[2] * now * this.MESH.speed);
        FSS.Vector3.set(
          vertex.position,
          this.MESH.xRange * this.geometry.segmentWidth * ox,
          this.MESH.yRange * this.geometry.sliceHeight * oy,
          this.MESH.zRange * offset * oz - offset,
        );
        FSS.Vector3.add(vertex.position, vertex.anchor);
      }
      // Set the Geometry to dirty
      this.geometry.dirty = true;
    },

    shaderRender() {
      this.MESH = Object.assign(this.MESH, this.mesh);
      this.LIGHT = Object.assign(this.LIGHT, this.light);
      this.renderer.render(this.scene);
      // Draw Lights
      if (this.LIGHT.draw && this.shouldAnimate) {
        for (let l = this.scene.lights.length - 1; l >= 0; l -= 1) {
          const light = this.scene.lights[l];
          let lx = light.position[0]; let
            ly = light.position[1];
          switch (this.type) {
            case 'canvas':
              this.renderer.context.lineWidth = 0.5;
              this.renderer.context.beginPath();
              this.renderer.context.arc(lx, ly, 10, 0, Math.PIM2);
              this.renderer.context.strokeStyle = light.ambientHex;
              this.renderer.context.stroke();
              this.renderer.context.beginPath();
              this.renderer.context.arc(lx, ly, 4, 0, Math.PIM2);
              this.renderer.context.fillStyle = light.diffuseHex;
              this.renderer.context.fill();
              break;
            case 'svg':
              lx += this.renderer.halfWidth;
              ly = this.renderer.halfHeight - ly;
              light.core.setAttributeNS(null, 'fill', light.diffuseHex);
              light.core.setAttributeNS(null, 'cx', lx);
              light.core.setAttributeNS(null, 'cy', ly);
              this.renderer.element.appendChild(light.core);
              light.ring.setAttributeNS(null, 'stroke', light.ambientHex);
              light.ring.setAttributeNS(null, 'cx', lx);
              light.ring.setAttributeNS(null, 'cy', ly);
              this.renderer.element.appendChild(light.ring);
              break;
            default:
              break;
          }
        }
      }
    },

    addEventListeners() {
      const container = this.$refs.shader;
      window.addEventListener('resize', this.onWindowResize);
      container.addEventListener('mousemove', this.onMouseMove);
    },
    removeEventListeners() {
      const container = this.$refs.shader;
      window.removeEventListener('resize', this.onWindowResize);
      container.removeEventListener('mousemove', this.onMouseMove);
    },

    onWindowResize() {
      const container = this.$refs.shader;
      this.resize(container.offsetWidth, container.offsetHeight);
      this.animate();
    },

    onMouseMove(event) {
      FSS.Vector3.set(this.attractor, event.x, this.renderer.height - event.y);
      FSS.Vector3.subtract(this.attractor, this.center);
    },
  },

};
</script>
