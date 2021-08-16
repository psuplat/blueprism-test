<template>
  <Menubar :model="menu_items" class="p-p-3" id="main-menu-bar">
    <template #start>
      <img alt="logo" src="@/assets/logo-blue.svg" class="p-mr-2" />
    </template>
  </Menubar>
  <h1 class="p-mt-3 p-p-3">Schedules</h1>
  <ProgressSpinner class="p-mt-6 p-d-flex p-jc-center" v-if="loading" />
  <div class="p-mt-3 p-p-3" v-if="!loading">
    <div class="p-grid container">
      <div class="p-sm-12 p-md-3 p-lg-3 cards">
          <div class="p-d-flex p-jc-end p-mb-3">
            <div class="p-field-checkbox">
              <label class="p-pr-1" for="retired">Show retired</label>
              <Checkbox v-model="show_retired" id="retired" :binary="true" />
            </div>
          </div>
          <ScrollPanel class="scrollable p-pt-3 p-pr-3">
            <template v-for="(schedule, sidx) in schedules" :key="sidx" >
              <Card v-if="!schedule.isRetired || (schedule.isRetired && show_retired)" class="p-shadow-10 p-mb-4" @click="getDetails(schedule.id)">
                  <template #title>
                      {{schedule.name}}
                  </template>
                  <template #subtitle>
                      {{schedule.description}}
                  </template>
                  <template #content>
                      <p v-tooltip.top="'start date'"><i class="pi pi-calendar"></i> {{schedule.startDate}}</p>
                      <p v-tooltip.top="'end date'"><i class="pi pi-calendar"></i> {{schedule.endDate}}</p>
                      <p v-tooltip.top="'number of taks'"><i class="pi pi-cog"></i> {{schedule.tasksCount}}</p>
                  </template>
                  <template #footer >
                      <Button class="p-button-success p-button-outlined p-button-sm" icon="pi pi-eye" label="Restore" v-if="schedule.isRetired" @click.stop="restore(sidx, schedule.id)" />
                      <Button class="p-button-secondary p-button-outlined p-button-sm" icon="pi pi-eye-slash" label="Retire" v-if="!schedule.isRetired" @click.stop="retire(sidx, schedule.id)" />
                  </template>
              </Card>
            </template>
          </ScrollPanel>
      </div>
      <div class="p-col-9" v-if="!isMobile()">
        <DataTable :value="details" v-if="details" sortMode="multiple" >
          <Column field="serverName" header="Server" :sortable="true"></Column>
          <Column field="startTime" header="Start Time" :sortable="true"></Column>
          <Column field="endTime" header="End Time" :sortable="true"></Column>
          <Column field="status" header="Status" :sortable="true">
            <template #body="slotProps">
              <Tag style="width:100%" v-if="slotProps.data.status.toLowerCase() == 'completed'" value="Completed" icon="pi pi-check" severity="success"></Tag>
              <Tag style="width:100%" v-else-if="slotProps.data.status.toLowerCase() == 'pending'" value="Pending" icon="pi pi-clock" severity="info"></Tag>
              <Tag style="width:100%" v-else-if="slotProps.data.status.toLowerCase() == 'running'" value="Running" icon="pi pi-chevron-right"></Tag>
              <Tag style="width:100%" v-else-if="slotProps.data.status.toLowerCase() == 'exception'" value="Exception" icon="pi pi-exclamation-circle" severity="danger"></Tag>
              <Tag style="width:100%" v-else-if="slotProps.data.status.toLowerCase() == 'terminated'" value="Terminated" icon="pi pi-times" severity="warning"></Tag>
              <span v-else>{{slotProps.data.status}}</span>
            </template>
          </Column>
        </DataTable>
        <p class="p-text-center" v-if="!isMobile() && !details">select schedule to view its details</p>
      </div>
    </div>
  </div>
  <Dialog  v-if="isMobile()" header="Schedule Details" v-model:visible="mobile_dialog" position="bottom">
    <DataTable :value="details" v-if="details" sortMode="multiple"  responsiveLayout="stack">
      <Column field="serverName" header="Server" :sortable="true"></Column>
      <Column field="startTime" header="Start Time" :sortable="true"></Column>
      <Column field="endTime" header="End Time" :sortable="true"></Column>
      <Column field="status" header="Status" :sortable="true">
        <template #body="slotProps">
          <Tag v-if="slotProps.data.status.toLowerCase() == 'completed'" value="Completed" icon="pi pi-check" severity="success"></Tag>
          <Tag v-else-if="slotProps.data.status.toLowerCase() == 'pending'" value="Pending" icon="pi pi-clock" severity="info"></Tag>
          <Tag v-else-if="slotProps.data.status.toLowerCase() == 'running'" value="Running" icon="pi pi-chevron-right"></Tag>
          <Tag v-else-if="slotProps.data.status.toLowerCase() == 'exception'" value="Exception" icon="pi pi-exclamation-circle" severity="danger"></Tag>
          <Tag v-else-if="slotProps.data.status.toLowerCase() == 'terminated'" value="Terminated" icon="pi pi-times" severity="warning"></Tag>
          <span v-else>{{slotProps.data.status}}</span>
        </template>
      </Column>
    </DataTable>
  </Dialog>
  <Toast position="top-right" :baseZIndex="2147483647" />
