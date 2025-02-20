<!--
  - Copyright 2021 James Lyne
  -
  - Licensed under the Apache License, Version 2.0 (the "License");
  - you may not use this file except in compliance with the License.
  - You may obtain a copy of the License at
  -
  - http://www.apache.org/licenses/LICENSE-2.0
  -
  - Unless required by applicable law or agreed to in writing, software
  - distributed under the License is distributed on an "AS IS" BASIS,
  - WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  - See the License for the specific language governing permissions and
  - limitations under the License.
  -->

<template>
	<div class="map" :style="{backgroundColor: mapBackground }" v-bind="$attrs" :aria-label="mapTitle">
		<template v-if="leaflet">
			<MapLayer v-for="[name, map] in maps" :key="name" :map="map" :name="name" :leaflet="leaflet"></MapLayer>
			<PlayersLayer v-if="playerMarkersEnabled" :leaflet="leaflet"></PlayersLayer>
			<MarkerSetLayer v-for="[name, markerSet] in markerSets" :key="name" :markerSet="markerSet" :leaflet="leaflet"></MarkerSetLayer>

			<LogoControl v-for="logo in logoControls" :key="JSON.stringify(logo)" :options="logo" :leaflet="leaflet"></LogoControl>
			<CoordinatesControl v-if="coordinatesControlEnabled" :leaflet="leaflet"></CoordinatesControl>
			<LinkControl v-if="linkControlEnabled" :leaflet="leaflet"></LinkControl>
			<ClockControl v-if="clockControlEnabled" :leaflet="leaflet"></ClockControl>

			<LoginControl v-if="loginEnabled" :leaflet="leaflet"></LoginControl>
			<ChatControl v-if="chatBoxEnabled" :leaflet="leaflet"></ChatControl>
		</template>
	</div>

	<MapContextMenu :leaflet="leaflet" v-if="leaflet"></MapContextMenu>
</template>

<script lang="ts">
import {computed, ref, defineComponent} from "@vue/runtime-core";
import {CRS, LatLng, LatLngBounds, PanOptions, ZoomPanOptions} from 'leaflet';
import {useStore} from '@/store';
import MapLayer from "@/components/map/layer/MapLayer.vue";
import PlayersLayer from "@/components/map/layer/PlayersLayer.vue";
import MarkerSetLayer from "@/components/map/layer/MarkerSetLayer.vue";
import CoordinatesControl from "@/components/map/control/CoordinatesControl.vue";
import ClockControl from "@/components/map/control/ClockControl.vue";
import LinkControl from "@/components/map/control/LinkControl.vue";
import ChatControl from "@/components/map/control/ChatControl.vue";
import LogoControl from "@/components/map/control/LogoControl.vue";
import {MutationTypes} from "@/store/mutation-types";
import LiveAtlasLeafletMap from "@/leaflet/LiveAtlasLeafletMap";
import {LoadingControl} from "@/leaflet/control/LoadingControl";
import MapContextMenu from "@/components/map/MapContextMenu.vue";
import {LiveAtlasLocation, LiveAtlasPlayer, LiveAtlasMapViewTarget} from "@/index";
import LoginControl from "@/components/map/control/LoginControl.vue";

