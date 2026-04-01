<!--

                  This file is part of the Folding@home Client.

          The fah-client runs Folding@home protein folding simulations.
                    Copyright (c) 2001-2026, foldingathome.org
                               All rights reserved.

       This program is free software; you can redistribute it and/or modify
       it under the terms of the GNU General Public License as published by
        the Free Software Foundation; either version 3 of the License, or
                       (at your option) any later version.

         This program is distributed in the hope that it will be useful,
          but WITHOUT ANY WARRANTY; without even the implied warranty of
          MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
                   GNU General Public License for more details.

     You should have received a copy of the GNU General Public License along
     with this program; if not, write to the Free Software Foundation, Inc.,
           51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.

                  For information regarding this software email:
                                 Joseph Coffland
                          joseph@cauldrondevelopment.com

-->

<script>
import Unit from './unit.js';

function cmp_wus(a, b) {
	return (
		new Date(b.assign.time).getTime() - new Date(a.assign.time).getTime()
	);
}

function unique_values(l, key) {
	l = l.reduce((set, o) => {
		set[o[key]] = 1;
		return set;
	}, {});
	return Object.keys(l).sort();
}

function to_days(ts) {
	return new Date(ts).getTime() / (24 * 60 * 60 * 1000);
}

function array_reduce_num(a, key, f) {
	return a.reduce((min, o) => {
		let v = o[key];
		if (isFinite(v) && (min == undefined || f(v, min))) return v;
		return min;
	}, undefined);
}

function array_min(a, key) {
	return array_reduce_num(a, key, (a, b) => a < b);
}
function array_max(a, key) {
	return array_reduce_num(a, key, (a, b) => b < a);
}

function array_avg(a, key) {
	let sum = a.reduce((sum, o) => {
		let v = o[key];
		return isFinite(v) ? sum + v : sum;
	}, 0);

	let count = a.reduce((count, o) => count + (isFinite(o[key]) ? 1 : 0), 0);

	return sum / count;
}

function filter_unit(filter, unit, key1, key2) {
	return filter[key1] == 'Any' || filter[key1] == unit[key2 || key1];
}

function format_tpf(tpf) {
	return tpf ? this.$util.time_interval(tpf) : '???';
}
function format_ppd(ppd) {
	return isFinite(ppd) ? ppd.toLocaleString() : '???';
}

export default {
	name: 'WUsView',

	data() {
		return {
			filtersOpen: true,
			filter: {
				machine: 'Any',
				project: 'Any',
				core: 'Any',
				os: 'Any',
				state: 'Any',
				resources: 'Any',
				complete: false,
				days: '',
			},
		};
	},

	computed: {
		filter_summary() {
			const total = this.all_wus.length;
			const shown = this.wus.length;
			const filtered = shown < total;

			if (!filtered) return null;

			const parts = [];
			if (this.filter.machine !== 'Any')
				parts.push(`machine: ${this.filter.machine}`);
			if (this.filter.project !== 'Any')
				parts.push(`project: ${this.filter.project}`);
			if (this.filter.core !== 'Any')
				parts.push(`core: ${this.filter.core}`);
			if (this.filter.os !== 'Any') parts.push(`os: ${this.filter.os}`);
			if (this.filter.state !== 'Any')
				parts.push(`state: ${this.filter.state}`);
			if (this.filter.resources !== 'Any')
				parts.push(`resources: ${this.filter.resources}`);
			if (this.filter.days) parts.push(`within ${this.filter.days}d`);
			if (this.filter.complete) parts.push('complete only');

			return { shown, total, parts };
		},
		columns() {
			return [
				'Machine',
				'Project',
				'Core',
				'OS',
				'Status',
				'Progress',
				'TPF',
				'PPD',
				'Assign Time',
			];
		},

		Unit() {
			return Unit;
		},
		all_wus() {
			return Array.from(this.$machs.get_units()).sort(cmp_wus);
		},

		wus() {
			let days = parseFloat(this.filter.days);

			return this.all_wus.filter(
				(unit) =>
					filter_unit(this.filter, unit, 'machine') &&
					filter_unit(this.filter, unit, 'project') &&
					filter_unit(this.filter, unit, 'core') &&
					filter_unit(this.filter, unit, 'os', 'os_title') &&
					filter_unit(this.filter, unit, 'state') &&
					filter_unit(this.filter, unit, 'resources') &&
					(!this.filter.days ||
						!isFinite(this.filter.days) ||
						to_days(new Date()) - to_days(unit.assign.time) <=
							days) &&
					(!this.filter.complete || unit.wu_progress == 1),
			);
		},

		machines() {
			return unique_values(this.all_wus, 'machine');
		},
		projects() {
			return unique_values(this.all_wus, 'project');
		},
		cores() {
			return unique_values(this.all_wus, 'core');
		},
		oses() {
			return unique_values(this.all_wus, 'os_title');
		},
		states() {
			return unique_values(this.all_wus, 'state');
		},
		resources() {
			return unique_values(this.all_wus, 'resources');
		},
		tpf_min() {
			return this.format_tpf(array_min(this.wus, 'tpf_secs'));
		},
		tpf_max() {
			return this.format_tpf(array_max(this.wus, 'tpf_secs'));
		},
		tpf_avg() {
			return this.format_tpf(array_avg(this.wus, 'tpf_secs'));
		},
		ppd_min() {
			return format_ppd(array_min(this.wus, 'ppd_raw'));
		},
		ppd_max() {
			return format_ppd(array_max(this.wus, 'ppd_raw'));
		},
		ppd_avg() {
			return format_ppd(Math.round(array_avg(this.wus, 'ppd_raw')));
		},
	},

	created() {
		this.default_filter = Object.assign({}, this.filter);
	},
	mounted() {
		this.$machs.wus_enable(true);
	},

	methods: {
		reset() {
			Object.assign(this.filter, this.default_filter);
		},
		format_tpf(tpf) {
			return tpf ? this.$util.time_interval(tpf) : '???';
		},
	},
};
</script>

