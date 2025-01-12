<template>
  <div v-resize="onResize">
    <div id="read" :style="{ width: width + 'px' }"> </div>
    <div ref="viewer" v-viewer v-show="false">
      <img :src="img.src" :alt="img.alt" />
    </div>
    <!-- <div v-show="menuShow || sidebarShow" class="cover" @click="hide"></div> -->
    <div v-show="sidebarShow" class="cover" @click="hide"></div>
    <ebook-menu />
    <ebook-sidebar />
    <font-setting />
    <bg-setting />
  </div>
</template>

<script>
  // import { EpubCFI } from 'epubjs' //这样导不进来，奇怪
  // import Epub from 'epubjs'
  import Epub from '@/assets/js/epub.85.fix'
  import EpubCFI from 'epubjs/src/epubcfi'
  import { mapActions, mapGetters, mapMutations } from 'vuex'
  import { flatten, GetReadProgress, throttle, ImagePath } from '@/util/read'
  import EbookMenu from '@/components/Read/EbookMenu'
  import EbookSidebar from '@/components/Read/EbookSidebar'
  import FontSetting from '@/components/Read/Menu/FontSetting'
  import BgSetting from './Menu/BgSetting'
  import { toByteArray } from 'base64-js'
  import md5 from 'md5'
  import { isMobile as IsMobile } from '@/util'

  export default {
    name: 'EbookReader',
    components: { BgSetting, FontSetting, EbookSidebar, EbookMenu },
    data() {
      return {
        img: {
          src: null,
          alt: null
        },
        rendition: null,
        promise: null,
        width: null,
        enableTouch: false,
        touchDetail: null
      }
    },
    props: {
      name: String
    },
    computed: {
      ...mapGetters(['book', 'menuShow', 'sidebarShow', 'fontSize','bookHash']),
      navigation() {
        return this.$store.state.read.navigation
      },
      isMobile() {
        return IsMobile()
      }
    },
    methods: {
      ...mapMutations([
        'updateBook',
        'updateCover',
        'updateNavigation',
        'updateMetadata',
        'updateBookAvailable',
        'updateMenuShow',
        'updateSidebarShow',
        'updatebookHash',
        'updateFontSize'
      ]),
      ...mapActions(['refreshLocation', 'getRendition']),
      hide() {
        this.updateMenuShow(false)
        this.updateSidebarShow(false)
      },
      show() {
        this.updateMenuShow(true)
      },
      prevPage() {
        if (this.rendition && !this.promise) {
          console.log('上一页')
          this.promise = this.rendition.prev().then(() => {
            this.refreshLocation([true, true])
            setTimeout(() => {
              this.promise = null
            }, 30)
          })
          this.hide()
        }
      },
      nextPage() {
        if (this.rendition && !this.promise) {
          console.log('下一页')
          this.promise = this.rendition.next().then(() => {
            this.refreshLocation([true, true])
            setTimeout(() => {
              this.promise = null
            }, 30)
          })
          this.hide()
        }
      },
      handleKeyDown: throttle(function handleKeyDown(e) {
        switch (e.keyCode) {
          case 37: //left
            this.prevPage()
            break
          case 39: //right
            this.nextPage()
            break
        }
      }, 300),
      handleMouseDown(e) {
        const time = e.timeStamp - this.timeStart
        // if (this.hide()) return
        if (e.target.localName === 'a' || e.target.parentNode.localName === 'a') return
        const path = e.path || e.composedPath()
        let X = 0,
          Y = 0
        if (e.type === 'touchend') {
          X = this.touchDetail.targetTouches[0].pageX % this.width
          Y = this.touchDetail.targetTouches[0].pageY
        } else {
          X = e.pageX % this.width
          Y = e.pageY
        }
        if (e.target.localName === 'img' && path) {
          const classList = [].concat(...path.map((item) => [].concat.apply([], item.classList)))
          if (classList.findIndex((item) => item === 'duokan-image-single') !== -1) {
            if (!this.isInArea(X)) {
              this.previewImg(e)
              return
            }
          }
        }
        if (time < 200) {
          if (X > this.width * 0.75) this.nextPage()
          else if (X < this.width * 0.25) this.prevPage()
          else if (Y < window.innerHeight * 0.75 && Y > window.innerHeight * 0.25) {
            if (this.menuShow) {
              this.hide()
            } else {
              this.show()
            }
          }
        }
      },
      handleMouseWheel(e) {
        if (e.detail) {
          e.detail > 0 ? this.nextPage() : this.prevPage()
        } else {
          e.deltaY > 0 ? this.nextPage() : this.prevPage()
        }
      },
      isInArea(offsetX) {
        return offsetX > this.width * 0.75 || offsetX < this.width * 0.25
      },
      async initEpub(book, cfi) {
        this.updateBook(book)
        // 指定渲染的位置和方式
        this.rendition = await this.getRendition({
          element: 'read',
          option: {
            width: this.width,
            height: window.innerHeight,
            // flow: 'auto',
            flow: 'paginated',
            manager: 'continuous',
            snap: this.isMobile
          }
        })

        this.rendition.display(cfi)

        this.initEvent()
        this.parseBook()
        book.ready
          .then(() => {
            window.device?.setResultOK()
            // 修改网页title
            document.title = book.package.metadata.title
            // return this.book.locations.generate(750 * (window.innerWidth / 375) * bookState.defaultFontSize / 16)
            return book.locations.generate()
          })
          .then(() => {
            // 书籍加载完毕
            this.updateBookAvailable(true)
            this.refreshLocation([true, true])
          })
      },
      initEvent() {
        let vueInstance = this
        const mousewheel = /Firefox/i.test(navigator.userAgent) ? 'DOMMouseScroll' : 'mousewheel'
        this.rendition.hooks.content.register(function (contents) {
          const baseName = contents.cfiBase.match(/\[(.*?)\]/)[1]
          vueInstance.navigation.forEach((navItem) => {
            if (navItem.href.indexOf(baseName) !== -1) {
              const href = navItem.href.match(/\/(.*?)$/)[1]
              let id = undefined
              if (href.indexOf('#') !== -1) {
                id = href.split('#')[1]
              }
              if (id) {
                //得到每个目录的cfi地址
                const node = contents.document.getElementById(id)
                const cfi = new EpubCFI(node, contents.cfiBase)
                navItem.cfi = cfi.toString()
              }
            }
          })
          contents.window.addEventListener(mousewheel, vueInstance.handleMouseWheel, true)
          contents.document.querySelectorAll('.duokan-image-single img').forEach((node) => {
            node.style.boxShadow = 'black 0 0 3px'
            node.style.cursor = 'pointer'
            node.style.border = '1px solid white'
          })
          contents.window.addEventListener('keydown', vueInstance.handleKeyDown)
          if (vueInstance.isMobile) {
            contents.document.addEventListener('touchmove', (e) => {
              if (!vueInstance.enableTouch) {
                vueInstance.enableTouch = true
              }
            })
            contents.document.addEventListener(
              'touchstart',
              (e) => {
                // console.log('touch', e)
                vueInstance.timeStart = e.timeStamp
                vueInstance.touchDetail = e
              },
              true
            )

            contents.document.addEventListener(
              'touchend',
              (e) => {
                // console.log('touchend', e)
                if (!vueInstance.enableTouch) {
                  e.stopPropagation()
                  vueInstance.handleMouseDown(e)
                } else {
                  vueInstance.enableTouch = false
                }
              },
              true
            )
          }
        })
        // 单击事件
        if (!vueInstance.isMobile) {
          this.rendition.on('mousedown', (e) => {
            this.timeStart = e.timeStamp
          })
          this.rendition.on('mouseup', this.handleMouseDown)
        }
        window.addEventListener('keydown', this.handleKeyDown)
      },
      parseBook() {
        this.book.loaded.cover.then((cover) => {
          if (window.device) {
            if (!window.device.fileExits(ImagePath + '/' + this.bookHash)) {
              this.book.archive.getBase64(cover || '/OEBPS/Images/cover.jpg').then((data) => {
                console.log('saveFile')
                window.device.saveFile(this.bookHash, 'Pictures', data)
              })
            }
          }
          this.book.archive.createUrl(cover || '/OEBPS/Images/cover.jpg').then((url) => {
            this.updateCover(url)
          })
        })
        this.book.loaded.metadata.then((metadata) => {
          this.updateMetadata(metadata)
        })
        this.book.loaded.navigation.then((nav) => {
          const navItem = flatten(nav.toc)

          function find(item, levle = 0) {
            return !item.parent
              ? levle
              : find(navItem.filter((parentItem) => parentItem.id === item.parent)[0], ++levle)
          }

          let basePath = this.book.package.navPath ? this.book.package.navPath.split('/') : false
          navItem.forEach((item) => {
            item.level = find(item)
            item.label = item.label.trim()

            if (basePath) {
              if (item.href.startsWith('#')) {
                item.href = this.book.package.navPath + item.href
              } else {
                basePath[basePath.length - 1] = item.href
                item.href = basePath.join('/')
              }
            }
          })
          this.updateNavigation(navItem)
        })
      },
      previewImg(event) {
        this.img.src = event.target.src
        this.img.alt = event.target.alt
        this.$refs.viewer.$viewer.show()
      },
      onResize() {
        if (this.rendition) {
          this.getWidth()
          this.rendition.settings.width = this.width
          this.rendition.resize(this.width, window.innerHeight)
        }
      },
      getWidth() {
        // 根据文档，在使用显示比例缩放的系统上，scrollLeft可能会为您提供一个十进制值。
        // 这导致了可能错误的移动位置
        // 这里将显示的宽度限定为8的倍数来解决问题
        const screenWidth = Math.round(window.innerWidth)
        const remainder = screenWidth % 8
        this.width = screenWidth - remainder
      }
    },
    destroyed() {
      window.removeEventListener('keydown', this.handleKeyDown)
    },
    async mounted() {
      this.getWidth()
      if (window.device) {
        // 连接App调试
        let vueInstance = this
        window.loadBook = async function (path, data) {
          let hash = md5(path)
          vueInstance.updatebookHash(hash)
          data = toByteArray(data)
          console.log(data.length)
          let book = new Epub()
          await book.open(data.buffer)
          vueInstance.initEpub(book, GetReadProgress(hash))
        }
        window.device.readBook()
      } else {
        const fileName = 'Test2.epub'
        this.updatebookHash(fileName)
        this.initEpub(new Epub(fileName), GetReadProgress(fileName))
      }
    }
  }
</script>

<style scoped lang="scss">
  #read {
    margin: 0 auto;
  }

  .cover {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
  }
</style>
