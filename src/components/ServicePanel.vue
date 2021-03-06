<template>
	<DataTable ref="table" :dataSource="dataSource" :columns="columns" id="ServicePanel">
		<template slot="toolbar" slot-scope="p">
			<button title="Refresh services" @click="updateData()"><i class="fas fa-sync-alt"></i></button> <!-- ToDo: Should be done automatically later -->
		</template>
		<template slot="actions" slot-scope="p">
			<button title="View on map" @click="viewService(p.row)"><i class="fas fa-map"></i></button>
			<button title="Details" @click="serviceInfo(p.row[p.col.id])" v-show="openEO.Capabilities.serviceInfo()"><i class="fas fa-info"></i></button>
			<button title="Edit" @click="editService(p.row[p.col.id])" v-show="openEO.Capabilities.updateService()"><i class="fas fa-edit"></i></button>
			<button title="Delete" @click="deleteService(p.row[p.col.id])" v-show="openEO.Capabilities.deleteService()"><i class="fas fa-trash"></i></button>
		</template>
	</DataTable>
</template>

<script>
import EventBus from '../eventbus.js';
import DataTable from './DataTable.vue';

export default {
	name: 'ServicePanel',
	props: ['openEO','userId'],
	components: {
		DataTable
	},
	data() {
		return {
			columns: {
				service_id: {
					name: 'ID',
					primaryKey: true
				},
				service_type: {
					name: 'Type'
				},
				job_id: {
					name: 'Job'
				},
				actions: {
					name: 'Actions',
					filterable: false,
					id: 'service_id'
				}
			}
		};
	},
	created() {
		EventBus.$on('serverChanged', this.updateData);
		EventBus.$on('serviceCreated', this.serviceCreated);
	},
	watch: { 
		userId(newVal, oldVal) {
			if (newVal !== null) {
				this.updateData();
			}
		}
	},
	methods: {
		dataSource() {
			let users = this.openEO.Users.getObject(this.userId);
			return users.getServices();
		},
		updateData() {
			if (!this.$refs.table || !this.openEO.Capabilities.createService()) {
				return;
			}
			else if (typeof this.userId !== 'string' && typeof this.userId !== 'number') {
				this.$refs.table.setNoData(401);  // "please authenticate"
			}
			else {
				this.$refs.table.retrieveData();
			}
		},
		serviceCreated(data) {
			this.$refs.table.addData(data);

			var options = {
				buttons: []
			};
			options.buttons.push({text: 'View', action: () => this.viewService(data)});
			if (this.openEO.Capabilities.serviceInfo()) {
				options.buttons.push({text: 'Details', action: () => this.serviceInfo(data.job_id)});
			}
			if (this.openEO.Capabilities.updateService()) {
				options.buttons.push({text: 'Edit', action: () => this.editService(data.job_id)});
			}
			if (this.openEO.Capabilities.deleteService()) {
				options.buttons.push({text: 'Delete', action: () => this.deleteService(data.job_id)});
			}
			this.$snotify.confirm('Service created!', null, options);
		},
		serviceInfo(id) {
			var serviceApi = this.openEO.Services.getObject(id);
			serviceApi.get()
				.then(data => {
					EventBus.$emit('showModal', 'Service: ' + id, data);
				}).catch(error => {
					this.$utils.error(this, 'Sorry, could not load service information.');
				});
		},
		editService(id) {
			var serviceApi = this.openEO.Services.getObject(id);
			// serviceApi.modify(null, null, {});
			this.$utils.error(this, 'Not implemented');
			// ToDo: Request what user wants to change
		},
		deleteService(id) {
			var serviceApi = this.openEO.Services.getObject(id);
			serviceApi.delete()
				.then(data => {
					this.$refs.table.removeData(id);
					EventBus.$emit('removeWebService', id);
				})
				.catch(error => {
					this.$utils.error(this, 'Sorry, could not delete service.');
				});
		},
		viewService(service) {
			this.$utils.info(this, 'Requesting tiles from server. Please wait...');
			EventBus.$emit('viewWebService', service);
			EventBus.$emit('showMapViewer');
		}
	}
}
</script>

<style>
#ServicePanel .service_id {
	width: 35%;
}
#ServicePanel .service_type {
	width: 10%;
}
#ServicePanel .job_id {
	width: 35%;
}
</style>