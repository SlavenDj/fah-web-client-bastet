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
	name: 'NewsView',

	computed: {
		feed() {
			return this.$news.get_feed();
		},
	},
};
</script>

<template lang="pug">
.news-view.page-view
  MainHeader

  .view-body
    // LOADING / EMPTY STATE
    .loading-state.view-panel(v-if="!feed.length")
      .spinner-icon
        i.fas.fa-circle-notch.fa-spin
      p Fetching latest news...

    // NEWS FEED
    article.view-panel.news-card(v-for="item in feed")
      
      .article-image(v-if="item.image")
        a(:href="item.url", target="_blank", tabindex="-1")
          img(:src="item.image", alt="Article Image")

      .article-content
        h2.article-title
          a(:href="item.url", target="_blank", v-html="item.title")

        .article-byline
          | By #[span.author {{item.author}}] &bull; #[span.date {{item.date}}]

        .article-body(v-html="item.description")

        .article-actions
          a.read-more(:href="item.url", target="_blank") 
            | Read full article &rarr;
</template>

<style lang="stylus">
.news-view .view-body
  display flex
  flex-direction column
  gap calc(var(--gap) * 1.5)

  .loading-state
    display flex
    flex-direction column
    align-items center
    justify-content center
    padding calc(var(--gap) * 4)
    color var(--subtitle-color)
    text-align center
    gap 12px
    font-size 1.1rem

    .spinner-icon
      font-size 2rem
      color var(--highlight-color)

  .news-card
    display flex
    flex-direction row-reverse
    gap calc(var(--gap) * 1.5)
    padding calc(var(--gap) * 1.5)
    transition transform 0.2s ease, box-shadow 0.2s ease

    &:hover
      transform translateY(-2px)
      box-shadow 0 4px 12px var(--shadow-color, rgba(0, 0, 0, 0.1))

    .article-content
      display flex
      flex-direction column
      gap calc(var(--gap) * 0.75)
      flex 1

    .article-title
      margin 0
      font-size 1.25rem
      line-height 1.3

      a
        color var(--body-fg)
        text-decoration none
        transition color 0.2s ease

        &:hover
          color var(--highlight-color, var(--link-color))

    .article-byline
      font-size 0.8rem
      color var(--subtitle-color)
      text-transform uppercase
      letter-spacing 0.5px
      font-weight 600

      .author
        color var(--body-fg)

    .article-body
      font-size 0.95rem
      line-height 1.6
      color var(--body-fg)
      opacity 0.9

      /* Ensure HTML content from feed spaces nicely */
      p
        margin-top 0
        margin-bottom 0.75em
        &:last-child
          margin-bottom 0

      a
        color var(--link-color)
        text-decoration none
        &:hover
          text-decoration underline

    .article-actions
      margin-top auto
      padding-top var(--gap)

      .read-more
        display inline-flex
        align-items center
        font-size 0.9rem
        font-weight bold
        color var(--highlight-color, var(--link-color))
        text-decoration none
        transition opacity 0.2s ease

        &:hover
          opacity 0.8
          text-decoration underline

    .article-image
      flex-shrink 0
      width 260px

      a
        display block
        height 100%
        min-height 160px
        border-radius var(--border-radius)
        overflow hidden
        background var(--table-header-bg)

        img
          width 100%
          height 100%
          object-fit cover
          transition transform 0.4s ease

        &:hover img
          transform scale(1.05)

/* MOBILE RESPONSIVE LAYOUT */
@media (max-width 650px)
  .news-view .view-body
    .news-card
      flex-direction column

      .article-image
        width 100%

        a
          min-height 200px
          height 200px

      .article-actions
        margin-top 0
</style>
