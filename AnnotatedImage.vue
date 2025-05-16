<!-- <script type="text/x-template" id="annotated-images-component-template"></script> -->
<template>
  <div id="img-container" ref="imgContainer">
    <img :src="mediaLink" ref="img" @load="onImageLoad" />
    <canvas v-if="showAnnotations" ref="canvas" id="img-canvas"></canvas>

    <!-- Vue-орієнтований рендер міток -->
    <div
      v-for="(label, index) in drawnLabels"
      :key="index"
      class="label"
      :style="{
        left: `${label.x}px`,
        top: `${label.y}px`,
        fontSize: '10px',
        backgroundColor: label.color,
      }"
    >
      {{ label.text }}
    </div>
  </div>
</template>

<script>
const AnnotatedImages = {
  template: "#vision-media-popup-component-template",
  props: [mediaLink, showAnnotations],
  data() {
    return {
      bboxes: {},
      drawnLabels: [],
    };
  },
  watch: {
    showAnnotations(val) {
      if (val) this.processAnnotations();
    },
  },
  methods: {
    convertAnnotationsToBboxes(annotations) {
      const bboxes = {};

      for (const category in annotations) {
        for (const id in annotations[category]) {
          const item = annotations[category][id];
          const label = item.label || item.rtype;

          const coords = [...item.coords];
          if (
            item.threshold !== undefined &&
            typeof item.threshold === "number"
          ) {
            coords.push(item.threshold);
          }

          if (!bboxes[label]) {
            bboxes[label] = [];
          }

          bboxes[label].push(coords);
        }
      }

      return bboxes;
    },

    drawBoundingBox([x, y, width, height], color, labelText) {
      const canvas = this.$refs.canvas;
      const rect = new paper.Path.Rectangle({
        rectangle: new paper.Rectangle(x, y, width, height),
        strokeColor: color,
        strokeWidth: 2,
      });

      this.drawnLabels.push({
        text: labelText,
        x: x - 1,
        y: y - 16,
        color,
      });
    },

    onImageLoad() {
      if (this.showAnnotations) {
        this.initBoundingBoxes();
      }
    },

    initBoundingBoxes() {
      const img = this.$refs.img;
      const canvas = this.$refs.canvas;
      this.drawnLabels = [];

      canvas.width = img.width;
      canvas.height = img.height;

      paper.setup(canvas);

      const scaleX = img.width / img.naturalWidth;
      const scaleY = img.height / img.naturalHeight;

      const categories = [
        { label: "operator", color: "deepskyblue" },
        { label: "person", color: "lightgreen" },
        { label: "Workstation", color: "tomato" },
      ];

      categories.forEach(({ label, color }) => {
        (this.bboxes[label] || []).forEach((box) => {
          const [x, y, width, height] = box.slice(0, 4);
          this.drawBoundingBox(
            [x * scaleX, y * scaleY, width * scaleX, height * scaleY],
            color,
            label
          );
        });
      });

      paper.view.draw();
    },

    processAnnotations() {
      this.bboxes = this.convertAnnotationsToBboxes(
        image_step.step_output.annotations
      );

      this.$nextTick(() => {
        const img = this.$refs.img;
        if (img.complete) {
          this.initBoundingBoxes();
        }
      });

      window.addEventListener("resize", this.initBoundingBoxes);
    },
  },
  beforeDestroy() {
    window.removeEventListener("resize", this.initBoundingBoxes);
  },
};
</script>

<style scoped>
#img-container {
  position: relative;
  display: inline-block;
}

#img {
  display: block;
  width: 100%;
  height: auto;
}

#img-canvas {
  position: absolute;
  top: 0;
  left: 0;
  z-index: 10;
  pointer-events: none;
}

.label {
  position: absolute;
  padding: 0 4px;
  font-family: sans-serif;
  border-radius: 2px;
  z-index: 20;
}
</style>
