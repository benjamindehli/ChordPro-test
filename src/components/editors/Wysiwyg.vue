<template>
  <div id="app">
    <nav></nav> <!-- TODO: main-navigation module -->
    <main id="mainContent">
      <div class="main-content">


        <div class="source-input">
          <div class="source-input-menu">
            <div v-on:click="showModal('editSongInfo')" v-for="menuItem in editorMenu.menuItems" class="source-input-menu-item">
              {{ menuItem.title }}
            </div>
            <div class="clearfix"></div>
          </div>

          <ul class="activeChord">
            <li v-for="variation in activeChord.variations" v-on:click='insertTextAtCursor(variation.markup)'>{{variation.name}}</li>
          </ul>
          <textarea id="source-input" v-on:keyup="updateFromMarkupEditor()" v-model="source"></textarea>
        </div>

        <div class="lines">
          <div class="meta-info">
            <h1 v-if="metaInfo.title">{{ metaInfo.title }}</h1>
            <h2 v-if="metaInfo.subtitle">{{ metaInfo.subtitle }}</h2>
            <div class="meta-info-data">
              <span v-if="metaInfo.key" class="key">
                Key: {{ metaInfo.key }}
              </span>
              <span v-if="metaInfo.time" class="time">Signature: {{ metaInfo.time }}</span>
              <span v-if="metaInfo.tempo" class="tempo">
                Bpm: {{ metaInfo.tempo }}
              </span>
              <span v-if="metaInfo.duration" class="duration">
                Duration: {{ metaInfo.duration }}
              </span>
            </div>
          </div>

          <div v-for="line in lines" class="line">
            <div v-for="directive in line.directives" class="directive">
              <span v-if="directive.type == 'comment'" class="comment">{{ directive.name}}</span>
            </div>
            <div v-html="line.chordString" class="chord-string"></div>
            <div v-html="line.lyricString" class="lyric-string"></div>
          </div>
        </div>
        <div class="clearfix"></div>
        <div v-for=""></div>
        <modals></modals>
      </div>



      <footer></footer> <!-- TODO: main-footer module -->
    </main>
  </div>
</template>

