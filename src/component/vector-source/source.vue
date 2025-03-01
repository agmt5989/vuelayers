<script>
  import VectorSource from 'ol/source/Vector'
  import { fetch } from 'whatwg-fetch'
  import { vectorSource } from '../../mixin'
  import { createGeoJsonFmt, getFeatureId, initializeFeature, loadingAll, transform } from '../../ol-ext'
  import { constant, difference, isEmpty, isFinite, isFunction, stubArray } from '../../util/minilo'
  import { makeWatchers } from '../../util/vue-helpers'

  export default {
    name: 'vl-source-vector',
    mixins: [vectorSource],
    props: {
      /**
       * Array of GeoJSON features with coordinates in the map view projection.
       * @type {Object[]} features
       */
      features: {
        type: Array,
        default: stubArray,
      },
      /**
       * Source loader factory.
       * Source loader should load features from some remote service, decode them and pas to `features` prop to render.
       * @type {(function(): FeatureLoader|undefined)} loaderFactory
       */
      loaderFactory: {
        type: Function,
        default: defaultLoaderFactory,
      },
      /**
       * Source format factory
       * @type {(function(): Feature|undefined)} formatFactory
       */
      formatFactory: {
        type: Function,
        default: defaultFormatFactory,
      },
      /**
       * String or url factory
       * @type {(string|function(): string|FeatureUrlFunction|undefined)} url
       */
      url: [String, Function],
      /**
       * Loading strategy factory.
       * Extent here in map view projection.
       * @type {(function(): LoadingStrategy|undefined)} strategyFactory
       */
      strategyFactory: {
        type: Function,
        default: defaultStrategyFactory,
      },
      overlaps: {
        type: Boolean,
        default: true,
      },
    },
    computed: {
      urlFunc () {
        if (!this.url) {
          return
        }

        let url = this.url
        if (!isFunction(url)) {
          url = constant(this.url)
        }
        // wrap strategy function to transform map view projection to source projection
        return (extent, resolution, projection) => url(
          transformExtent(extent, projection, this.resolvedDataProjection),
          resolution,
          this.resolvedDataProjection,
        )
      },
      loaderFunc () {
        const loader = this.loaderFactory(this)
        // wrap strategy function to transform map view projection to source projection
        return async (extent, resolution, projection) => {
          const features = await loader(
            transformExtent(extent, projection, this.resolvedDataProjection),
            resolution,
            this.resolvedDataProjection,
          )

          if (Array.isArray(features)) {
            this.addFeatures(features)
          }
        }
      },
      loadingStrategy () {
        return this.strategyFactory()
      },
      dataFormat () {
        return this.formatFactory()
      },
    },
    methods: {
      /**
       * @return {VectorSource}
       * @protected
       */
      createSource () {
        return new VectorSource({
          attributions: this.attributions,
          features: this._featuresCollection,
          projection: this.resolvedDataProjection,
          loader: this.loaderFunc,
          useSpatialIndex: this.useSpatialIndex,
          wrapX: this.wrapX,
          logo: this.logo,
          strategy: this.loadingStrategy,
          format: this.dataFormat,
          url: this.urlFunc,
          overlaps: this.overlaps,
        })
      },
      /**
       * @return {void}
       * @protected
       */
      mount () {
        this::vectorSource.methods.mount()
        this.addFeatures(this.features)
      },
      /**
       * @return {void}
       * @protected
       */
      unmount () {
        this.clear()
        this::vectorSource.methods.unmount()
      },
    },
    watch: {
      features: {
        deep: true,
        handler (features) {
          if (!this.$source) return

          features = features.slice().map(feature => initializeFeature({
            ...feature,
          }))
          this.addFeatures(features)

          const forRemove = difference(this.getFeatures(), features, (a, b) => getFeatureId(a) === getFeatureId(b))
          this.removeFeatures(forRemove)
        },
      },
      ...makeWatchers([
        'loadingStrategy',
        'dataFormat',
        'urlFunc',
        'loaderFactory',
        'formatFactory',
        'strategyFactory',
        'overlaps',
      ], () => function () {
        this.scheduleRecreate()
      }),
    },
  }

  /**
   * @return {LoadingStrategy}
   */
  function defaultStrategyFactory () {
    return loadingAll
  }

  /**
   * @return {GeoJSON}
   */
  function defaultFormatFactory () {
    return createGeoJsonFmt()
  }

  /**
   * Default loader for provided URL.
   *
   * @param vm
   * @return {Function}
   */
  function defaultLoaderFactory (vm) {
    return (extent, resolution, projection) => {
      let url = vm.$source.getUrl()
      if (isFunction(url)) {
        url = url(extent, resolution, projection)
      }

      if (isEmpty(url)) {
        return []
      }

      return fetch(url, {
        credentials: 'same-origin',
        mode: 'cors',
      }).then(response => response.text())
        .then(text => {
          if (!vm.$source) {
            return []
          }

          return vm.$source.getFormat().readFeatures(text, {
            featureProjection: vm.viewProjection,
            dataProjection: vm.resolvedDataProjection,
          })
        })
    }
  }

  function transformExtent (extent, sourceProj, destProj) {
    extent = extent.slice()
    if (isFinite(extent[0]) && isFinite(extent[1])) {
      [extent[0], extent[1]] = transform([extent[0], extent[1]], sourceProj, destProj)
    }
    if (isFinite(extent[2]) && isFinite(extent[3])) {
      [extent[2], extent[3]] = transform([extent[2], extent[3]], sourceProj, destProj)
    }
    return extent
  }
</script>
