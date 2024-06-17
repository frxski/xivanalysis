import {t} from '@lingui/macro'
import {Trans} from '@lingui/react'
import Color from 'color'
import {Event, Events} from 'event'
import {filter, oneOf} from 'parser/core/filter'
import {dependency} from 'parser/core/Injectable'
import {CounterGauge, Gauge as CoreGauge, TimerGauge} from 'parser/core/modules/Gauge'
import Suggestions, {SEVERITY, TieredSuggestion} from 'parser/core/modules/Suggestions'
import React from 'react'

const FADE_AMOUNT = 0.25
const RICOCHET_COLOR = Color('#9767C7').fade(FADE_AMOUNT)
const GAUSS_ROUND_COLOR = Color('#41BAEC').fade(FADE_AMOUNT)
const GAUSS_ROUND_TIME_REQUIRED = 30000
const RICOCHET_TIME_REQUIRED = 30000
const TIMER_FADE = 0.75
const HEAT_BLAST_REFUND = 15000

const OVERCAP_SEVERITY = {
	GAUSS_ROUND: {
		2: SEVERITY.MINOR,
		3: SEVERITY.MEDIUM,
		4: SEVERITY.MAJOR,
	},
	RICHOCHET: {
		2: SEVERITY.MINOR,
		3: SEVERITY.MEDIUM,
		4: SEVERITY.MAJOR,
	},
}

interface GaugeModifier {
	event: 'action'
	type: 'generate' | 'spend'
	amount: number
}

type GaugeMap = Map<number, GaugeModifier>

export class OGCDCharges extends CoreGauge {
	static override handle = 'ogcdcharges'
	static override title = t('mch.ogcdcharges.title')`Gauss Round / Ricochet Charges`

	@dependency private suggestions!: Suggestions

	private gaussRound = this.add(new CounterGauge({
		graph: {
			handle: 'gaussRound',
			label: <Trans id="mch.ogcdcharges.resource.gaussround">Gauss Round</Trans>,
			color: GAUSS_ROUND_COLOR,
			collapse: true,
		},
		maximum: 3,
		initialValue: 3,
	}))

	private gaussRoundTimer = this.add(new TimerGauge({
		maximum: GAUSS_ROUND_TIME_REQUIRED,
		onExpiration: this.onCompleteGaussRoundTimer.bind(this),
		graph: {
			handle: 'gaussRound',
			label: <Trans id="mch.ogcdcharges.resource.gaussround.timer">Gauss Round Timer</Trans>,
			color: GAUSS_ROUND_COLOR.fade(TIMER_FADE),
		},
	}))

	private ricochet = this.add(new CounterGauge({
		graph: {
			handle: 'ricochet',
			label: <Trans id="mch.ogcdcharges.resource.ricochet">Richochet</Trans>,
			color: RICOCHET_COLOR,
			collapse: true,
		},
		maximum: 3,
		initialValue: 3,
	}))

	private ricochetTimer = this.add(new TimerGauge({
		maximum: RICOCHET_TIME_REQUIRED,
		onExpiration: this.onCompleteRicochetTimer.bind(this),
		graph: {
			handle: 'ricochet',
			label: <Trans id="mch.ogcdcharges.resource.ricochet.timer">Ricochet Timer</Trans>,
			color: RICOCHET_COLOR.fade(TIMER_FADE),
		},
	}))

	private gaussRoundModifiers: GaugeMap = new Map([
		[this.data.actions.GAUSS_ROUND.id, {event: 'action', type: 'spend', amount: 1}],
	])

	private ricochetModifiers: GaugeMap = new Map([
		[this.data.actions.RICOCHET.id, {event: 'action', type: 'spend', amount: 1}],
	])

	private addGaugeHooks(gauge: CounterGauge, modifiers: GaugeMap) {
		const castActions = []

		for (const action of modifiers.keys()) {
			castActions.push(action)
		}

		const baseFilter = filter<Event>().source(this.parser.actor.id)

		const actionFilter = baseFilter
			.type(oneOf(['action']))
			.action(oneOf(castActions))

		this.addEventHook(actionFilter, this.onAction(gauge, modifiers))
	}

	private onCompleteTimer(gauge: CounterGauge, timer: TimerGauge) {
		gauge.generate(1)
		timer.reset()
		if (!gauge.capped) {
			timer.start()
		}
	}

	private onCompleteGaussRoundTimer() {
		this.onCompleteTimer(this.gaussRound, this.gaussRoundTimer)
	}

	private onCompleteRicochetTimer() {
		this.onCompleteTimer(this.ricochet, this.ricochetTimer)
	}

	override initialise() {
		super.initialise()

		this.addGaugeHooks(this.gaussRound, this.gaussRoundModifiers)
		this.addGaugeHooks(this.ricochet, this.ricochetModifiers)

		const heatBlastFilter = filter<Event>()
			.source(this.parser.actor.id)
			.type('action')
			.action(this.data.actions.HEAT_BLAST.id)

		this.addEventHook(heatBlastFilter, this.onHeatBlast)
		this.addEventHook('complete', this.onComplete)
	}

	private onAction(gauge: CounterGauge, modifiers: GaugeMap) {
		return (event: Events['action' | 'combo']) => {
			const modifier = modifiers.get(event.action)

			if (modifier && modifier.event === event.type) {
				if (modifier.type === 'spend' && gauge.capped) {
					this.gaussRoundTimer.start()
					this.ricochetTimer.start()
				}
				gauge.spend(modifier.amount)
			}
		}
	}

	private refundPartialCooldown(gauge: CounterGauge, timer: TimerGauge, amount: number) {
		if (gauge.capped) {
			return
		}

		if (timer.remaining < amount) {
			const remainder = timer.remaining
			this.onCompleteTimer(gauge, timer)
			if (!gauge.capped) {
				timer.set(timer.remaining - amount + remainder)
			}
			return
		}

		timer.set(timer.remaining - amount)
	}

	private onHeatBlast() {
		this.refundPartialCooldown(this.gaussRound, this.gaussRoundTimer, HEAT_BLAST_REFUND)
		this.refundPartialCooldown(this.ricochet, this.ricochetTimer, HEAT_BLAST_REFUND)
	}

	private onComplete() {
		this.suggestions.add(new TieredSuggestion({
			icon: this.data.actions.GAUSS_ROUND.icon,
			content: <Trans id="mch.gauge.suggestions.gauss_round-waste.content">
				Try not to go into Hypercharge windows with multiple Gauss Round/Ricochet stacks, as it makes overcapping extremely easy.
			</Trans>,
			tiers: OVERCAP_SEVERITY.GAUSS_ROUND,
			value: this.gaussRound.overCap,
			why: <Trans id="mch.gauge.suggestions.gauss_round-waste.why">
				You lost {this.gaussRound.overCap} Gauss Round uses due to leaving it off cooldown.
			</Trans>,
		}))

		this.suggestions.add(new TieredSuggestion({
			icon: this.data.actions.RICOCHET.icon,
			content: <Trans id="mch.gauge.suggestions.ricochet.content">
				Try not to go into Hypercharge windows with multiple Gauss Round/Ricochet stacks, as it makes overcapping extremely easy.
			</Trans>,
			tiers: OVERCAP_SEVERITY.RICHOCHET,
			value: this.ricochet.overCap,
			why: <Trans id="mch.gauge.suggestions.ricochet-waste.why">
				You lost {this.ricochet.overCap} Ricochet uses due to leaving it off cooldown.
			</Trans>,
		}))
	}
}