<template lang="pug">
.wus-view.page-view
  MainHeader

  .view-body
    .filter-panel.view-panel
      .filter-panel-header(@click="filtersOpen = !filtersOpen")
        HelpBalloon.header-title(name="Work Unit Stats"): p.
          Filter Work Units to compute stats on different groups of units.

        .filter-panel-controls
          .filter-summary-badge(v-if="filter_summary",
            title="Filters active")
            | {{filter_summary.parts.length}} filter{{filter_summary.parts.length > 1 ? 's' : ''}} active

          Button.button-icon(
            :icon="filtersOpen ? 'chevron-up' : 'chevron-down'",
            :title="filtersOpen ? 'Collapse filters' : 'Expand filters'")

      transition(name="filter-collapse")
        table.view-table.wu-filters(v-show="filtersOpen")
          thead
            tr
              th Machine
              th Project
              th Core
              th OS
              th State
              th Resources
              th Within
              th Complete
              th.actions Actions

          tbody
            tr
              td
                select(v-model="filter.machine")
                  option(value="Any") Any
                  option(v-for="machine in machines", :value="machine") {{machine}}

              td
                select(v-model="filter.project")
                  option(value="Any") Any
                  option(v-for="project in projects", :value="project") {{project}}

              td
                select(v-model="filter.core")
                  option(value="Any") Any
                  option(v-for="core in cores", :value="core") {{core}}

              td
                select(v-model="filter.os")
                  option(value="Any") Any
                  option(v-for="os in oses", :value="os") {{os}}

              td
                select(v-model="filter.state")
                  option(value="Any") Any
                  option(v-for="v in states", :value="v") {{v}}

              td
                select(v-model="filter.resources")
                  option(value="Any") Any
                  option(v-for="r in resources", :value="r") {{r}}

              td(title="Only include units assigned within this number of days.")
                input(v-model="filter.days", type=number, placeholder="days",
                  :class="{error: !isFinite(filter.days)}")

              td(title="Only include completed units.")
                input(type="checkbox", v-model="filter.complete")

              td.actions
                span
                  Button.button-icon(icon="refresh", @click="reset",
                    title="Reset stats filter")

      .filter-result-bar(v-if="filter_summary")
        span.filter-result-text
          | Showing
          strong  {{filter_summary.shown}}
          |  of
          strong  {{filter_summary.total}}
          |  work units
          template(v-if="filter_summary.parts.length")
            |  &mdash; filtered by
            em  {{filter_summary.parts.join(', ')}}
        Button.button-icon(icon="times", @click="reset",
          title="Clear all filters")

    p(v-if="!wus.length") No matching work units.
    table.view-table.wu-stats(v-else)
      thead
        tr
          th
          th Average
          th Min
          th Max

      tbody
        tr(title="Time Per Frame.  Time to complete 1% of the unit.")
          th TPF
          td {{tpf_avg}}
          td {{tpf_min}}
          td {{tpf_max}}

        tr(title="Points Per Day")
          th PPD
          td {{ppd_avg}}
          td {{ppd_min}}
          td {{ppd_max}}

    HelpBalloon.header-title(name="Recent Work Unit History"): p.
      A log of recent work WUs completed by your machines.

    .units-view(:style="Unit.get_column_grid_style(columns, ' 1fr')")
      UnitHeaders(:columns="columns") Info

      UnitsView(:units="wus", :columns="columns", v-slot="{unit}")
        Button.button-icon(icon="info-circle",
          :route="`/unit/${unit.id}`", title="View Work Unit details")</template>

<style lang="stylus">
.wus-view
  .filter-panel
    padding 0
    overflow hidden

    .filter-panel-header
      display flex
      align-items center
      justify-content space-between
      padding var(--gap) calc(var(--gap) * 1.5)
      cursor pointer
      user-select none

      &:hover
        background var(--table-header-bg)

      .filter-panel-controls
        display flex
        align-items center
        gap var(--gap)

      .filter-summary-badge
        font-size 11px
        padding 2px 8px
        border-radius var(--border-radius)
        background var(--highlight-color)
        color var(--body-bg)
        font-weight bold
        white-space nowrap

    .wu-filters
      td
        padding 0
        select, input
          width 100%
          border-radius 0

      .actions
        text-align center

      td.actions
        padding 0 var(--gap)

    .filter-result-bar
      display flex
      align-items center
      justify-content space-between
      padding calc(var(--gap) / 2) calc(var(--gap) * 1.5)
      border-top var(--table-border)
      background var(--table-header-bg)
      font-size 90%
      color var(--subtitle-color)

      strong
        color var(--body-fg)

      em
        font-style normal
        color var(--body-fg)

  .filter-collapse-enter-active,
  .filter-collapse-leave-active
    transition max-height 0.2s ease, opacity 0.15s ease
    overflow hidden
    max-height 200px

  .filter-collapse-enter-from,
  .filter-collapse-leave-to
    max-height 0
    opacity 0
</style>
