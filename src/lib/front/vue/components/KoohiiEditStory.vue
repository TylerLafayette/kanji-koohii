<template>

  <div style="min-height:100px;background:#ccc;">
   
  <form name="EditStory" method="post" action="/study/kanji/1">

    <!-- we still need this for the "Add to learned list" submit which is NOT ajax -->
    <input v-model="kanjiData.ucs_id" type="hidden" name="ucs_code">

    <div id="my-story" lang="ja">

      <div ref="maskArea" class="padding rtkframe">

        <!-- left -->
        <div class="left">
          
          <div class="framenum" title="Frame number">{{ kanjiData.framenum }}</div>

          <div :class="{ kanji: true, onhover: isReviewMode }">
            <cjk-lang-ja>{{ kanjiData.kanji }}</cjk-lang-ja>
          </div>

          <div class="strokecount" title="Stroke count">[{{ kanjiData.strokecount }}]<br/>
            <span style="font-size:120%"><cjk-lang-ja>{{ kanjiData.onyomi }}</cjk-lang-ja></span>
          </div>
        
        </div>
        <!-- /left -->

        <!-- right -->
        <div class="right">
         
          <div class="keyword">
            <span class="JSEditKeyword" title="Click to edit the keyword" @click="onKeyword">{{ displayKeyword }}</span>
          </div>

          <div id="storybox">

            <!-- view / edit story -->

            <div v-if="isEditing" id="storyedit">
              
              <div v-if="koohiiformGetErrors" class="formerrormessage">
                <span v-html="koohiiformGetErrors"></span>
              </div>

              <textarea id="frmStory" v-model="postStoryEdit" name="txtStory"></textarea>

              <!-- FIXME  refactor to flex... -->
              <div class="controls valign">
                <div style="float:left;">
                  <input id="storyedit_public" v-model="postStoryPublic" type="checkbox" name="chkPublic">
                  <label for="storyedit_public">Share this story</label>
                </div>
                <div style="float:right;">
                  <koohii-chars-left :text="postStoryEdit" :max-length="512" :warning-limit="20" />
                  <input type="button" value="Save changes" title="Save/Update story" @click.prevent="onSubmit">
                  <input type="button" value="Cancel" name="cancel" title="Cancel changes" @click="onCancel">
                </div>
                <div class="clear"></div>
              </div>
            </div>
                    
            <div v-else id="storyview">

              <div id="sv-textarea" class="bookstyle" title="Click to edit your story" @click="onEditStory">
                
                <template v-if="postStoryView.length">

                  <div v-html="postStoryView"></div>

                  <div v-if="isFavoriteStory" class="favstory">
                    <i class="fa fa-star"></i>You starred this story
                  </div>

                </template>

                <template v-else>
                  <div  class="empty">[ click here to enter your story ]</div>              
                </template>

              </div>

              <template v-if="!isReviewMode">
                <div v-if="showLearnButton" class="controls">
                  <!-- handle via legacy code / page load -->
                  <input type="submit" name="doLearned" value="Add to learned list" class="btn btn-success" />
                </div>
      
                <div v-if="showLearnedMessage" class="msg-relearned">
                  This kanji is ready for review in the <strong>learned</strong> list.
                </div>
              </template>

            </div><!-- /storyview -->

          </div><!-- /storybox -->

        </div><!-- /right -->

        <div class="clear"></div>

      </div><!-- /rtkframe -->

      <div class="bottom"></div>

    </div>
    <!-- /#my-story -->

  </form>

  </div>

</template>

<script>
import Dom, { insertAfter } from '@lib/koohii/dom.js'

import { KoohiiAPI } from '@lib/KoohiiAPI.js'

//comps
import KoohiiCharsLeft     from '@components/KoohiiCharsLeft.vue'
import CjkLangJa         from '@components/CjkLangJa.vue'
import KoohiiSharedStory   from '@components/KoohiiSharedStory.vue'   // instantiated after publishing a story

//mixins
import KoohiiForm          from '@lib/mixins/KoohiiForm.js'
import KoohiiLoading       from '@lib/mixins/KoohiiLoading.js'


