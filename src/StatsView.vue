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
	name: 'StatsView',

	computed: {
		stats() {
			return this.$stats.get_data();
		},
		team() {
			return this.$stats.get_team();
		},
		is_anon() {
			return this.$stats.is_anon();
		},
		uid() {
			return this.stats.id;
		},

		team_owner() {
			for (let team of this.$adata.teams || [])
				if (team.team == this.team.team) return true;
			return false;
		},

		// UI Logic for milestones
		userProgress() {
			const rank = this.stats.rank || 1000000;
			const next = this.nextMilestoneValue(rank);
			const prev = this.prevMilestoneValue(rank);
			const totalRange = prev - next;
			const currentProgress = prev - rank;
			return Math.min(
				100,
				Math.max(0, (currentProgress / totalRange) * 100),
			).toFixed(1);
		},
	},

	methods: {
		nextMilestoneValue(rank) {
			if (rank <= 100) return 10;
			if (rank <= 1000) return 100;
			if (rank <= 10000) return 1000;
			if (rank <= 100000) return 10000;
			return 100000;
		},
		prevMilestoneValue(rank) {
			if (rank <= 100) return 100;
			if (rank <= 1000) return 1000;
			if (rank <= 10000) return 10000;
			if (rank <= 100000) return 100000;
			return 1000000;
		},
		top(rank) {
			rank = rank == undefined ? 1000001 : rank;
			for (let i = 2; i < 7; i++) {
				let level = Math.pow(10, i);
				if (rank <= level) return this.$util.human_number(level, 0);
			}
		},
		tpercent(user, team) {
			if (!team) return '0.0%';
			return ((user / team) * 100).toFixed(1) + '%';
		},
	},
};
</script>

<template lang="pug">
.stats-view.page-view
  MainHeader

  .view-body
    .stats-grid
      // USER STATS CARD
      .stats-card(v-if="stats.id")
        .card-header
          .avatar-container
            .fa.fa-user-circle.user-avatar-icon
          .user-info
            .user-name-row
              a(v-if="stats.id", :href="$stats.url + '/donor/id/' + stats.id", target="_blank") {{stats.name}}
              span(v-else) Anonymous
            .user-badge(v-if="stats.rank") 
              i.fa.fa-trophy
              |  Rank # {{stats.rank.toLocaleString()}}

        .milestone-section
          .milestone-label
            span Next Goal: Top {{nextMilestoneValue(stats.rank || 1000000).toLocaleString()}}
            span {{userProgress}}%
          ProgressBar(:progress="userProgress")

        .main-stats-grid
          .stat-box
            .stat-value {{ (stats.score || 0).toLocaleString() }}
            .stat-label Total Points
          .stat-box
            .stat-value {{ (stats.wus || 0).toLocaleString() }}
            .stat-label WUs Completed

        .activity-section
          h3 Client Activity
          .activity-row
            .activity-item
              span.dot.active
              span 7 Days: 
              strong {{ stats.active_7 }}
            .activity-item
              span.dot
              span 50 Days: 
              strong {{ stats.active_50 }}

        .awards-section
          h3 Achievement Awards
          .awards-flex
            Award(title="WUs", :user="uid", wus, :disabled="!stats.wus")
            Award(title="Points", :user="uid", :disabled="!stats.wus")

      .stats-card.no-info(v-else-if="is_anon")
        i.fa.fa-user-secret
        p Folding Anonymously
        small Join the community by creating an account!

      // TEAM STATS CARD
      .stats-card(v-if="team.team")
        .card-header
          .avatar-container
            img.team-logo(v-if="team.logo", :src="team.logo")
            .fa.fa-users.team-avatar-icon(v-else)
          .user-info
            .user-name-row
              a(:href="$stats.url + '/team/' + team.team", target="_blank") {{team.name}}
              Button.button-icon(v-if="team_owner", icon="pencil", title="Edit team", route="/account/teams")
            .user-badge.team-badge(v-if="team.trank") 
              i.fa.fa-globe
              |  World Rank # {{team.trank.toLocaleString()}}

        .milestone-section
          .milestone-label
            span Team Contribution
            span {{ tpercent(team.score, team.tscore) }}
          ProgressBar(:progress="parseFloat(tpercent(team.score, team.tscore))")

        .main-stats-grid
          .stat-box
            .stat-value {{ (team.tscore || 0).toLocaleString() }}
            .stat-label Team Points
          .stat-box
            .stat-value {{ (team.score || 0).toLocaleString() }}
            .stat-label Your Share

        .activity-section
          h3 Contribution Details
          .contribution-data
            p You have contributed 
              strong {{ (team.wus || 0).toLocaleString() }} 
              | Work Units to this team.

        .awards-section
          h3 Team Honors
          .awards-flex
            Award(title="Team WUs", :team="team.team", wus, :disabled="!team.twus")
            Award(title="Team Points", :team="team.team", :disabled="!team.tscore")

      .stats-card.no-info(v-else)
        i.fa.fa-flag-o
        p No Team Selected
        small Teamwork makes the folding dream work!