export default defineComponent({
	components: {
		MapContextMenu,
		MapLayer,
		PlayersLayer,
		MarkerSetLayer,
		CoordinatesControl,
		ClockControl,
		LinkControl,
		ChatControl,
		LogoControl,
		LoginControl
	},

	setup() {
		const store = useStore(),
			leaflet = undefined as any,

			maps = computed(() => store.state.maps),
			markerSets = computed(() => store.state.markerSets),
			configuration = computed(() => store.state.configuration),

			playerMarkersEnabled = computed(() => store.getters.playerMarkersEnabled),
			coordinatesControlEnabled = computed(() => store.getters.coordinatesControlEnabled),
			clockControlEnabled = computed(() => store.getters.clockControlEnabled),
			linkControlEnabled = computed(() => store.state.components.linkControl),
			chatBoxEnabled = computed(() => store.state.components.chatBox),
			loginEnabled = computed(() => store.state.components.login),
			logoControls = computed(() => store.state.components.logoControls),

			currentWorld = computed(() => store.state.currentWorld),
			currentMap = computed(() => store.state.currentMap),
			mapBackground = computed(() => store.getters.mapBackground),

			followTarget = computed(() => store.state.followTarget),
			viewTarget = computed(() => store.state.viewTarget),
			parsedUrl = computed(() => store.state.parsedUrl),

			//Location and zoom to pan to upon next projection change
			scheduledView = ref<LiveAtlasMapViewTarget|null>(null),

			mapTitle = computed(() => store.state.messages.mapTitle);

		return {
			leaflet,
			maps,
			markerSets,
			configuration,

			playerMarkersEnabled,
			coordinatesControlEnabled,
			clockControlEnabled,
			linkControlEnabled,
			chatBoxEnabled,
			loginEnabled,

			logoControls,
			followTarget,
			viewTarget,
			parsedUrl,
			mapBackground,

			currentWorld,
			currentMap,

			scheduledView,

			mapTitle
		}
	},

	watch: {
		followTarget: {
			handler(newValue, oldValue) {
				if (newValue) {
					this.updateFollow(newValue, !oldValue || newValue.name !== oldValue.name);
				}
			},
			deep: true
		},
		viewTarget: {
			handler(newValue) {
				if (newValue) {
					//Immediately clear if on the correct world, to allow repeated panning
					if (this.currentWorld && newValue.location.world === this.currentWorld.name) {
						useStore().commit(MutationTypes.CLEAR_VIEW_TARGET, undefined);
					}

					this.setView(newValue);
				}
			},
			deep: true
		},
		currentMap(newValue, oldValue) {
			if(this.leaflet && newValue) {
				let viewTarget = this.scheduledView;

				if(!viewTarget && oldValue) {
					viewTarget = {location: oldValue.latLngToLocation(this.leaflet.getCenter(), 64) as LiveAtlasLocation};
				} else if(!viewTarget) {
					viewTarget = {location: {x: 0, y: 0, z: 0} as LiveAtlasLocation};
				}

				viewTarget.options = {
					animate: false,
					noMoveStart: false,
				}

				this.setView(viewTarget);
				this.scheduledView = null;
			}
		},
		currentWorld(newValue, oldValue) {
			const store = useStore();

			if(newValue) {
				store.state.currentMapProvider!.populateWorld(newValue);
				let viewTarget = this.scheduledView || {} as LiveAtlasMapViewTarget;

				// Abort if follow target is present, to avoid panning twice
				if(store.state.followTarget && store.state.followTarget.location.world === newValue.name) {
					return;
				// Abort if pan target is present, to avoid panning to the wrong place.
				// Also clear it to allow repeated panning.
				} else if(store.state.viewTarget && store.state.viewTarget.location.world === newValue.name) {
					store.commit(MutationTypes.CLEAR_VIEW_TARGET, undefined);
					return;
				// Otherwise pan to url location, if present
				} else if(store.state.parsedUrl?.location) {
					viewTarget.location = store.state.parsedUrl.location;

					if(!oldValue) {
						if(typeof store.state.parsedUrl?.zoom !== 'undefined') {
							viewTarget.zoom = store.state.parsedUrl?.zoom;
						} else if(typeof newValue.defaultZoom !== 'undefined') {
							viewTarget.zoom = newValue.defaultZoom;
						} else {
							viewTarget.zoom = store.state.configuration.defaultZoom;
						}
					}

					store.commit(MutationTypes.CLEAR_PARSED_URL, undefined);
				// Otherwise pan to world center
				} else {
					viewTarget.location = newValue.center;
				}

				if(viewTarget.zoom == null) {
					if(typeof newValue.defaultZoom !== 'undefined') {
						viewTarget.zoom = newValue.defaultZoom;
					} else {
						viewTarget.zoom = store.state.configuration.defaultZoom;
					}
				}

				//Set pan location for when the projection changes
				this.scheduledView = viewTarget;
			}
		},
		parsedUrl: {
			handler(newValue) {
				if(!newValue || !this.currentMap || !this.leaflet) {
					return;
				}

				this.setView({
					location: {...newValue.location, world: newValue.world},
					map: newValue.map,
					zoom: newValue.zoom,
					options: {
						animate: false,
						noMoveStart: true,
					}
				});
			},
			deep: true,
		}
	},

	mounted() {
		this.leaflet = new LiveAtlasLeafletMap(this.$el.nextElementSibling, Object.freeze({
			zoom: this.configuration.defaultZoom,
			center: new LatLng(0, 0),
			fadeAnimation: false,
			zoomAnimation: true,
			zoomControl: true,
			preferCanvas: true,
			attributionControl: false,
			crs: CRS.Simple,
			worldCopyJump: false,
			// markerZoomAnimation: false,
		})) as LiveAtlasLeafletMap;

		window.addEventListener('keydown', this.handleKeydown);

		this.leaflet.createPane('vectors');

		this.leaflet.addControl(new LoadingControl({
			position: 'topleft',
			delayIndicator: 500,
		}));

		this.leaflet.on('moveend', () => {
			if(this.currentMap) {
				useStore().commit(MutationTypes.SET_CURRENT_LOCATION, this.currentMap
					.latLngToLocation(this.leaflet!.getCenter(), 64));
			}
		});

		this.leaflet.on('zoomend', () => {
			useStore().commit(MutationTypes.SET_CURRENT_ZOOM, this.leaflet!.getZoom());
		});
	},

	unmounted() {
		window.removeEventListener('keydown', this.handleKeydown);
		this.leaflet.remove();
	},

	methods: {
		handleKeydown(e: KeyboardEvent) {
			if(e.key === 'Escape') {
				e.preventDefault();
				this.leaflet.getContainer().focus();
			}
		},
		setView(target: LiveAtlasMapViewTarget) {
			const store = useStore(),
				currentWorld = store.state.currentWorld,
				currentMap = store.state.currentMap?.name,
				targetWorld = target.location.world ? store.state.worlds.get(target.location.world) : currentWorld;

			if(!this.leaflet) {
				console.warn('Ignoring setView as leaflet not initialised');
				return;
			}

			if(!targetWorld) {
				console.warn(`Ignoring setView with unknown world ${target.location.world}`);
				return;
			}

			if(targetWorld && (targetWorld !== currentWorld) || (target.map && currentMap !== target.map)) {
				const mapName = target.map && targetWorld!.maps.has(target.map) ?
					targetWorld!.maps.get(target.map)!.name :
					targetWorld!.maps.entries().next().value[1].name;

				this.scheduledView = target;

				try {
					store.commit(MutationTypes.SET_CURRENT_MAP, {worldName: targetWorld!.name, mapName});
				} catch(e) {
					//Clear scheduled move if change fails
					console.warn(`Failed to handle map setView`, e);
					this.scheduledView = null;
				}
			} else {
				console.debug('Moving to', JSON.stringify(target));
				if(typeof target.zoom !== 'undefined') {
					this.leaflet!.setZoom(target.zoom, target.options as ZoomPanOptions);
				}

				if('min' in target.location) { // Bounds
					this.leaflet!.fitBounds(new LatLngBounds(
						store.state.currentMap?.locationToLatLng(target.location.min) as LatLng,
						store.state.currentMap?.locationToLatLng(target.location.max) as LatLng,
					), target.options);
				} else { // Location
					const location = store.state.currentMap?.locationToLatLng(target.location) as LatLng;
					this.leaflet!.panTo(location, target.options as PanOptions);
				}
			}
		},
		updateFollow(player: LiveAtlasPlayer, newFollow: boolean) {
			const store = useStore(),
				currentWorld = store.state.currentWorld;

			let map = undefined;

			if(player.hidden) {
				console.warn(`Cannot follow ${player.name}. Player is hidden from the map.`);
				return;
			}

			if(!currentWorld || currentWorld.name !== player.location.world || newFollow) {
				map = store.state.configuration.followMap;
			}

			this.setView({
				location: player.location,
				map,
				zoom: (newFollow) ? store.state.configuration.followZoom : undefined,
			});
		}
	}
})
</script>

<style lang="scss" scoped>
	@import '../scss/_mixins.scss';

	.map {
		width: 100%;
		height: 100%;
		background: transparent;
		z-index: 0;
		cursor: default;
		box-sizing: border-box;
		position: relative;

		&:focus:before {
			content: '';
			position: absolute;
			top: 0;
			right: 0;
			bottom: 0;
			left: 0;
			border: 0.2rem solid var(--outline-focus);
			display: block;
			z-index: 2000;
			pointer-events: none;
		}

		@include focus-reset {
			&:before {
				content: none;
			}
		}
	}
</style>
