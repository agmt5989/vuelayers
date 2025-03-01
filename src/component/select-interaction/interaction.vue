<template>
  <i :class="[$options.name]" style="display: none !important;">
    <slot :features="featuresDataProj"/>
  </i>
</template>

<script>
  import { never, shiftKeyOnly, singleClick } from 'ol/events/condition'
  import Feature from 'ol/Feature'
  import SelectInteraction from 'ol/interaction/Select'
  import Vue from 'vue'
  import { merge as mergeObs } from 'rxjs/observable'
  import { map as mapOp, debounceTime } from 'rxjs/operators'
  import { interaction, projTransforms, stylesContainer, featuresContainer } from '../../mixin'
  import { getFeatureId, createStyle, defaultEditStyle, getLayerId, initializeFeature } from '../../ol-ext'
  import { observableFromOlEvent } from '../../rx-ext'
  import { hasInteraction, hasMap } from '../../util/assert'
  import { constant, difference, forEach, isEqual, isFunction, mapValues, stubArray } from '../../util/minilo'
  import mergeDescriptors from '../../util/multi-merge-descriptors'
  import { makeWatchers } from '../../util/vue-helpers'

  export default {
    name: 'vl-interaction-select',
    mixins: [interaction, featuresContainer, stylesContainer, projTransforms],
    props: {
      /**
       * A function that takes an `ol.Feature` and an `ol.layer.Layer` and returns `true` if the feature may be selected or `false` otherwise.
       * @type {function|undefined}
       */
      filter: {
        type: Function,
        default: constant(true),
      },
      /**
       * A list of layers from which features should be selected. Alternatively, a filter function can be provided.
       * @type {string[]|function|undefined}
       */
      layers: {
        type: [Array, Function],
        default: undefined,
      },
      /**
       * Hit-detection tolerance. Pixels inside the radius around the given position will be checked for features.
       * This only works for the canvas renderer and not for WebGL.
       * @type {number}
       */
      hitTolerance: {
        type: Number,
        default: 0,
      },
      /**
       * A boolean that determines if the default behaviour should select only single features or all (overlapping)
       * features at the clicked map position.
       * @type {boolean}
       */
      multi: {
        type: Boolean,
        default: false,
      },
      /**
       * Selected features as array of GeoJSON features with coordinates in the map view projection.
       * @type {string[]|number[]|Object[]}
       */
      features: {
        type: Array,
        default: stubArray,
      },
      /**
       * Wrap the world horizontally on the selection overlay.
       * @type {boolean}
       */
      wrapX: {
        type: Boolean,
        default: true,
      },
      /**
       * A function that takes an `ol.MapBrowserEvent` and returns a boolean to indicate whether that event should
       * be handled. By default, this is `ol.events.condition.never`. Use this if you want to use different events
       * for `add` and `remove` instead of `toggle`.
       * @type {function|undefined}
       */
      addCondition: {
        type: Function,
        default: never,
      },
      /**
       * A function that takes an `ol.MapBrowserEvent` and returns a boolean to indicate whether that event should be handled.
       * This is the event for the selected features as a whole. By default, this is `ol.events.condition.singleClick`.
       * Clicking on a feature selects that feature and removes any that were in the selection. Clicking outside any feature
       * removes all from the selection.
       * @type {function|undefined}
       */
      condition: {
        type: Function,
        default: singleClick,
      },
      /**
       * A function that takes an `ol.MapBrowserEvent` and returns a boolean to indicate whether that event should be handled.
       * By default, this is `ol.events.condition.never`. Use this if you want to use different events for `add` and `remove`
       * instead of `toggle`.
       * @type {function|undefined}
       */
      removeCondition: {
        type: Function,
        default: never,
      },
      /**
       * A function that takes an `ol.MapBrowserEvent` and returns a boolean to indicate whether that event should be handled.
       * This is in addition to the `condition` event. By default, `ol.events.condition.shiftKeyOnly`, i.e. pressing `shift`
       * as well as the `condition` event, adds that feature to the current selection if it is not currently selected,
       * and removes it if it is.
       * @type {function|undefined}
       */
      toggleCondition: {
        type: Function,
        default: shiftKeyOnly,
      },
    },
    computed: {
      layerFilter () {
        return Array.isArray(this.layers)
          ? layer => this.layers.includes(getLayerId(layer))
          : this.layers
      },
    },
    methods: {
      /**
       * @return {Select}
       * @protected
       */
      createInteraction () {
        return new SelectInteraction({
          features: this._featuresCollection,
          multi: this.multi,
          wrapX: this.wrapX,
          filter: this.filter,
          layers: this.layerFilter,
          style: this.createStyleFunc(),
          addCondition: this.addCondition,
          condition: this.condition,
          removeCondition: this.removeCondition,
          toggleCondition: this.toggleCondition,
        })
      },
      /**
       * @return {function(feature: Feature): Style}
       * @protected
       */
      getDefaultStyles () {
        const defaultStyles = mapValues(defaultEditStyle(), styles => styles.map(createStyle))

        return function __selectDefaultStyleFunc (feature) {
          if (feature.getGeometry()) {
            return defaultStyles[feature.getGeometry().getType()]
          }
        }
      },
      /**
       * @returns {Object}
       * @protected
       */
      getServices () {
        return mergeDescriptors(
          this::interaction.methods.getServices(),
          this::stylesContainer.methods.getServices(),
        )
      },
      /**
       * @return {Interaction|undefined}
       * @protected
       */
      getStyleTarget () {
        return this.$interaction
      },
      /**
       * @return {void}
       * @protected
       */
      mount () {
        this::interaction.methods.mount()
        this.features.forEach(this.select)
      },
      /**
       * @return {void}
       * @protected
       */
      unmount () {
        this.unselectAll()
        this::interaction.methods.unmount()
      },
      /**
       * @param {Object|Vue|Feature|string|number} feature
       * @return {void}
       * @throws {Error}
       */
      select (feature) {
        hasMap(this)

        if (feature instanceof Vue) {
          feature = feature.$feature
        }

        if (!(feature instanceof Feature)) {
          const featureId = getFeatureId(feature)
          if (!featureId) {
            throw new Error('Undefined feature id')
          }

          feature = undefined
          forEach(this.$map.getLayers().getArray(), layer => {
            if (this.layerFilter && !this.layerFilter(layer)) {
              return false
            }

            const source = layer.getSource()
            if (source && isFunction(source.getFeatureById)) {
              feature = source.getFeatureById(featureId)
            }

            return !feature
          })
        }

        if (!feature) return

        this.addFeature(feature)
      },
      /**
       * @param {Object|Vue|Feature|string|number} feature
       * @return {void}
       */
      unselect (feature) {
        this.removeFeature(feature)
      },
      /**
       * Removes all features from selection.
       * @return {void}
       */
      unselectAll () {
        this.clearFeatures()
      },
      /**
       * @param {Array<{style: Style, condition: (function|boolean|undefined)}>|function(feature: Feature): Style|Vue|undefined} styles
       * @return {void}
       * @protected
       */
      setStyle (styles) {
        if (styles !== this._styles) {
          this._styles = styles
          this.scheduleRefresh()
        }
      },
      /**
       * @return {void}
       * @protected
       */
      subscribeAll () {
        this::interaction.methods.subscribeAll()
        this::subscribeToInteractionChanges()
      },
    },
    watch: {
      features: {
        deep: true,
        handler (features) {
          if (!this.$interaction) return

          features = features.slice().map(feature => initializeFeature({
            ...feature,
          }))
          this.addFeatures(features)

          let forUnselect = difference(this.getFeatures(), features, (a, b) => getFeatureId(a) === getFeatureId(b))
          this.removeFeatures(forUnselect)
        },
      },
      featuresDataProj: {
        deep: true,
        handler (value, prev) {
          if (!isEqual(value, prev)) {
            this.$emit('update:features', value)
          }
        },
      },
      ...makeWatchers([
        'filter',
        'hitTolerance',
        'multi',
        'wrapX',
        'addCondition',
        'condition',
        'removeCondition',
        'toggleCondition',
      ], () => function () {
        this.scheduleRecreate()
      }),
    },
    stubVNode: {
      empty: false,
      attrs () {
        return {
          class: this.$options.name,
        }
      },
    },
  }

  /**
   * @return {void}
   * @private
   */
  function subscribeToInteractionChanges () {
    hasInteraction(this)

    const select = observableFromOlEvent(this._featuresCollection, 'add')
      .pipe(
        mapOp(({ element }) => ({ type: 'select', feature: element }))
      )
    const unselect = observableFromOlEvent(this._featuresCollection, 'remove')
      .pipe(
        mapOp(({ element }) => ({ type: 'unselect', feature: element }))
      )
    const events = mergeObs(select, unselect)

    this.subscribeTo(events, evt => this.$emit(evt.type, evt.feature))
    // emit event to allow `sync` modifier
    this.subscribeTo(events.pipe(debounceTime(1000 / 60)), () => {
      this.$emit('update:features', this.featuresDataProj)
    })
  }
</script>