</template>
<style>
@font-face {
  font-family: "Raleway";
  src: local("Raleway"),
    url(./assets/fonts/Raleway-Light.ttf) format("truetype");
}
body {
  margin: 0;
  font-family: "Raleway", Helvetica, Arial;
}
.scrollable {
  width: 100%;
  height: 70vh;
}
</style>
<script>
import axios from "axios";
import Menubar from "primevue/menubar";
import ProgressSpinner from 'primevue/progressspinner';
import Checkbox from 'primevue/checkbox';
import Card from 'primevue/card';
import Button from "primevue/button";
import DataTable from 'primevue/datatable';
import Column from 'primevue/column';
import Tag from 'primevue/tag';
import Dialog from 'primevue/dialog';
import Toast from "primevue/toast";
import ScrollPanel from 'primevue/scrollpanel';

export default {
  name: "Home",
  components: {
    Menubar,
    ProgressSpinner,
    Checkbox,
    Card,
    Button,
    DataTable,
    Column,
    Tag,
    Dialog,
    Toast,
    ScrollPanel
  },
  data() {
    return {
      menu_items: [],
      schedules: [],
      loading: true,
      show_retired:false,
      details: false,
      mobile_dialog: false
    };
  },
  computed: {},
  mounted() {
    this.getSchedules();
  },
  methods: {
    isMobile() {
      if (
        /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(
          navigator.userAgent
        )
      ) {
        return true;
      } else {
        return false;
      }
    },
    getSchedules() {
      this.loading.true;
      axios({
        method: 'GET',
        url: 'http://localhost:3000/schedules',
      }).then(
        (result) => {
          this.schedules = result.data;
          this.loading = false;
        },
        () => {}
      );
    },
    getDetails(schedule_id) {
      this.details = [];
      axios({
        method: 'GET',
        url: 'http://localhost:3000/scheduleLogs?scheduleId='+schedule_id,
      }).then(
        (result) => {
          this.details = result.data;
          if(this.isMobile) {
            this.mobile_dialog = true;
          }
        },
        () => {}
      );
    },
    retire(idx, schedule_id) {
      let d = this.schedules[idx];
      d.isRetired = true;
      axios({
        method: 'PUT',
        url: 'http://localhost:3000/schedules/'+schedule_id,
        data: JSON.stringify(d),
        headers: {
          'Content-Type': 'application/json'
        }
      }).then(
        () => {
          //instead of calling the api again, we will simply change the property on existing data
          this.schedules[idx].isRetired = true;

          //also, if the currently show table is from the retired schedule, clear it
          console.log('length:', this.details.length)
          
          if(this.details.length > 0 && this.details[0].scheduleId == schedule_id) {
            this.details = false;
          }
          this.$toast.add({
            severity: "success",
            summary: "Success",
            detail: "Schedule retired",
            life: 3000,
          });
        },
        () => {
          this.$toast.add({
              severity: "error",
              summary: "Error",
              detail: "Error occured when trying to retire the selected schedule",
              life: 3000,
            });
        }
      );
    },
    restore(idx, schedule_id) {
      let d = this.schedules[idx];
      d.isRetired = false;
      axios({
        method: 'PUT',
        url: 'http://localhost:3000/schedules/'+schedule_id,
        data: JSON.stringify(d),
        headers: {
          'Content-Type': 'application/json'
        }
      }).then(
        () => {
          //instead of calling the api again, we will simply change the property on existing data
          this.schedules[idx].isRetired = false;

          this.$toast.add({
            severity: "success",
            summary: "Success",
            detail: "Schedule restored",
            life: 3000,
          });
        },
        () => {
          this.$toast.add({
              severity: "error",
              summary: "Error",
              detail: "Error occured when trying to restore the selected schedule",
              life: 3000,
            });
        }
      );
    },
  },
};
</script>