export default {
  name: 'KoohiiEditStory',

  components: {
    CjkLangJa,
    KoohiiCharsLeft
  },

  mixins: [
    KoohiiForm,
    KoohiiLoading
  ],

  props: {
    // See ./apps/koohii/modules/study/templates/editSuccess.php

    // framenum, kanji, ucs_id, keyword, onyomi, strokecount, ... (cf. kanjisPeer::getKanjiByUCS())
    kanjiData: { type: Object, required: true },
    // user edted keyword, or null
    custKeyword: { type: String, default: null },
    
    // true if instanced from the Flashcard Review page (the "Edit Story" dialog)
    isReviewMode:       { type: Boolean, default: false },
    // show a starred story in reviewmode when user's story is empty
    isFavoriteStory:    { type: Boolean, default: false },

    // Study page only, "Add to learned list" functionality
    showLearnButton:    { type: Boolean, default: false },
    showLearnedMessage: { type: Boolean, default: false },

    // ajax state
    postStoryView:      { type: String, default: '' },
    postStoryEdit:      { type: String, default: '' },
    postStoryPublic:    { type: Boolean, default: false }
  },

  data() {
    return {
      // Edit Keyword dialog instance
      oEditKeyword: null,

      isEditing: false,

      // holds instance of a KoohiiSharedStory component (visual feedback for sharing a story)
      vmStoryPublished: null,

      // keep a copy to cancel changes
      uneditedStory: ''
    }
  },

  computed: {
    displayKeyword()
    {
      return this.custKeyword || this.kanjiData.keyword
    },

    editKeywordUrl()
    {
      return '/study/editkeyword/id/' + this.kanjiData.ucs_id
    }
  },

  beforeDestroy()
  {
    // (legacy code) free resources/events used by Edit Keyword dialog
    if (this.oEditKeyword) {
      this.oEditKeyword.destroy()
      this.oEditKeyword = null
    }
  },

  methods: {

    onEditStory()
    {
      this.editStory()
    },

    onSubmit()
    {
      this.koohiiloadingShow({ target: this.$refs.maskArea })

      KoohiiAPI.postUserStory(
        { 
          ucsId:      this.kanjiData.ucs_id,
          txtStory:   this.postStoryEdit,
          isPublic:   this.postStoryPublic,
          reviewMode: this.isReviewMode
        },
        { 
          then: this.onSaveStoryResponse.bind(this)
        }
      )
    },

    onSaveStoryResponse(tron)
    {
      const props = tron.getProps()

      this.koohiiloadingHide()

      this.koohiiformHandleResponse(tron)

      if (tron.hasErrors()) return

      // keep it simple for now, after a POST forget about the "starred story" thing
      this.isFavoriteStory = false

      this.postStoryView = props.postStoryView
      this.isEditing = false

      // FIXME -- temporary code for user feedback (should use Vue based SharedStories list)

      // destroy previous instance if created
      if (this.vmStoryPublished) {
        this.vmStoryPublished.$destroy()
        this.vmStoryPublished = null
      }

      //
      // update/add/remove a shared story dynamically
      // 
      // delete story from page if already shared
      let $elSharedStory = Dom('#'+props.sharedStoryId)
      if ($elSharedStory.el()) {
        let el = $elSharedStory.closest('.rtkframe')
        Dom(el).remove()
      }

      if (!this.isReviewMode && props.isStoryShared) {
        // add the story in "new & updated"
        const elMount = document.createElement('div')
        insertAfter(elMount, '#sharedstories-new .title')

        const vmProps = {
          profileLink: props.sharedStoryAuthor,
          story:  this.postStoryView.replace(/<br\/>/g, ' '), // remove the line breaks
          divId:  props.sharedStoryId
        }
        this.vmStoryPublished = VueInstance(KoohiiSharedStory, elMount, vmProps)
      }
    },

    // currently called by SharedStoriesComponent.js ajax handler (after user clicked a "copy" button)
    onCopySharedStory(storyText)
    {
      if (this.isEditing) {
        window.alert('Can not copy a story since you are currently editing a story.')
        return
      }

      this.editStory(storyText)

      this.$nextTick(function() {
        // Dom('#main_container')[0].scrollIntoView(true)

        // scroll to top of window
        let dx = window.pageXOffset || document.documentElement.scrollLeft || document.body.scrollLeft || 0
        window.scrollTo(dx, 0)
      })
    },

    onCancel()
    {
      this.doCancel()
    },

    doCancel()
    {
      this.postStoryEdit = this.uneditedStory
      this.isEditing = false
    },

    /**
     * Edit Story or Edit a copy of another user's story.
     * 
     * @param {Object} sCopyStory   The "copy" story feature will set this to the copied story text.
     */  
    editStory(sCopyStory)
    {
      this.uneditedStory = this.postStoryEdit

      // edit a new story, cancel will restore the previous one
      if (sCopyStory) {
        this.postStoryEdit = sCopyStory
        this.postStoryPublic = false      // default to private after copying a story
      }
      
      this.isEditing = true

      // note:AFTER toggling isEditing,order is important!
      this.$nextTick(function() {
       
        const elTextArea = Dom('#frmStory')[0];
        // DOM is now updated
        this.setCaretToEnd(elTextArea)
      })

    },

    // (legacy code) instance Edit Keyword dialog, not yet refactored to Vue
    onKeyword(event)
    {
      const el = event.target
      // console.log('onKeyword() %o', el)

      // callback with the modified keyword
      const callback = (keyword) => {
        // console.log('onKeyword callback()')
        el.innerHTML = keyword;
        
        // invalidate cached dialog
        this.oEditKeyword.destroy()
        this.oEditKeyword = null
      }

      if (!this.oEditKeyword) {
        const url     = this.editKeywordUrl
        const options = { context: ["my-story", "tr", "tr", null, [-6, 6]] }
        this.oEditKeyword = new App.Ui.EditKeywordComponent(url, options, callback)
      }
      else {
        this.oEditKeyword.show()
      }

      return false
    },

    // cross-browser (now obsolete?) move caret to end of input field
    setCaretToEnd(element)
    {
      if (element.createTextRange) {
        var range = element.createTextRange()
        range.collapse(false)
        range.select()
      }
      else if (element.setSelectionRange) {
        element.focus()
        var length = element.value.length
        element.setSelectionRange(length, length)
      }
    }
  }

  // created()
  // {
  //   console.log('KoohiiEditStory::created()')

  //   this.postStoryView   = this.postStoryView
  //   this.postStoryEdit   = this.postStoryEdit
  //   this.postStoryPublic = this.postStoryPublic
  // }

}
</script>

