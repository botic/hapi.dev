<template>
  <div class="container">
    <FamilyNav
      :moduleAPI="moduleAPI"
      :menu="getMenu"
      :version="getVersion"
      :versions="versionsArray"
      :results="results"
      :indexResults="indexResults"
      :search="search"
      @search="onChildSearch"
      @previous="onChildIndex"
      @next="onChildIndex"
      @input="onChildInput"
    />
    <div class="tutorial-markdown-window">
      <Install :name="name" :moduleAPI="moduleAPI" :version="version" />
      <FamilyDisplay :display="getDisplay" />
      <Changelog :milestones="getMilestones" />
    </div>
    <div class="preload">
      <img src="/img/clipboardCheck.png" alt="clipboard" />
    </div>
  </div>
</template>

<script>
import FamilyDisplay from '~/components/family/FamilyDisplay.vue';
import FamilyNav from '~/components/family/FamilyNav.vue';
import Install from '~/components/family/Install.vue';
import Changelog from '../../components/resources/Changelog.vue';
let Toc = require('markdown-toc');
let Semver = require('semver');

export default {
  components: {
    FamilyDisplay,
    FamilyNav,
    Install,
    Changelog
  },
  head() {
    return {
      title: 'hapi.dev - ' + this.$route.params.family + ' v' + this.getVersion,
      meta: [
        {
          hid: 'description',
          name: 'description',
          content: 'View the APIs for the hapi modules'
        }
      ]
    };
  },
  data() {
    return {
      display: '',
      modules: this.modules,
      version: '',
      menu: '',
      name: this.$route.params.family,
      indexResults: 0,
      search: '',
      results: [],
      uls: {},
      links: {}
    };
  },
  methods: {
    goToAnchor() {
      let hash = document.location.hash;
      if (hash != '') {
        setTimeout(function() {
          if (location.hash) {
            window.scrollTo(0, 0);
            window.location.href = hash;
          }
        }, 1);
      } else {
        return false;
      }
    },
    onChildSearch() {
      let headlines = [];
      let text = [];
      this.indexResults = 0;
      const headers = ['H2', 'H3', 'H4', 'H5', 'H6'];
      let pages = document.querySelector('.family-markdown-wrapper').querySelectorAll('*');

      //Check if search item is in a headline
      for (let page of pages) {
        if (headers.indexOf(page.nodeName) !== -1 && page.innerHTML.indexOf(this.search.toLowerCase()) !== -1) {
          headlines.push(page);
        } else if (headers.indexOf(page.nodeName) === -1 && page.innerHTML.indexOf(this.search.toLowerCase()) !== -1) {
          text.push(page);
        }
      }

      this.results = headlines.concat(text);
      if (this.results.length) {
        document.querySelector('.family-search-results').classList.add('nav-display');
        if (window.innerWidth <= 900) {
          document.body.scrollTo(0, this.results[this.indexResults].offsetTop + 166);
        } else {
          window.scrollTo(0, this.results[this.indexResults].offsetTop);
        }
      } else if (this.results.length === 0) {
        document.querySelector('.family-search-error').classList.add('nav-display');
      }
    },
    onChildIndex(value) {
      this.$data.indexResults = value;
      window.scrollTo(0, this.results[this.indexResults].offsetTop);
    },
    onChildInput(value) {
      this.$data.search = value;
    },
    setClipboards() {
      let headers = document.querySelectorAll(
        '.family-markdown-wrapper h2, .family-markdown-wrapper h3, .family-markdown-wrapper h4, .family-markdown-wrapper h5, .changelog-wrapper h2'
      );

      for (let header of headers) {
        header.classList.add('api-doc-header');
        header.innerHTML =
          header.innerHTML + "<span class='api-clipboardCheck api-clipboard' title='Copy link to clipboard'></span>";
      }

      let clipboards = document.querySelectorAll('.api-clipboard');

      for (let clipboard of clipboards) {
        clipboard.addEventListener('click', function(event) {
          let copyLink = clipboard.parentNode.firstElementChild.href;
          const el = document.createElement('textarea');
          el.value = copyLink;
          document.body.appendChild(el);
          el.select();
          document.execCommand('copy');
          document.body.removeChild(el);
          clipboard.classList.remove('api-clipboard');
          clipboard.classList.add('api-clipboardCheck');
          setTimeout(function() {
            clipboard.classList.add('api-clipboard');
            clipboard.classList.remove('api-clipboardCheck');
          }, 3000);
        });
      }
    },
    setClasses() {
      //Set TOC classes
      let anchors = document.querySelectorAll('.family-nav-select-wrapper a');
      let code = document.querySelectorAll('.family-nav-select-wrapper a code');

      for (let link of anchors) {
        link.classList.add('family-anchor');
        this.links[link.hash] = link.getBoundingClientRect().top;
        link.addEventListener('click', function(event) {
          let currentActive = document.querySelector('.family-active');
          if (currentActive) {
            currentActive.classList.remove('family-active');
          }
          link.classList.add('family-active');
          if (
            link.parentElement.children[1] &&
            link.parentElement.children[1].classList.contains('family-ul-display')
          ) {
            link.parentElement.children[1].classList.remove('family-ul-display');
            link.classList.remove('family-minus');
            link.classList.add('family-plus');
          } else if (
            link.parentElement.children[1] &&
            !link.parentElement.children[1].classList.contains('family-ul-display')
          ) {
            link.parentElement.children[1].classList.add('family-ul-display');
            link.classList.remove('family-plus');
            link.classList.add('family-minus');
          }
        });
      }

      for (let c of code) {
        c.classList.add('family-code');
      }

      let familyUls = document.querySelectorAll('.family-nav-select-wrapper > ul ul');

      for (let ul of familyUls) {
        this.uls[ul.getBoundingClientRect().top] = {
          name: ul,
          top: ul.getBoundingClientRect().top,
          bottom: ul.getBoundingClientRect().bottom
        };
      }

      let links = document.querySelectorAll('#' + this.$route.params.family + ' a');
      let points = {};
      let offsets = [];
      for (let i = 0; i < links.length; i++) {
        let point = document.querySelector(
          `.tutorial-markdown-window h1 a[href*='${links[i].href.replace(/^[^_]*#/, '')}'],
          .tutorial-markdown-window h2 a[href*='${links[i].href.replace(
            /^[^_]*#/,
            ''
          )}'], .tutorial-markdown-window h3 a[href*='${links[i].href.replace(
            /^[^_]*#/,
            ''
          )}'], .tutorial-markdown-window h4 a[href*='${links[i].href.replace(
            /^[^_]*#/,
            ''
          )}'], .tutorial-markdown-window h5 a[href*='${links[i].href.replace(/^[^_]*#/, '')}']`
        );
        if (point && point.id !== 'changelog') {
          if (point.id) {
            points[point.offsetTop + 220] = {
              name: '#' + point.id
            };
          } else {
            points[point.offsetTop + 220] = {
              name: point.hash
            };
          }

          offsets.push(point.offsetTop + 220);
        }
        if (point.id === 'changelog') {
          points[point.getBoundingClientRect().top] = {
            name: '#' + point.id
          };
          offsets.push(point.getBoundingClientRect().top);
        }
      }
      offsets = [...new Set(offsets)];

      let currentElement = document.querySelector('.markdown-wrapper');

      for (let ul of familyUls) {
        ul.parentNode.children[0].classList.remove('family-minus');
        ul.parentNode.children[0].classList.add('family-plus');
        ul.classList.add('family-hide');
      }

      let that = this;

      //Add active class to elements on scroll
      window.onscroll = function() {
        let location = document.documentElement.scrollTop;
        let locationBody = document.body.scrollTop;
        let actives = document.getElementsByClassName('family-active');
        let active;
        let element;
        let i = 0;
        if (window.innerHeight + window.scrollY < document.body.offsetHeight + 96) {
          for (i in offsets) {
            if (offsets[i] <= location || offsets[i] <= locationBody) {
              let aClass = points[offsets[i]].name;
              for (let active of actives) {
                active.classList.remove('family-active');
              }
              element = document.querySelector(`.api-nav-wrapper a[href='${aClass}']`);
              if (element && element.children.length !== 0) {
                document.querySelector(`a[href*='${aClass}']`).classList.add('family-active');
                active = document.querySelector('.family-active');
              } else if (element && element.children.length === 0) {
                document.querySelector(`a[href*='${aClass}']`).classList.add('family-active');
                active = document.querySelector('.family-active');
              }
            }
          }
        }

        if (active) {
          let activeClass;
          activeClass = active.hash;
          let activeLink = document.querySelector(`a[href*='${activeClass}']`);
          let activePosition = that.links[activeLink.hash];
          for (let key in that.uls) {
            if (activePosition >= that.uls[key].top && activePosition < that.uls[key].bottom) {
              that.uls[key].name.classList.add('family-ul-display');
              that.uls[key].name.parentElement.children[0].classList.remove('family-plus');
              that.uls[key].name.parentElement.children[0].classList.add('family-minus');
            }
          }
          let bottom = active.getBoundingClientRect().bottom;
          let location = active.getBoundingClientRect();
          if (bottom > window.innerHeight || location.top < 131) {
            active.scrollIntoView(false);
          }
          if (that.$route.hash === active.hash && bottom === 0) {
            active.scrollIntoView(false);
          }
        }
      };
    }
  },
  computed: {
    getDisplay() {
      return this.moduleAPI[this.$route.params.family].displays[this.getVersion];
    },
    getVersion() {
      return this.$store.getters.loadVersion;
    },
    getMenu() {
      return this.moduleAPI[this.$route.params.family].menus[this.getVersion];
    },
    getMilestones() {
      return this.milestoneList;
    }
  },
  async asyncData({ params, $axios, route, store }) {
    const options = {
      headers: {
        accept: 'application/vnd.github.v3.raw+json',
        authorization: 'token ' + process.env.GITHUB_TOKEN
      }
    };
    let moduleAPI = {};
    moduleAPI[params.family] = {
      menus: {},
      displays: {},
      versions: {},
      license: {}
    };
    let version = '';
    let license = '';
    let versionsArray = [];
    let milestoneList = [];
    let m = [];
    let milestones = [];

    if (store.getters.loadModules.includes(params.family)) {
      try {
        let branches = await $axios.$get('https://api.github.com/repos/hapijs/' + params.family + '/branches', options);

        for (let branch of branches) {
          if (branch.name.match(/^v+[0-9]+|\bmaster\b/g)) {
            const apiPackage = await $axios.$get(
              'https://api.github.com/repos/hapijs/' + params.family + '/contents/package.json?ref=' + branch.name,
              options
            );
            if (!versionsArray.includes(apiPackage.version)) {
              moduleAPI[params.family].versions[apiPackage.version] = branch.name;
              await versionsArray.push(apiPackage.version);
            }
          }
        }

        for (let apiVersion of versionsArray) {
          const api = await $axios.$get(
            'https://api.github.com/repos/hapijs/' +
              params.family +
              '/contents/API.md?ref=' +
              moduleAPI[params.family].versions[apiVersion],
            options
          );

          let rawString = await api.toString();

          //Auto generate TOC
          let apiTocString = '';
          let apiTocArray = await rawString.match(/\n#.+/g);
          let pattern = '####'

          for (let i = 0; i < apiTocArray.length; ++i) {
            let testPattern = apiTocArray[i].match(/(?=#)(.*)(?=\s)/);
            if (testPattern[0].length < pattern.length) {
              pattern = testPattern[0]
            }
            apiTocString = apiTocString + apiTocArray[i];
          }
          let test = apiTocArray[0];
          apiTocString = apiTocString + '\n' + pattern + ' Changelog';
          let finalMenu = Toc(apiTocString, { bullets: '-' }).content;

          //Split API menu from content
          let finalDisplay = await rawString.replace(/\/>/g, '></a>');
          finalMenu = await finalMenu.replace(/Boom\./g, '');
          finalMenu = await finalMenu.replace(/\(([^#\*]+)\)/g, '()');
          const apiHTML = await $axios.$post(
            'https://api.github.com/markdown',
            {
              text: finalDisplay,
              mode: 'markdown'
            },
            {
              headers: {
                authorization: 'token ' + process.env.GITHUB_TOKEN
              }
            }
          );
          let apiString = await apiHTML.toString();
          let finalHtmlDisplay = await apiString.replace(/user-content-/g, '');
          moduleAPI[params.family].menus[apiVersion] = await finalMenu;
          moduleAPI[params.family].displays[apiVersion] = await finalHtmlDisplay;
        }
      } catch (err) {
        console.log("GITHUB ERRRORRRRRRRR!!!", err.message);
      }

      //Changelog
      const mileOptions = {
        headers: {
          accept: 'application/vnd.github.v3.raw+json',
          authorization: 'token ' + process.env.GITHUB_TOKEN
        }
      };
      for (let p = 1; p <= 2; ++p) {
        milestones = await $axios.$get(
          'https://api.github.com/repos/hapijs/' + params.family + '/milestones?state=closed&per_page=100&page=' + p,
          mileOptions
        );
        await m.push(milestones);
      }

      let flatM = await [].concat(...m);

      let sortedMilestones = await flatM.sort(function(a, b) {
        if (!Semver.valid(a.title)) {
          a.title = Semver.clean(a.title + '.0', { loose: true });
        }
        return Semver.compare(b.title, a.title);
      });

      //Get milestone issues
      for (let milestone of sortedMilestones) {
        let m = await $axios.$get(
          'https://api.github.com/repos/hapijs/' +
            params.family +
            '/issues?state=closed&milestone=' +
            milestone.number +
            '&per_page=200',
          mileOptions
        );
        if (m.length > 0) {
          milestoneList.push(m);
        }
      }
    }

    versionsArray = await versionsArray.sort((a, b) => Semver.compare(b, a));

    return { moduleAPI, version, versionsArray, license, milestoneList };
  },
  created() {
    if (!this.$store.getters.loadModules.includes(this.$route.params.family)) {
      return this.$nuxt.error({ statusCode: 404 });
    }
    let version = this.versionsArray.includes(this.$route.query.v) ? this.$route.query.v : this.versionsArray[0];
    this.$store.commit('setDisplay', 'family');
    this.$store.commit('setVersion', version);
    (!this.$route.query.v || !this.versionsArray.includes(this.$route.query.v)) &&
      this.$router.push({
        path: this.$route.path,
        query: { v: this.versionsArray[0] },
        hash: this.$route.hash
      });
    this.$data.menu = this.moduleAPI[this.$route.params.family].menus[this.getVersion];
  },
  mounted() {
    this.setClasses();
    this.goToAnchor();
    this.setClipboards();
  }
};
</script>

<style lang="scss">
@import '../../assets/styles/main.scss';
@import '../../assets/styles/api.scss';
@import '../../assets/styles/markdown.scss';

.family-title {
  margin: 20px 0 -16px 100px;
  padding-bottom: 16px;
  box-sizing: border-box;
  border-bottom: 1px solid $dark-white;
  display: inline-block;
}

.family-anchor {
  display: inline-block;
  color: $gray;
  font-size: 0.85em;
  height: 100%;
  width: 100%;
  padding: 2px 0;
}

.family-anchor:hover {
  color: $gray;
}

.family-code {
  background: $off-white;
  color: $gray;
  font-family: 'Lato', sans-serif;
  font-size: 1em;
  padding: 0;
  border: none;
}

.family-plus,
.family-minus,
.family-plus code,
.family-minus code {
  position: relative;
  color: $orange;
  text-decoration: none;
}

.family-plus:hover,
.family-minus:hover {
  color: $orange;
}

.family-plus:after {
  content: '\002B';
  display: flex;
  justify-content: center;
  align-items: center;
  position: absolute;
  font-size: 20px;
  top: 0;
  bottom: 0;
  left: -17px;
  height: 31px;
  width: 15px;
  z-index: 100;
}

.family-minus:after {
  content: '\2212';
  color: inherit;
  display: flex;
  justify-content: center;
  align-items: center;
  position: absolute;
  font-size: 20px;
  top: 0;
  bottom: 0;
  left: -17px;
  height: 31px;
  width: 15px;
  z-index: 100;
}

.family-active,
.family-active * {
  position: relative;
  color: #fff !important;
  background: $gray !important;
  height: 31px;
}

.family-active {
  display: inline-block;
  left: -70px;
  padding: 2px 30px 2px 70px !important;
  width: 402px !important;
  overflow-wrap: break-word;
}

.family-active:after {
  position: absolute;
  left: 53px;
  height: 31px;
}

.family-hide {
  display: none;
}

.family-ul-display {
  display: block !important;
}

.changelog-header {
  margin: 20px 0 10px 0;
  padding-top: 10px;
  border-top: 1px solid #ddd;
  width: 100%;
}

h1 a {
  display: block;
  position: relative;
  top: -116px;
  visibility: hidden;
}

.preload {
  display: none;
}

.api-doc-header {
  position: relative;
}

.api-clipboardCheck {
  position: relative;
  display: inline-block;
  width: 17px;
  height: 17px;
  margin: 0 0 0 5px;
  opacity: 0.7;
  background: url('/img/clipboardCheck.png');
  background-size: contain;
  transition: all 0.2s;
}

.api-clipboard {
  position: relative;
  display: inline-block;
  width: 17px;
  height: 17px;
  margin: 0 0 0 5px;
  background: url('/img/clipboard.png');
  background-size: contain;
  opacity: 0.3;
  cursor: pointer;
  transition: all 0.2s;
}

.api-clipboard:hover {
  opacity: 0.7;
}

@media screen and (max-width: 900px) {
  .ecosystem-title {
    margin: 20px 0 0 0;
    padding-bottom: 16px;
  }
}
</style>