<script>
  import Modals from './Modals.vue';
  import * as quark from 'quark-gui';

  export default {
    name: 'app',
    components: {
      modals: Modals
    },
    data () {
      return {
        metaInfo: {},
        source: `{t:this is the title} {st:this is the subtitle}
        {key:Am}{time:4/4}{tempo:96}{duration:3:26}
        {c:this is a comment}
        {c:this is a another comment}
        thi[am]s is th[g]e firs[dm]t line
        and th[am]is is the [gm7]second`,
        keys: ["C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"],
        chords: [],
        activeChord: {},
        editorMenu: {
          menuItems: [
          {
            title: 'Edit songinfo',
            iconClass: '',
            modalName: 'editSonginfo'
          },
          {
            title: 'Insert chord',
            iconClass: '',
            modalName: 'insertChord'
          },
          {
            title: 'Insert comment',
            iconClass: '',
            modalName: 'insertComment'
          },
          {
            title: 'Transpose',
            iconClass: '',
            modalName: 'transpose'
          },
          {
            title: 'Export',
            iconClass: '',
            modalName: 'export'
          }
          ]
        },
        menuButtons: []
      }
    },
    created (){
      var chords = []
      this.keys.forEach(function(key){
        chords.push({
          name: key,
          variations: this.getVariations(key)
        });
      }.bind(this));
      this.chords = chords;
      this.getMetaInfo();
    },
    computed: {
      lines (){
        return this.getLines(this.source);
      },
    },
    methods: {
      updateFromMarkupEditor: function(){
        this.getMetaInfo();
      },
      getVariations: function(key){
        var variations = [
        {
          name: key,
          markup: `[${key}]`
        },
        {
          name: `${key} minor`,
          markup: `[${key}m]`
        },
        {
          name: `${key} 6`,
          markup: `[${key}6]`
        },
        {
          name: `${key} 7`,
          markup: `[${key}7]`
        },
        {
          name: `${key} Maj7`,
          markup: `[${key}maj7]`
        },
        ];
        return variations;

      },
      getMetaInfo: function(){
        var metaInfo = {}
        this.lines.forEach(function(line){
          line.directives.forEach(function(directive){
            if (directive.type == 'title') metaInfo.title = directive.name;
            if (directive.type == 'subtitle') metaInfo.subtitle = directive.name;
            if (directive.type == 'key') metaInfo.key = directive.name;
            if (directive.type == 'time') metaInfo.time = directive.name;
            if (directive.type == 'tempo') metaInfo.tempo = directive.name;
            if (directive.type == 'duration') metaInfo.duration = directive.name;
          });
        });
        this.metaInfo = metaInfo;
      },
      getLines: function(source){
        var lines = [];
        source.split('\n').forEach(function(line) {
          var directives = this.getDirectives(line);

          line = this.removeDirectives(line);
          var parsedLine = {
            lyricString: this.parseLyricLine(line),
            chordString: this.parseChordLine(line),
            directives: directives
          };
          lines.push(parsedLine);
        }.bind(this));
        return lines;
      },
      getDirectives: function(line){
        var directives = this.parseDirectives(line) !== undefined ? this.parseDirectives(line) : [];
        return directives;
      },
      removeDirectives: function(line){
        var directives = this.parseDirectives(line);
        if (directives !== undefined && directives.length){
          directives.forEach(function(directive){
            line = line.replace(directive.markup, '');
          })
        }
        return line;
      },
      parseDirectives: function(line) {
        var regex = /\{\s*([^:}]*)\s*:{0,1}\s*(.*?)\s*}/g;
        var m;
        var directives = [];
        while ((m = regex.exec(line)) !== null) {
          if (m.index === regex.lastIndex) {
            regex.lastIndex++;
          }
          var directive = {};
          m.forEach((match, groupIndex) => {
            if (groupIndex == 0) {directive.markup = match}
              if (groupIndex == 1) {
                directive.type = this.getDirectiveType(match)
              }
              if (groupIndex == 2) {
                directive.name = match
              }
            });
          directives.push(directive);
        }
        return directives;
      },
      getDirectiveType: function(directiveType) {
        directiveType = directiveType.toLowerCase();
        switch (directiveType) {
          case 't':
          return 'title';
          case 'st':
          return 'subtitle';
          case 'c':
          return 'comment';
          default:
          return directiveType;
        }
      },
      parseLyricLine: function(line){
        var chords = this.getChords(line)
        if (chords.length){
          chords.forEach(function(chord){
            line = line.replace(chord.markup, "");
          });
        }
        return line;
      },
      parseChordLine: function(line){
        var chords = this.getChords(line)
        var index = 0;
        var chordline = "";
        if(chords.length){
          let charactersRemoved = 0;
          while (index < line.length){
            var isChord = false;
            chords.forEach(function(chord){
              if (chord.position == index){
                chordline += chord.name;
                index += chord.markup.length + chord.name.length;
                isChord = true;
              }
            });
            if (!isChord){
              chordline += "&nbsp;";
              index++;
            }
          }
        }else if(line.length){
          chordline = "&nbsp;";
        }
        return chordline;
      },
      getChords: function(line) {
        var regex = /\[([a-zA-Z0-9#+\/]*)\]/g;
        var m;
        var chords = [];
        while ((m = regex.exec(line)) !== null) {
          if (m.index === regex.lastIndex) {
            regex.lastIndex++;
          }
          var chord = {};
          m.forEach(function(match, groupIndex) {
            if (groupIndex == 0) {
              chord.markup = match
            }
            if (groupIndex == 1) {
              chord.name = match
            }
          });
          chord.position = m.index;
          chords.push(chord);
        }
        return chords;
      },
      insertTextAtCursor: function(text) {
        var el = document.getElementById("source-input");
        var val = el.value, endIndex, range;
        endIndex = el.selectionEnd;
        this.source = val.slice(0, el.selectionStart) + text + val.slice(endIndex);
      },
      showModal: function(modalId){
        document.getElementById(modalId).classList.add('active');
      },
      hideModal: function(modalId){
        document.getElementById(modalId).classList.remove('active');
      }
    }
  }
</script>

<style lang="css">
  ul.addChord, ul.activeChord{
    list-style: none;
    padding: 0;
  }

  ul.addChord li{
    display: inline-block;
    border: 1px solid #DDD;
    margin-right: 4px;
    margin-bottom: 4px;
    padding: 4px;
    border-radius: 3px;
    text-align: center;
    width: 34px;
    cursor: pointer;
  }

  ul.addChord li.active{
    background-color: #ddd;
  }
  ul.activeChord li{
    color: #C32A22;
    cursor: pointer;
  }

  .meta-info .meta-info-data span{
    display: block;
  }

  .comment{
    font-style: italic;
    color: #777;
  }

  .chord-string, .lyric-string{
    font-family: "Consolas", "Lucida Console", Monaco, monospace;
  }

  .chord-string{
    color: #C32A22;
  }

  .clearfix{
    clear: both;
  }
  @media (min-width: 768px){
    .source-input{
      width: 40%;
      float: left;
      padding: 16px;
    }
    .source-input .source-input-menu{
      position: relative;
      z-index: 2;
      border-bottom: 1px solid #e3e3e3;
    }
    .source-input .source-input-menu .source-input-menu-item{
      display: block;
      float: left;
    }
    .source-input .source-input-menu .source-input-menu-item .source-input-menu-sub-item{
      position: absolute;
    }
    .source-input textarea{
      border: none;
      padding-top: 40px;
    }
    .lines{
      width: 60%;
      float: left;
      padding: 16px;
    }
    ul.addChord {
      position: fixed;
      z-index: 2;
      width: 40%;
      background: #d6d6df;
      top: 0;
      margin: 0;
      left: 0;
    }
  }
</style>
