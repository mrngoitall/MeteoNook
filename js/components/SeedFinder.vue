<template>
	<div>
		<p class='mt-3'>{{ $t('sIntro') }}</p>
		<h4 class='mt-4'>{{ $t('sTTypes') }}</h4>
		<div v-html="timeify($t('sTypes1'))"></div>
		<img class='float-md-right img-thumbnail wtExplainerImg mb-2' :src='explainerImg' title='Weather type example screenshots'>
		<div v-html="timeify($t('sTypes2'))"></div>
		<h4 class='mt-4'>{{ $t('sTStars') }}</h4>
		<div v-html="timeify($t('sStars'))"></div>
		<h4 class='mt-4'>{{ $t('sTTips') }}</h4>
		<div v-html="timeify($t('sTips'))"></div>

		<h4 class='mt-4'>{{ $t('tStep1') }}</h4>
		<b-form-radio-group class='mb-3' stacked id='seedFinderHemisphere' v-model.number='hemisphere'>
			<b-form-radio value='0'>{{ $t('tNorthernHemi') }}</b-form-radio>
			<b-form-radio value='1'>{{ $t('tSouthernHemi') }}</b-form-radio>
		</b-form-radio-group>

		<p>
			{{ $t('tResetBlurb') }}
			<b-button size='sm' variant='outline-secondary' @click='resetData'>
				{{ $t('tResetButton') }}
			</b-button>
		</p>

		<p>
			{{ $t('tExportBlurb') }}
			<b-button size='sm' variant='outline-secondary' @click='exportData'>
				{{ $t('tExportButton') }}
			</b-button>
			<b-modal :title="$t('deTitle')" hide-footer id='exportDataModal'>
				<b-form-textarea ref='exportedDataTextarea' :value='exportedData' readonly rows='6' @click='selectAllExportData'></b-form-textarea>
			</b-modal>
		</p>

		<div class='d-sm-flex'>
			<b-calendar
				class='align-self-start mr-3'
				min='2000-01-01' max='2060-12-31'
				label-help=''
				:locale='$i18n.locale'
				v-model='selectedDay'
				:date-info-fn='dateClass'
				value-as-date
				></b-calendar>
			<day-editor
				class='flex-grow-1'
				:day='getDay(selectedDay)'
				:hemisphere='hemisphere'
				@show-day="showDayModal"
			>
			</day-editor>
		</div>

		<h4 class='mt-4'>{{ $t('tStep2') }}</h4>
		<guess-worker-view
			:days='days'
			:hemisphere='hemisphere'
			@preview-seed='previewSeed'
			>
		</guess-worker-view>
	</div>
</template>

<script lang='ts'>
import { Vue, Component, Watch } from 'vue-property-decorator'
import DayEditor from './DayEditor.vue'
import GuessWorkerView from './GuessWorkerView.vue'
import {createDayInfo, DayInfo, DayType, ShowerType, Hemisphere, isDayNonEmpty, DayForecast} from '../model'
import {readStorageObject, readStorage, writeStorageObject, writeStorage, makeTime} from '../utils'
import { BFormTextarea } from 'bootstrap-vue'

export type SeedFinderDays = {[key: string]: DayInfo}

const sfDaysKey = 'meteonook_sf_days'
const sfHemisphereKey = 'meteonook_sf_hemisphere'
function loadSFDays(): SeedFinderDays {
	let obj = readStorageObject<SeedFinderDays>(sfDaysKey)
	if (obj !== null)
		return obj

	obj = readStorageObject<SeedFinderDays>('meteonook_data', (data: any) => data.version === 2)
	if (obj !== null) {
		delete obj.version
		return obj
	}

	return {}
}

function loadSFHemisphere(): Hemisphere {
	let hemi = readStorage(sfHemisphereKey, blob => (blob == 'n') ? Hemisphere.Northern : Hemisphere.Southern)
	if (hemi !== null)
		return hemi
	else
		return Hemisphere.Northern
}

function saveSFDays(days: SeedFinderDays) {
	writeStorageObject(sfDaysKey, days)
}
function saveSFHemisphere(hemisphere: Hemisphere) {
	writeStorage(sfHemisphereKey, (hemisphere == Hemisphere.Northern) ? 'n' : 's')
}



@Component({components: {DayEditor, GuessWorkerView}})
export default class SeedFinder extends Vue {
	selectedDay = new Date()
	days: SeedFinderDays = loadSFDays()
	hemisphere = loadSFHemisphere()

	$refs!: {
		exportedDataTextarea: BFormTextarea
	}

	getDay(date: Date) {
		const key = `${date.getFullYear()}-${date.getMonth()+1}-${date.getDate()}`
		if (this.days[key] === undefined)
			Vue.set(this.days, key, createDayInfo(date))
		return this.days[key]
	}
	
	resetData() {
		this.days = {}
	}

	timeify(str: string): string {
		// accessing time12 here forces this component to be
		// updated whenever time12 changes...
		const awfulHack = (this.$root as any).time12

		return str.replace(/[012]\d:\d\d:\d\d/g, time => {
			const [hS, mS, sS] = time.split(':')
			const h = parseInt(hS, 10), m = parseInt(mS, 10), s = parseInt(sS, 10)
			return this.$d(makeTime(h, m, s), 'timeHMS')
		}).replace(/[012]\d:\d\d/g, time => {
			const [hS, mS] = time.split(':')
			const h = parseInt(hS, 10), m = parseInt(mS, 10)
			return this.$d(makeTime(h, m), 'timeHM')
		}).replace(/[012]\dH/g, time => {
			const h = parseInt(time.slice(0, 2), 10)
			return this.$d(makeTime(h, 0), 'timeH')
		})
	}

	dateClass(ymd: string, date: Date): string {
		const key = `${date.getFullYear()}-${date.getMonth()+1}-${date.getDate()}`
		const day = this.days[key]
		if (day !== undefined && isDayNonEmpty(day))
			return 'table-info'
		return ''
	}

	@Watch('hemisphere')
	onHemisphereChanged() { saveSFHemisphere(this.hemisphere) }
	@Watch('days', {deep: true})
	onDaysChanged() { saveSFDays(this.days) }

	previewSeed(seed: number, multiFlag: boolean) {
		this.$emit('preview-seed', seed, this.hemisphere, multiFlag)
	}
	showDayModal(day: DayForecast) {
		this.$emit('show-day', day)
	}

	get explainerImg(): string {
		switch (this.$i18n.locale) {
			case 'es': return 'explainer-es.jpg'
			default:   return 'explainer-en.jpg'
		}
	}


	exportedData = ''
	exportData() {
		this.$bvModal.show('exportDataModal')
		this.exportedData = JSON.stringify({days: this.days, hemisphere: this.hemisphere})
	}
	selectAllExportData() {
		const textarea = this.$refs.exportedDataTextarea.$el as HTMLTextAreaElement
		textarea.setSelectionRange(0, textarea.value.length)
	}
}
</script>

<style>
.progress-bar { transition: none; }

.wtExplainerImg {
	width: 45%;
	max-width: 720px;
	margin-left: .5rem;
}
@media (max-width: 767.98px) {
	.wtExplainerImg {
		width: 80%; max-width: 400px;
		display: block;
		margin-left: auto;
		margin-right: auto;
	}
}
</style>
