<template lang="pug">
// geojs annotation layer
</template>

<script>
import bind from 'lodash-es/bind';
import forEach from 'lodash-es/forEach';
import indexOf from 'lodash-es/indexOf';
import differenceBy from 'lodash-es/differenceBy';

const annotationTypes = ['line', 'point', 'polygon', 'rectangle', 'edit'];
const stateEvents = ['add', 'update', 'remove'];

export default {
  props: {
    labels: {
      type: Boolean,
      default: false,
    },
    editable: {
      type: Boolean,
      default: false,
    },
    drawing: {
      type: [String, Boolean],
      default: false,
      validator(value) {
        return value === false || indexOf(annotationTypes, value) >= 0;
      },
    },
    editing: {
      type: [Number, Boolean],
      default: false,
      validator(value) {
        return value === false || value >= 0;
      },
    },
    annotations: {
      type: Array,
      default: () => [],
    },
    initialGeojson: {
      type: [Object, Array],
      default: () => Object(),
    },
  },
  data() {
    return {
      state: [],
    };
  },
  computed: {
    activeAnnotation() {
      return this.$geojsLayer.annotationById(this.editing);
    },
    geojson: {
      cache: false,
      get() {
        return this.$geojsLayer.geojson();
      },
    },
  },
  watch: {
    drawing() {
      this.$geojsLayer.mode(this.drawing || null);
    },
    state() {
      this.$emit('update:annotations', this.state);
    },
    annotations() {
      const toRemove = differenceBy(
        this.$geojsLayer.annotations(), this.annotations,
        annotation => annotation.id(),
      );
      const toAdd = differenceBy(
        this.annotations, this.$geojsLayer.annotations(),
        annotation => annotation.id(),
      );
      forEach(toRemove, annotation => this.$geojsLayer.removeAnnotation(annotation));
      forEach(toAdd, annotation => this.$geojsLayer.addAnnotation(annotation));
    },
  },
  mounted() {
    // This is in place purely for testing because there is no way
    // in @vue/test-utils to put mocks in place *before* mount is called.
    //   https://github.com/vuejs/vue-test-utils/issues/560
    this.$parent = this.$parent || this.$options.testParent;

    if (!this.$parent || !this.$parent.$geojsMap) {
      throw new Error('Annotation layer must be a child of a GeojsMapViewport');
    }
    this.$geojs = this.$parent.$geojs;
    this.$geojsLayer = this.$parent.$geojsMap.createLayer(
      'annotation', {
        showLabels: this.labels,
        clickToEdit: this.editable,
      });
    this.$geojsLayer.geojson(this.initialGeojson);
    this.state = this.$geojsLayer.annotations();
    this.$geojsLayer.geoOn(
      this.$geojs.event.annotation.mode,
      bind(this.resetAnnotationMode, this),
    );

    forEach(stateEvents, (event) => {
      this.$geojsLayer.geoOn(
        this.$geojs.event.annotation[event],
        bind(this.resetAnnotationState, this),
      );
    });
  },
  methods: {
    resetAnnotationMode() {
      const mode = this.$geojsLayer.mode() || false;
      let editing = false;

      this.resetAnnotationState();
      if (mode === 'edit') {
        editing = this.$geojsLayer.currentAnnotation.id();
      }

      if (this.drawing !== mode) {
        this.$emit('update:drawing', mode);
      }
      if (editing !== this.editing) {
        this.$emit('update:editing', editing);
      }
    },
    resetAnnotationState() {
      this.state = this.$geojsLayer.annotations();
    },
  },
};
</script>