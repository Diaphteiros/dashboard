<!--
Copyright (c) 2019 by SAP SE or an SAP affiliate company. All rights reserved. This file is licensed under the Apache Software License, v. 2 except as noted otherwise in the LICENSE file

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

<template>
  <transition-group name="list">
    <v-layout row v-for="(worker, index) in internalWorkers" :key="worker.id" class="list-item pt-2" :class="{ 'grey lighten-5': index % 2 }">
      <worker-input-generic
        ref="workerInput"
        :worker="worker"
        :workers="internalWorkers"
        :cloudProfileName="cloudProfileName"
        :zones="zones"
        @valid="onWorkerValid">
        <v-btn v-show="index>0 || internalWorkers.length>1"
          small
          slot="action"
          outline
          icon
          class="grey--text lighten-2"
          @click.native.stop="onRemoveWorker(index)">
          <v-icon>mdi-close</v-icon>
        </v-btn>
      </worker-input-generic>
    </v-layout>
    <v-layout row key="addWorker" class="list-item pt-2">
      <v-flex>
        <v-btn
          :disabled="!(machineTypes.length > 0)"
          small
          @click="addWorker"
          outline
          fab
          icon
          class="cyan darken-2 ml-1">
          <v-icon class="cyan--text text--darken-2">add</v-icon>
        </v-btn>
        <v-btn
          :disabled="!(machineTypes.length > 0)"
          @click="addWorker"
          flat
          class="cyan--text text--darken-2">
          Add Worker Group
        </v-btn>
      </v-flex>
    </v-layout>
  </transition-group>
</template>

<script>
import WorkerInputGeneric from '@/components/ShootWorkers/WorkerInputGeneric'
import { mapGetters } from 'vuex'
import { shortRandomString } from '@/utils'
import forEach from 'lodash/forEach'
import get from 'lodash/get'
import find from 'lodash/find'
import head from 'lodash/head'
import omit from 'lodash/omit'
import assign from 'lodash/assign'
const uuidv4 = require('uuid/v4')

export default {
  name: 'manage-workers',
  components: {
    WorkerInputGeneric
  },
  props: {
    userInterActionBus: {
      type: Object
    }
  },
  data () {
    return {
      internalWorkers: undefined,
      valid: false,
      cloudProfileName: undefined,
      zones: undefined,
      workers: undefined
    }
  },
  computed: {
    ...mapGetters([
      'cloudProfileByName',
      'machineTypesByCloudProfileNameAndZones',
      'volumeTypesByCloudProfileNameAndZones',
      'defaultMachineImageForCloudProfileName'
    ]),
    machineTypes () {
      return this.machineTypesByCloudProfileNameAndZones({ cloudProfileName: this.cloudProfileName, zones: this.zones })
    },
    volumeTypes () {
      return this.volumeTypesByCloudProfileNameAndZones({ cloudProfileName: this.cloudProfileName, zones: this.zones })
    },
    defaultMachineImage () {
      return this.defaultMachineImageForCloudProfileName(this.cloudProfileName)
    }
  },
  methods: {
    setInternalWorkers (workers) {
      this.internalWorkers = []
      if (workers) {
        forEach(workers, worker => {
          const id = uuidv4()
          const internalWorker = assign({}, worker, { id })
          this.internalWorkers.push(internalWorker)
        })
      }
      this.validateInput()
    },
    addWorker () {
      const id = uuidv4()
      const name = `worker-${shortRandomString(5)}`
      const volumeType = get(head(this.volumeTypes), 'name')
      const volumeSize = volumeType ? '50Gi' : undefined
      const machineType = get(head(this.machineTypes), 'name')
      const machineImage = this.defaultMachineImage
      this.internalWorkers.push({
        id,
        name,
        machineType,
        volumeType,
        volumeSize,
        autoScalerMin: 1,
        autoScalerMax: 2,
        maxSurge: 1,
        machineImage
      })
      this.validateInput()
    },
    onRemoveWorker (index) {
      this.internalWorkers.splice(index, 1)
      this.$nextTick(() => {
        // Need to evaluate the other components as well, as valid state may depend on each other (e.g. duplicate name)
        // Lack of doing so can lead to worker valid state != true even if conflict has been resolved, if input happens
        // on component which did not report valid = false in the first place
        forEach(this.$refs.workerInput, workerInput => {
          workerInput.validateInput()
        })
        this.validateInput()
      })
    },
    setDefaultWorker () {
      this.internalWorkers = []
      this.addWorker()
    },
    onWorkerValid ({ valid, id }) {
      const worker = find(this.internalWorkers, { id })
      const wasValid = worker.valid
      worker.valid = valid
      if (valid !== wasValid) {
        // Need to evaluate the other components as well, as valid state may depend on each other (e.g. duplicate name)
        // Lack of doing so can lead to worker valid state != true even if conflict has been resolved, if input happens
        // on component which did not report valid = false in the first place
        forEach(this.$refs.workerInput, workerInput => {
          workerInput.validateInput()
        })
      }
      this.validateInput()
    },
    getWorkers () {
      const workers = []
      forEach(this.internalWorkers, internalWorker => {
        const worker = omit(internalWorker, ['id', 'valid'])
        workers.push(worker)
      })
      return workers
    },
    validateInput () {
      let valid = true
      forEach(this.internalWorkers, worker => {
        if (!worker.valid) {
          valid = false
        }
      })

      this.valid = valid
      this.$emit('valid', this.valid)
    },
    setWorkersData ({ workers, cloudProfileName, zones }) {
      this.cloudProfileName = cloudProfileName
      this.zones = zones
      this.setInternalWorkers(workers)
    }
  },
  mounted () {
    if (this.userInterActionBus) {
      this.userInterActionBus.on('updateCloudProfileName', cloudProfileName => {
        this.cloudProfileName = cloudProfileName
        this.$nextTick(() => {
          // set worker when all props (including zone) have been updated
          this.setDefaultWorker()
        })
      })
      this.userInterActionBus.on('updateZones', zones => {
        this.zones = zones
      })
    }
  }
}
</script>