</template>

<style lang="stylus">
.stats-view
  .view-body
    padding var(--gap)
    background var(--body-bg)

  .stats-grid
    display grid
    grid-template-columns repeat(auto-fit, minmax(400px, 1fr))
    gap calc(var(--gap) * 2)
    width 100%
    max-width 1200px
    margin 0 auto

  .stats-card
    background var(--panel-bg)
    border var(--border)
    border-radius 12px
    padding calc(var(--gap) * 1.5)
    box-shadow var(--shadow)
    transition transform 0.2s ease, box-shadow 0.2s ease
    display flex
    flex-direction column
    gap var(--gap)

    &:hover
      transform translateY(-2px)
      box-shadow 0 8px 24px rgba(0,0,0,0.15)

    &.no-info
      justify-content center
      align-items center
      text-align center
      color var(--subtitle-color)
      min-height 300px
      i
        font-size 4rem
        margin-bottom 1rem
        opacity 0.5
      p
        font-size 1.5rem
        font-weight bold
        margin 0

  .card-header
    display flex
    align-items center
    gap var(--gap)
    border-bottom 1px solid var(--border-color)
    padding-bottom var(--gap)

  .avatar-container
    width 64px
    height 64px
    background var(--table-header-bg)
    border-radius 50%
    display flex
    justify-content center
    align-items center
    overflow hidden
    border 2px solid var(--link-color)

    .user-avatar-icon, .team-avatar-icon
      font-size 2rem
      color var(--link-color)

    .team-logo
      width 100%
      height 100%
      object-fit cover

  .user-info
    flex 1
    display flex
    flex-direction column
    overflow hidden

  .user-name-row
    display flex
    align-items center
    gap 0.5rem
    a
      font-size 1.5rem
      font-weight bold
      color var(--title-color)
      text-decoration none
      white-space nowrap
      overflow hidden
      text-overflow ellipsis
      &:hover
        color var(--link-color)

  .user-badge
    background var(--link-color)
    color white
    padding 2px 10px
    border-radius 20px
    font-size 0.8rem
    width fit-content
    margin-top 4px
    &.team-badge
      background var(--success-color)

  .milestone-section
    margin var(--gap) 0

    .milestone-label
      display flex
      justify-content space-between
      font-size 0.9rem
      margin-bottom 5px
      color var(--subtitle-color)
      font-weight bold

  .main-stats-grid
    display grid
    grid-template-columns 1fr 1fr
    gap var(--gap)
    margin var(--gap) 0

    .stat-box
      background var(--table-odd)
      padding var(--gap)
      border-radius 8px
      text-align center
      border 1px solid var(--border-color)

      .stat-value
        font-size 1.8rem
        font-weight 800
        color var(--link-color)
        font-family var(--mono-font)

      .stat-label
        font-size 0.75rem
        text-transform uppercase
        letter-spacing 1px
        color var(--subtitle-color)

  .activity-section
    h3
      font-size 1rem
      margin 0 0 0.5rem 0
      color var(--title-color)

    .activity-row
      display flex
      gap var(--gap)

    .activity-item
      font-size 0.9rem
      display flex
      align-items center
      gap 5px

      .dot
        width 8px
        height 8px
        border-radius 50%
        background #666
        &.active
          background var(--success-color)
          box-shadow 0 0 8px var(--success-color)

  .awards-section
    border-top 1px solid var(--border-color)
    padding-top var(--gap)
    h3
      font-size 1rem
      margin 0 0 1rem 0

    .awards-flex
      display flex
      gap calc(var(--gap) / 2)
      flex-wrap wrap

@media (max-width 600px)
  .stats-grid
    grid-template-columns 1fr
</style>
