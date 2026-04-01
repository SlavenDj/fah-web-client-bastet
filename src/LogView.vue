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
export default {
	name: 'LogView',
	props: ['mach', 'query'],

	data() {
		return {
			search: '',
			errors: false,
			warnings: false,
			follow: true,
			line_height: 20, // Fallback default
			view_height: 0,
			scroll_percent: 0,
			log_top: 0,
			top_line: 0,
			resizeObserver: null,
			scroll_raf: null,
			fade_timer: null,
		};
	},

	watch: {
		count() {
			// Use nextTick to allow DOM update before calculating scroll end
			this.$nextTick(() => {
				if (this.follow) this.scroll_to_end();
				else this.update_scroll();
			});
		},
		follow(val) {
			if (val) this.scroll_to_end();
		},
		query(newQuery) {
			if (newQuery !== undefined) this.search = newQuery;
		},
	},

	computed: {
		count() {
			return this.lines.length;
		},

		visible_lines() {
			return this.lines
				.slice(this.start, this.end)
				.map((line) => [line[0], this.$util.ansi2html(line[1])]);
		},

		log() {
			let log = this.mach.get_data().log || [];
			return log.map((line, index) => [index, line]);
		},

		line_filter() {
			let searchExp = null;
			if (this.search) {
				// Safely escape user input for Regex
				const escapedSearch = this.search.replace(
					/[.*+?^${}()|[\]\\]/g,
					'\\$&',
				);
				searchExp = new RegExp(escapedSearch, 'i');
			}

			let errWarnExp = null;
			if (this.errors || this.warnings) {
				errWarnExp = new RegExp(
					':[' +
						(this.errors ? 'E' : '') +
						(this.warnings ? 'W' : '') +
						'] :',
				);
			}

			return (lineText) => {
				if (searchExp && !searchExp.test(lineText)) return false;
				if (errWarnExp && !errWarnExp.test(lineText)) return false;
				return true;
			};
		},

		lines() {
			if (!this.search && !this.errors && !this.warnings) return this.log;
			return this.log.filter((line) => this.line_filter(line[1]));
		},

		start() {
			// Buffer a few lines above for smoother scrolling
			return Math.max(0, this.top_line - 5);
		},

		end() {
			// Buffer lines below to prevent white space while scrolling fast
			return Math.min(this.count, this.top_line + this.view_lines + 10);
		},

		view_lines() {
			return this.line_height
				? Math.ceil(this.view_height / this.line_height)
				: 0;
		},

		wrapper_height() {
			return this.line_height * this.count;
		},

		content_offset() {
			return this.start * this.line_height;
		},
	},

	mounted() {
		this.search = this.query || '';
		this.mach.log_enable(true);

		this.calculate_dimensions();

		// Observe resizing dynamically
		this.resizeObserver = new ResizeObserver(() => {
			this.calculate_dimensions();
			this.update_scroll();
		});

		if (this.$refs.log) {
			this.resizeObserver.observe(this.$refs.log);
		}

		this.update();
	},

	unmounted() {
		this.mach.log_enable(false);
		if (this.resizeObserver) this.resizeObserver.disconnect();
		if (this.scroll_raf) cancelAnimationFrame(this.scroll_raf);
		if (this.fade_timer) clearTimeout(this.fade_timer);
	},

	methods: {
		calculate_dimensions() {
			let log = this.$refs.log;
			if (!log) return;

			this.view_height = log.clientHeight;

			// Compute actual line-height
			let e = document.createElement('div');
			e.className = 'log-line';
			e.innerHTML = '&nbsp;';
			e.style.visibility = 'hidden';
			e.style.position = 'absolute';
			log.appendChild(e);
			this.line_height = e.getBoundingClientRect().height || 20;
			log.removeChild(e);
		},

		reset() {
			this.search = '';
			this.errors = this.warnings = false;
		},

		update() {
			this.$nextTick(() => {
				if (this.follow) this.scroll_to_end();
				this.update_scroll();
			});
		},

		scroll_to_end() {
			let log = this.$refs.log;
			if (log) {
				log.scrollTop = log.scrollHeight;
				this.update_scroll();
			}
		},

		update_scroll() {
			let log = this.$refs.log;
			if (!log) return;

			this.log_top = log.scrollTop;
			const maxScroll = Math.max(0, log.scrollHeight - log.clientHeight);

			// Snap to follow if within 10px of bottom (handles floating point/zoom rounding errors)
			if (maxScroll > 0) {
				this.follow = maxScroll - this.log_top <= 10;
			}

			this.top_line = Math.floor(this.log_top / this.line_height);

			let percent =
				maxScroll > 0
					? Math.floor((this.log_top / maxScroll) * 100)
					: 100;

			// Compute scroll percent for display overlay
			if (
				this.scroll_percent !== percent &&
				!(this.scroll_percent === 0 && percent === 100)
			) {
				this.scroll_percent = percent;

				clearTimeout(this.fade_timer);
				if (this.$refs.percent) {
					this.$refs.percent.classList.remove('fade-out');
					this.fade_timer = setTimeout(() => {
						if (this.$refs.percent)
							this.$refs.percent.classList.add('fade-out');
					}, 800);
				}
			}
		},

		scroll() {
			// Use requestAnimationFrame for fluid 60fps virtualization updates
			if (this.scroll_raf) cancelAnimationFrame(this.scroll_raf);
			this.scroll_raf = requestAnimationFrame(() => {
				this.update_scroll();
			});
		},
	},
};
</script>

<template lang="pug">
.log-view.page-view.fixed-view
  ViewHeader(title="Machine Log", :subtitle="mach.get_name()")

  .view-body
    .view-panel.log-controls
      label Search
      input(v-model="search", type="text", placeholder="Search in log...")
      label(title="Filter log for error messages").
        #[input(v-model="errors", type="checkbox")] Errors
      label(title="Filter log for warning messages").
        #[input(v-model="warnings", type="checkbox")] Warnings
      
      Button.button-icon(title="Reset search", icon="repeat", @click="reset", :disabled="!search && !errors && !warnings")
      Button.button-icon(title="Auto-scroll to latest log entries", :icon="follow ? 'toggle-down' : 'pause'", @click="follow = !follow", :class="{ 'active-follow': follow }")

    .log-percent.fade-out(ref="percent")
      div {{scroll_percent}}%

    .view-panel.log(@scroll="scroll", ref="log")
      .log-line(v-if="!log.length") Loading log...
      .log-line(v-else-if="!count") No matching log lines.

      .log-wrapper(v-else, ref="wrap", :style="{height: wrapper_height + 'px'}")
        .log-content(:style="{'padding-top': content_offset + 'px'}")
          .log-line(v-for="line in visible_lines", :key="line[0]", v-html="line[1]")
</template>

<style lang="stylus">
.log-view
  .view-body
    position relative
    font-family var(--mono-font)

    .view-panel
      color var(--log-fg)
      background var(--log-bg)

    .log-controls
      display flex
      flex-wrap wrap
      gap var(--gap)
      align-items center

      input[type=text]
        flex 1

      .active-follow
        color var(--link-color)

    .log-percent
      position absolute
      left 10%
      top 45%
      width 80%
      margin auto
      text-align center
      font-size 32pt
      opacity 1
      pointer-events none
      color var(--panel-fg)

      &.fade-out
        opacity 0
        transition opacity 1s linear

    .log.view-panel
      flex 1
      overflow auto
      padding var(--gap)
      will-change scroll-position

      .log-line
        white-space nowrap
</style>
