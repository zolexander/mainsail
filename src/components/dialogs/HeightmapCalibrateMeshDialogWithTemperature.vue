<template>
    <v-dialog :value="show" persistent :max-width="400" @keydown.esc="closeDialog">
        <panel
            :title="$t('Heightmap.BedMeshCalibrate')"
            :icon="mdiGrid"
            card-class="heightmap-calibrate-dialog"
            :margin-bottom="false">
            <template #buttons>
                <v-btn icon tile @click="closeDialog">
                    <v-icon>{{ mdiCloseThick }}</v-icon>
                </v-btn>
            </template>
            <v-card-text>
                <v-text-field
                    ref="input"
                    v-model="name"
                    :label="$t('Heightmap.Name')"
                    required
                    :rules="rules"
                    @update:error="
                        (newVal) => {
                            isInvalidName = newVal
                        }
                    "
                    @keyup.enter="calibrateMesh" />
            </v-card-text>
            <v-card-actions>
                <v-spacer />
                <v-btn text @click="closeDialog">{{ $t('Heightmap.Abort') }}</v-btn>
                <v-btn :disabled="isInvalidName" color="primary" text @click="calibrateMesh">
                    {{ $t('Heightmap.Calibrate') }}
                </v-btn>
            </v-card-actions>
        </panel>
    </v-dialog>
</template>
<script lang="ts">
import { Component, Mixins, Prop, Watch } from 'vue-property-decorator'
import BaseMixin from '@/components/mixins/base'
import { mdiCloseThick, mdiGrid } from '@mdi/js'

@Component
export default class HeightmapRenameProfileDialogWithTemperature extends Mixins(BaseMixin) {
    mdiCloseThick = mdiCloseThick
    mdiGrid = mdiGrid

    @Prop({ type: Boolean, required: true }) show!: boolean

    $refs!: {
        input: HTMLInputElement
    }

    isInvalidName = false
    name = ''
    get cooldownGcode(): string {
        return this.$store.getters['gui/presets/getCooldownGcode']
    }

    rules = [
        (value: string) => !!value || this.$t('Heightmap.InvalidNameEmpty'),
        // eslint-disable-next-line no-control-regex
        (value: string) => value === value.replace(/[^\x00-\x7F]/g, '') || this.$t('Heightmap.InvalidNameAscii'),
    ]

    calibrateMesh(): void {
        const gcode = `BED_MESH_CALIBRATE PROFILE="${this.name}"`
        const preheat = "SET_HEATER_TEMPERATURE HEATER=heater_bed TARGET=60"
        this.$store.dispatch('server/addEvent',{message: 'G28',type: 'command'}).then(()=>{
            this.$socket.emit('printer.gcode.script', { script: 'G28' }, { loading: 'homeAll' })
            this.$store.dispatch('server/addEvent',{message: preheat,type: 'command'}).then(()=>{
                this.$socket.emit('printer.gcode.script', { script: preheat }, { loading: 'homeAll' })
                this.$store.dispatch('server/addEvent', {message:"M190 S60",type:'command'}).then(()=>{
                        this.$socket.emit('printer.gcode.script', { script: 'M190 S60' })
                        this.$store.dispatch('server/addEvent', { message: gcode, type: 'command' })
                        this.$socket.emit('printer.gcode.script', { script: gcode }, { loading: 'bedMeshCalibrate' })
                        this.$store.dispatch('server/addEvent',{message: 'G28',type: 'command'}).then(()=>{
                        this.$socket.emit('printer.gcode.script', { script: 'G28' }, { loading: 'homeAll' })
                        this.$store.dispatch('server/addEvent', { message: this.cooldownGcode, type: 'command' })
                        this.$socket.emit('printer.gcode.script', { script: this.cooldownGcode })
                        this.$store.dispatch('server/addEvent',{ message: 'G91', type: 'command'})
                        this.$socket.emit('printer.gcode.script',{ script: 'G91'})
                        this.$store.dispatch('server/addEvent',{ message: 'G1 Z+100 F1500', type: 'command'})
                        this.$socket.emit('printer.gcode.script',{ script: 'G1 Z+100 F1500'})
                    });
                });
            });
        });
        this.closeDialog()
    }

    closeDialog() {
        this.$emit('close')
    }
    @Watch('show')
    showChanged() {
        if (this.show) {
            this.name = 'default'

            this.$nextTick(() => {
                setTimeout(() => {
                    this.$refs.input?.focus()
                }, 100)
            })
        }
    }
}
</script>
