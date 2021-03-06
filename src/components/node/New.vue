<template>
	<div class="container">
		<div class="row">
			<div class="col-lg-12">
				<div class="page-header">
					<h1>{{ $t('navigation.nodes') }}</h1>
				</div>
			</div>
		</div>

		<div class="row">
			<div class="col-lg-9">
				<form class="form-horizontal well">
					<fieldset>
						<legend v-if="!this.$route.params.id">{{ $t('nodes.new.newnode') }}</legend>
						<legend v-if="this.$route.params.id">{{ $t('nodes.new.editnode') }}</legend>
						<div class="form-group">
							<label class="col-lg-2 control-label">{{ $t('nodes.new.name.title') }}</label>
							<div class="col-lg-10">
								<input type="text" v-model="form.name" class="form-control">
								<span v-if="editing" v-html="this.$i18n.t('general.duplication')" class="help-block"></span>
							</div>
						</div>
						<div class="form-group">
							<label class="col-lg-2 control-label">{{ $t('general.latlong.title') }}</label>
							<div class="col-lg-3">
								<input type="number" lang="en-150" v-model.number="form.latitude.value" min="0" max="90" placeholder="0 - 90" class="form-control">
							</div>
							<div class="col-lg-2">
								<select class="form-control" v-model.number="form.latitude.orientation">
									<option value="1">{{ $t('general.latlong.north') }}</option>
									<option value="-1">{{ $t('general.latlong.south') }}</option>
								</select>
							</div>
							<div class="col-lg-3">
								<input type="number" lang="en-150" v-model.number="form.longitude.value" min="0" max="180" placeholder="0 - 180" class="form-control">
							</div>
							<div class="col-lg-2">
								<select class="form-control" v-model.number="form.longitude.orientation">
									<option value="1">{{ $t('general.latlong.east') }}</option>
									<option value="-1">{{ $t('general.latlong.west') }}</option>
								</select>
							</div>
						</div>
						<div class="form-group">
							<label class="col-lg-2 control-label">{{ $t('general.latlong.picker') }}</label>
							<div class="col-lg-10">
								<v-map :zoom="map.zoom" :center="map.center" v-on:l-click="mapClicked" style="height: 30em">
									<v-tilelayer :url="map.url" :attribution="map.attribution"></v-tilelayer>
									<v-marker :lat-lng="map.marker"></v-marker>
								</v-map>
							</div>
						</div>
						<div class="form-group">
							<label class="col-lg-2 control-label">{{ $t('general.owner') }}</label>
							<div class="col-lg-10">
								<multiselect v-model="form.owners" :options="formData.users" :multiple="true" :close-on-select="false" :hide-selected="true" :clear-on-select="true" placeholder="Type to search" label="name" track-by="name"></multiselect>
							</div>
						</div>
						<div class="form-group">
							<label class="col-lg-2 control-label">{{ $t('nodes.new.status.title') }}</label>
							<div class="col-lg-10">
								<select class="form-control" v-model="form.status">
									<option value="ONLINE">{{ $t('nodes.new.status.online') }}</option>
									<option value="SUSPENDED">{{ $t('nodes.new.status.suspended') }}</option>
								</select>
							</div>
						</div>
						<div class="form-group">
							<div class="col-lg-10 col-lg-offset-2">
								<button type="submit" @click="submitForm" class="btn btn-primary">{{ $t('general.submit') }}</button>
								<router-link to="/nodes"><button class="btn btn-default">{{ $t('general.abort') }}</button></router-link>
							</div>
						</div>
					</fieldset>
				</form>
			</div>
			<div class="col-lg-3">
				<h2>{{ $t('general.information') }}</h2>
				<p v-html="$t('nodes.new.information.help')"></p>
			</div>
		</div>
	</div>
</template>

