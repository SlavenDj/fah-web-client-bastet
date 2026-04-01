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
			log_level: 'all', // Replaces individual errors/warnings booleans
			follow: true,
			line_height: 20,
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
		log_level() {
			this.$nextTick(() => {
				if (this.follow) this.scroll_to_end();
				else this.update_scroll();
			});
		},
	},

	computed: {
		log() {
			let log = this.mach.get_data().log || [];
			return log.map((line, index) => [index, line]);
		},

		lines() {
			if (this.log_level === 'all') return this.log;

			const errExp = /(:E\s*:|ERROR:?)/i;
			const warnExp = /(:W\s*:|WARNING:?)/i;

			return this.log.filter((line) => {
				if (this.log_level === 'error') return errExp.test(line[1]);
				if (this.log_level === 'warn')
					return errExp.test(line[1]) || warnExp.test(line[1]);
				return true;
			});
		},

		analyzed_log() {
			// Evaluates the current timeline for minimap plotting (Search marks, Errors, Warnings)
			const marks = [];
			let searchCount = 0;
			const searchExp = this.search
				? new RegExp(
						this.search.replace(/[.*+?^${}()|[\]\\]/g, '\\$&'),
						'i',
					)
				: null;
			const total = this.lines.length;

			if (total === 0) return { marks, searchCount };

			for (let i = 0; i < total; i++) {
				const line = this.lines[i][1];
				let isSearchMatch = searchExp && searchExp.test(line);

				if (isSearchMatch) {
					searchCount++;
					marks.push({
						top: (i / total) * 100,
						type: 'search',
						index: i,
					});
				} else if (!this.search) {
					// Highlight timeline warnings/errors if no specific search query is active
					if (/(:E\s*:|ERROR:?)/i.test(line)) {
						marks.push({
							top: (i / total) * 100,
							type: 'error',
							index: i,
						});
					} else if (/(:W\s*:|WARNING:?)/i.test(line)) {
						marks.push({
							top: (i / total) * 100,
							type: 'warn',
							index: i,
						});
					}
				}
			}

			// Optimization cap: limit minimap div nodes if logs are extremely massive
			let finalMarks = marks;
			if (marks.length > 1000) {
				const step = Math.ceil(marks.length / 1000);
				finalMarks = marks.filter((_, idx) => idx % step === 0);
			}

			return { marks: finalMarks, searchCount };
		},

		minimap_marks() {
			return this.analyzed_log.marks;
		},
		search_match_count() {
			return this.analyzed_log.searchCount;
		},
		count() {
			return this.lines.length;
		},

		start() {
			return Math.max(0, this.top_line - 5);
		},

		end() {
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

		visible_lines() {
			// format_line handles ANSI replacement and Semantic highlighting right on render limits
			return this.lines
				.slice(this.start, this.end)
				.map((line) => [line[0], this.format_line(line[1])]);
		},
	},

	mounted() {
		this.search = this.query || '';
		this.mach.log_enable(true);
		this.calculate_dimensions();

		this.resizeObserver = new ResizeObserver(() => {
			this.calculate_dimensions();
			this.update_scroll();
		});

		if (this.$refs.log) this.resizeObserver.observe(this.$refs.log);
		this.update();
	},

	unmounted() {
		this.mach.log_enable(false);
		if (this.resizeObserver) this.resizeObserver.disconnect();
		if (this.scroll_raf) cancelAnimationFrame(this.scroll_raf);
		if (this.fade_timer) clearTimeout(this.fade_timer);
	},

	methods: {
		format_line(text) {
			// 1. Strip ANSI codes
			let clean = text.replace(/\x1B\[[0-9;]*[a-zA-Z]/g, '');

			// 2. Escape standard HTML to prevent injection
			let html = clean
				.replace(/&/g, '&amp;')
				.replace(/</g, '&lt;')
				.replace(/>/g, '&gt;');

			// 3. Apply FAH semantic highlighting
			// -> Timestamps
			html = html.replace(
				/\b(\d{2}:\d{2}:\d{2})\b/g,
				'<span class="hl-time">$1</span>',
			);
			// -> PRCG / Project Details
			html = html.replace(
				/(Project:?\s*\d+)/gi,
				'<span class="hl-project">$1</span>',
			);
			html = html.replace(
				/\b(Run\s*\d+,\s*Clone\s*\d+,\s*Gen\s*\d+)\b/gi,
				'<span class="hl-project">$1</span>',
			);
			html = html.replace(
				/\b(PRCG\s*\d+,\d+,\d+,\d+)\b/gi,
				'<span class="hl-project">$1</span>',
			);
			// -> Work Units & Slots
			html = html.replace(
				/\b((?:FS|WU)\d{2})\b/g,
				'<span class="hl-fah">$1</span>',
			);
			// -> Errors & Warnings
			html = html.replace(
				/(:E\s*:|ERROR:?)/gi,
				'<span class="hl-error">$&</span>',
			);
			html = html.replace(
				/(:W\s*:|WARNING:?)/gi,
				'<span class="hl-warning">$&</span>',
			);

			// 4. Dynamic Search Highlighting (Only wrap outside of semantic HTML tags)
			if (this.search) {
				const escapedSearch = this.search.replace(
					/[.*+?^${}()|[\]\\]/g,
					'\\$&',
				);
				const searchExp = new RegExp(`(${escapedSearch})`, 'ig');

				const parts = html.split(/(<[^>]+>)/);
				for (let i = 0; i < parts.length; i++) {
					if (!parts[i].startsWith('<')) {
						parts[i] = parts[i].replace(
							searchExp,
							'<mark>$1</mark>',
						);
					}
				}
				html = parts.join('');
			}

			return html;
		},

		on_minimap_click(e) {
			const rect = e.currentTarget.getBoundingClientRect();
			const percent = Math.max(
				0,
				Math.min(1, (e.clientY - rect.top) / rect.height),
			);
			const targetIndex = Math.floor(percent * this.lines.length);
			const log = this.$refs.log;

			if (log) {
				log.scrollTop = targetIndex * this.line_height;
				this.follow = false;
			}
		},

		calculate_dimensions() {
			let log = this.$refs.log;
			if (!log) return;
			this.view_height = log.clientHeight;

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
			this.log_level = 'all';
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

			if (maxScroll > 0) {
				this.follow = maxScroll - this.log_top <= 10;
			}

			this.top_line = Math.floor(this.log_top / this.line_height);

			let percent =
				maxScroll > 0
					? Math.floor((this.log_top / maxScroll) * 100)
					: 100;

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
      span.badge(v-if="search", :class="search_match_count ? 'badge-warn' : 'badge-muted'") {{search_match_count}} matches
      span.badge.badge-warn(v-else-if="log_level !== 'all'") {{lines.length}} lines
      span.badge.badge-muted(v-else) {{log.length}} lines
      
      label Level
      select(v-model="log_level")
        option(value="all") All Levels
        option(value="warn") Warnings & Errors
        option(value="error") Errors Only
      
      Button.button-icon(title="Reset filters", icon="repeat", @click="reset", :disabled="!search && log_level === 'all'")
      Button.button-icon(title="Auto-scroll to latest log entries", :icon="follow ? 'toggle-down' : 'pause'", @click="follow = !follow", :class="{ 'active-follow': follow }")

    .log-percent.fade-out(ref="percent")
      div {{scroll_percent}}%

    .log-container
      .view-panel.log(@scroll="scroll", ref="log")
        .log-line(v-if="!log.length") Loading log...
        .log-line(v-else-if="!count") No matching log lines.

        .log-wrapper(v-else, ref="wrap", :style="{height: wrapper_height + 'px'}")
          .log-content(:style="{'padding-top': content_offset + 'px'}")
            .log-line(v-for="line in visible_lines", :key="line[0]", v-html="line[1]")
      
      .minimap(v-if="minimap_marks.length", @click="on_minimap_click", title="Click to jump to point in timeline")
        .minimap-mark(v-for="(mark, i) in minimap_marks", :key="i", :class="'type-' + mark.type", :style="{top: mark.top + '%'}")

</template>

<style lang="stylus">
.log-view
  .view-body
    position relative
    font-family var(--mono-font)
    display flex
    flex-direction column

    .view-panel
      color var(--log-fg)
      background var(--log-bg)

    .log-controls
      display flex
      flex-wrap wrap
      gap var(--gap)
      align-items center
      margin-bottom var(--gap)

      input[type=text]
        flex 1
      select
        padding 4px 8px

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
      z-index 50

      &.fade-out
        opacity 0
        transition opacity 1s linear

    .log-container
      position relative
      flex 1
      display flex
      min-height 0
      overflow hidden

      .log.view-panel
        flex 1
        overflow-y auto
        padding var(--gap)
        padding-right 24px /* Provide clearance so text doesn't overlap the minimap */
        will-change scroll-position

        .log-line
          white-space nowrap

      .minimap
        position absolute
        right 18px /* Slightly offset to not interrupt standard browser scrollbars */
        top 0
        bottom 0
        width 12px
        background rgba(128, 128, 128, 0.1)
        border-radius 4px
        z-index 10
        cursor pointer
        transition background 0.2s ease

        &:hover
            background rgba(128, 128, 128, 0.25)

        .minimap-mark
          position absolute
          left 0
          right 0
          height 3px
          opacity 0.8
          &.type-search
            background-color var(--hl-search-bg, #ffeb3b)
            z-index 3
          &.type-error
            background-color var(--hl-error-fg, #f44336)
            z-index 2
          &.type-warn
            background-color var(--hl-warning-fg, #ff9800)
            z-index 1

/* Semantic HTML Highlighting Overrides */
.hl-time
  color var(--hl-time-fg, #888)
.hl-project
  color var(--hl-project-fg, #4caf50)
  font-weight bold
.hl-fah
  color var(--hl-fah-fg, #2196f3)
.hl-error
  color var(--hl-error-fg, #f44336)
  font-weight bold
.hl-warning
  color var(--hl-warning-fg, #ff9800)
  font-weight bold
mark
  background-color var(--hl-search-bg, rgba(255, 235, 59, 0.4))
  color inherit
  border-radius 2px
</style>