<style lang="scss">
// Used on Study page & Edit Story Dialog (review page)

/* self story */
#my-story {
  position: relative;
  background: #fff;
  border: 1px solid #E8E5C9;
  box-shadow: 0 1px 2px 0px rgba(170, 165, 130, 0.3);
  .rtkframe {
    border: none;
    padding: 16px 17px 0;
    background: none;
  }
  .bottom {
    width: 100%;
    height: 23px;
    overflow: hidden;
    clear: both;
    background: #fff;
    background: linear-gradient(to bottom, #fff 0%, #f8f8f8 100%);
  }
  .msg-relearned {
    color: #61932B;
    padding: 10px 5px 0;
  }
  .kk-charsleft {
    margin: 0 0.5em 0 0;
  }
}

.rtkframe {
  clear: both;
  position: relative;
  background: white;
  padding: 15px;

  .left {
    float: left; width: 68px; text-align: center;
  }
  .right {
    margin-left: 82px;
  }
  .framenum {
    font-size: 14px;
    line-height: 1em;
    margin: 0 0 20px;
  }
  .kanji {
    width: 100%;
    min-height: 50px;
    margin: 0 0 8px;
    font-size: 50pt;
    line-height: 1em;
  }
  .strokecount {
    color: #8e8e8e;
  }
  .keyword {
    font: 20px Georgia, Times New Roman, sans-serif;
    letter-spacing: 2px;
    text-align: right;
    .edition {
      font-size: 0.6em;
    }
  }

  #storybox {
    padding: 14px 0 0;
  }
  #sv-textarea {
    padding: 5px;
    height: auto;
    min-height: 100px;
    &:hover {
      background: #f5f5f5;
    }
  }

  .bookstyle .empty {
    color: #888;
  }

  .favstory {
    margin: 1em 0 0;
    color: #666;
    font-style: italic;
    i {
      margin-right: 0.5em;
      color: #666;
    }
  }
}

#storyview .controls {
  padding-right: 16px;
  margin: 12px 0 0;
  text-align: right;
}

.rtkframe #storyedit {
  textarea {
    width: 100%;
    height: 153px;
    padding: 5px;
    border: 1px solid #e8e5c9;
    background: #f5f5f5;
    box-sizing: border-box;
  }
  .controls {
    margin: 4px 0 0;
    height: 25px;
  }
}

.viewtoggle a {
  font-size: 80%;
  font-weight: normal;
  &:active, &:visited, &:hover {
    font-size: 80%;
    font-weight: normal;
  }
}

/* story content styling (My Stories and Shared Stories) */

.rtkframe .bookstyle {
  color: #100800;
  font-size: 14px;
  line-height: 1.5em;
  text-align: justify;
  /*ie fix expandbox*/
  word-wrap: break-word;
}

.bookstyle em {
  letter-spacing: 0.05em;
}

.rtkframe .bookstyle {
  a {
    font-weight: normal;
    text-decoration: none;
    &:hover {
      text-decoration: underline;
    }
  }
  .frnr {
    color: #484;
    font-family: sans-serif;
  }
}

#storyview .bookstyle {
  line-height: 1.5em;
}

@media (min-width: 600px) {
  .rtkframe {
    .bookstyle, textarea {
      font-size: 16px;
    }
  }
}


/* Story popup (flashcard review page) kanji shows on mouseover */
#my-story .onhover {
  span {
    visibility: hidden;
    color: #fff;
  }

  &:hover {
    background: none;
    span {
      visibility: visible;
      color: #000;
      background: #fff;
    }
  }

  /* Story edit form errors */
  .formerrormessage { background:none; margin:0.5em 0; padding:0 5px; border:none; }
}

/* DIALOG mode */

/* desktop dialog : use a fixed width (looks nicer), hide the big close button */
.rtk-skin-dlg {
  #my-story {
    width: 500px;
    border: none;
  }
  .editstory-close {
    display: none;
  }
}

/* mobile dialog */
.rtk-mobl-dlg {
  #my-story {
    box-shadow: none;
    border: none;
    /* remove decorations */
    .rtkframe {
      padding: 16px 10px 0;
    }
    .right {
      margin-left: 0;
    }

    #sv-textarea {
      border: 1px solid #fff;
      background: #f5f5f5;
      border-radius: 5px;
    }
  }

  .left {
    width: 50px;
    display: none;
  }

  /* hint edit box by default for touch */
  #JSEditStoryLoading { width:auto; }
}

</style>