<script>
	import Vue from 'vue';
	import Vue2Leaflet from 'vue2-leaflet';
	import 'leaflet/dist/leaflet.css';
	Vue.component('v-map', Vue2Leaflet.Map);
	Vue.component('v-tilelayer', Vue2Leaflet.TileLayer);
	Vue.component('v-marker', Vue2Leaflet.Marker);

	export default {
		created() {
			this.$http.get('users').then(response => {
				response.body.forEach(user => {
					this.formData.users.push({name: user.name});
				});
			});

			// load data of given id
			if (this.$route.params.id) {
				this.$http.get('nodes/' + this.$route.params.id).then(response => {
					this.editing = true;

					let ownerNames = [];
					response.body.ownerNames.forEach(owner => {
						ownerNames.push({name: owner});
					});

					this.form.name = response.body.name;
					this.form.latitude.value = Math.abs(response.body.latitude);
					this.form.latitude.orientation = (response.body.latitude >= 0 ? 1 : -1);
					this.form.longitude.value = Math.abs(response.body.longitude);
					this.form.longitude.orientation = (response.body.longitude >= 0 ? 1 : -1);
					this.form.owners = ownerNames;
					this.form.status = response.body.status;

					// center location picker to node
					this.map.center = [this.form.latitude.value * this.form.latitude.orientation, this.form.longitude.value * this.form.longitude.orientation];
				}, response => {
					this.$router.push('/nodes');
				});
			}
		},
		data() {
			return {
				editing: false,
				form: {
					name: '',
					latitude: {
						value: 0,
						orientation: 1
					},
					longitude: {
						value: 0,
						orientation: 1
					},
					owners: [],
					status: 'ONLINE'
				},
				formData: {
					users: []
				},
				map: {
					zoom: this.$store.getters.map.zoom,
					center: this.$store.getters.map.center,
					attribution: '&copy; <a href="https://www.openstreetmap.org/copyright" target="_blank">OpenStreetMap</a>',
					url: this.$store.getters.url.map,
					marker: {
						lat: 0,
						lng: 0
					}
				}
			};
		},
		methods: {
			submitForm(event) {
				event.preventDefault();

				// check for input in all fields
				if (!this.$helpers.checkForInput(this, this.form)) {
					return false;
				}

				// check if location is valid
				if (this.form.latitude.value === '' || this.form.longitude.value === '') {
					this.$swal({
						title: 'Invalid location',
						html: 'Please check the given location and make sure to use your locale\'s decimal separator (e.g. <code>,</code> in German or <code>.</code> in US-English).',
						type: 'error'
					});
					return false;
				}

				let ownerNames = [];
				this.form.owners.forEach(owner => {
					ownerNames.push(owner.name);
				});

				let body = {
					latitude: this.form.latitude.value * this.form.latitude.orientation,
					longitude: this.form.longitude.value * this.form.longitude.orientation,
					ownerNames: ownerNames,
					status: this.form.status
				};

				this.$helpers.checkForOverwritingAndSend(this, this.$route.params.id, 'nodes/' + this.form.name, body, '/nodes');
			},
			mapClicked(e) {
				this.map.marker = {
					lat: e.latlng.lat,
					lng: e.latlng.lng
				};

				// set position in form
				this.form.latitude.value = Math.abs(e.latlng.lat).toFixed(6);
				this.form.latitude.orientation = (e.latlng.lat >= 0 ? 1 : -1);
				this.form.longitude.value = Math.abs(e.latlng.lng).toFixed(6);
				this.form.longitude.orientation = (e.latlng.lng >= 0 ? 1 : -1);
			},
			locationChanged() {
				this.map.marker = {
					lat: this.form.latitude.value * this.form.latitude.orientation,
					lng: this.form.longitude.value * this.form.longitude.orientation
				};
			}
		},
		watch: {
			'form.latitude.value': function() {
				this.locationChanged();
			},
			'form.latitude.orientation': function() {
				this.locationChanged();
			},
			'form.longitude.value': function() {
				this.locationChanged();
			},
			'form.longitude.orientation': function() {
				this.locationChanged();
			}
		}
	};
</script>

<style scoped>

</style>
