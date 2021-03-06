<template>
  <div id="gallery-view">
    <div class="view-title">
      相册 - {{ filterList.length }} <i class="el-icon-caret-bottom" @click="toggleHandleBar" :class="{'active': handleBarActive}"></i>
    </div>
    <transition name="el-zoom-in-center">
      <el-row v-show="handleBarActive">
        <el-col :span="20" :offset="2" :class="{ 'long-list': filterList.length > 4 }">
          <el-row class="handle-bar" :gutter="16">
            <el-col :span="6" v-for="item in $picBed" :key="item.type">
              <div class="pic-bed-item" @click="choosePicBed(item.type)" :class="{active: choosedPicBed.indexOf(item.type) !== -1}">
                {{ item.name }}
              </div>
            </el-col>
          </el-row>
          <el-row class="handle-bar" :gutter="16">
            <el-col :span="12">
              <el-input placeholder="搜索" size="mini" v-model="searchText">
                <i slot="suffix" class="el-input__icon el-icon-close" v-if="searchText" @click="cleanSearch" style="cursor: pointer"></i>
              </el-input>
            </el-col>
            <el-col :span="6">
              <div class="item-base copy" :class="{ active: isMultiple(choosedList)}" @click="multiCopy">
                <i class="el-icon-document"></i> 批量复制
              </div>
            </el-col>
            <el-col :span="6">
              <div class="item-base delete" :class="{ active: isMultiple(choosedList)}" @click="multiDelete">
                <i class="el-icon-delete"></i> 批量删除
              </div>
            </el-col>
          </el-row>
        </el-col>
      </el-row>
    </transition>
    <el-row class="gallery-list" :class="{ small: handleBarActive }">
      <el-col :span="20" :offset="2">
        <el-row :gutter="16">
          <gallerys
            :images="images"
            :index="idx"
            @close="handleClose"
            :options="options"
          ></gallerys>
          <el-col :span="6" v-for="(item, index) in images" :key="item.id" class="gallery-list__img">
            <div 
              class="gallery-list__item"
              v-lazy:background-image="item.imgUrl"
              @click="zoomImage(index)"
            >
            </div>
            <div class="gallery-list__tool-panel">
              <i class="el-icon-document" @click="copy(item.imgUrl)"></i>
              <i class="el-icon-edit-outline" @click="openDialog(item)"></i>
              <i class="el-icon-delete" @click="remove(item.id)"></i>
              <el-checkbox v-model="choosedList[item.id]" class="pull-right" @change=" handleBarActive = true"></el-checkbox>
            </div> 
          </el-col>
        </el-row>
      </el-col>
    </el-row>
    <el-dialog
      :visible.sync="dialogVisible"
      title="修改图片URL"
      width="500px"
      :modal-append-to-body="false"
    >
      <el-input v-model="imgInfo.imgUrl"></el-input>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="confirmModify">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>
<script>
import gallerys from 'vue-gallery'
import pasteStyle from '~/main/utils/pasteTemplate'
export default {
  name: 'gallery',
  components: {
    gallerys
  },
  data () {
    return {
      images: [],
      idx: null,
      options: {
        titleProperty: 'fileName',
        urlProperty: 'imgUrl',
        closeOnSlideClick: true
      },
      dialogVisible: false,
      imgInfo: {
        id: null,
        imgUrl: ''
      },
      choosedList: {},
      choosedPicBed: [],
      searchText: '',
      handleBarActive: false
    }
  },
  created () {
    this.getGallery()
    this.$electron.ipcRenderer.on('updateGallery', (event) => {
      this.$nextTick(() => {
        this.filterList = this.getGallery()
      })
    })
  },
  computed: {
    filterList: {
      get () {
        return this.getGallery()
      },
      set (val) {
        return this.val
      }
    }
  },
  methods: {
    getGallery () {
      if (this.choosedPicBed.length > 0) {
        let arr = []
        this.choosedPicBed.forEach(item => {
          let obj = {
            type: item
          }
          if (this.searchText) {
            obj.fileName = this.searchText
          }
          arr = arr.concat(this.$db.read().get('uploaded').filter(obj => {
            return obj.fileName.indexOf(this.searchText) !== -1 && obj.type === item
          }).reverse().value())
        })
        this.images = arr
      } else {
        if (this.searchText) {
          let data = this.$db.read().get('uploaded')
            .filter(item => {
              return item.fileName.indexOf(this.searchText) !== -1
            }).reverse().value()
          this.images = data
        } else {
          this.images = this.$db.read().get('uploaded').slice().reverse().value()
        }
      }
      return this.images
    },
    zoomImage (index) {
      this.idx = index
      this.changeZIndexForGallery(true)
    },
    changeZIndexForGallery (isOpen) {
      if (isOpen) {
        document.querySelector('.main-content.el-row').style.zIndex = 101
      } else {
        document.querySelector('.main-content.el-row').style.zIndex = 10
      }
    },
    handleClose () {
      this.idx = null
      this.changeZIndexForGallery(false)
    },
    copy (url) {
      const style = this.$db.read().get('settings.pasteStyle').value() || 'markdown'
      const copyLink = pasteStyle(style, url)
      const obj = {
        title: '复制链接成功',
        body: copyLink,
        icon: url
      }
      const myNotification = new window.Notification(obj.title, obj)
      this.$electron.clipboard.writeText(copyLink)
      myNotification.onclick = () => {
        return true
      }
    },
    remove (id) {
      this.$confirm('此操作将把该图片移出相册, 是否继续?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$db.read().get('uploaded').removeById(id).write()
        const obj = {
          title: '操作结果',
          body: '删除成功'
        }
        const myNotification = new window.Notification(obj.title, obj)
        myNotification.onclick = () => {
          return true
        }
        this.getGallery()
      }).catch(() => {
        return true
      })
    },
    openDialog (item) {
      this.imgInfo.id = item.id
      this.imgInfo.imgUrl = item.imgUrl
      this.dialogVisible = true
    },
    confirmModify () {
      this.$db.read().get('uploaded')
        .getById(this.imgInfo.id)
        .assign({imgUrl: this.imgInfo.imgUrl})
        .write()
      const obj = {
        title: '修改图片URL成功',
        body: this.imgInfo.imgUrl,
        icon: this.imgInfo.imgUrl
      }
      const myNotification = new window.Notification(obj.title, obj)
      myNotification.onclick = () => {
        return true
      }
      this.dialogVisible = false
      this.getGallery()
    },
    choosePicBed (type) {
      let idx = this.choosedPicBed.indexOf(type)
      if (idx !== -1) {
        this.choosedPicBed.splice(idx, 1)
      } else {
        this.choosedPicBed.push(type)
      }
    },
    cleanSearch () {
      this.searchText = ''
    },
    isMultiple (obj) {
      return Object.values(obj).some(item => item)
    },
    multiDelete () {
      if (Object.values(this.choosedList).some(item => item)) {
        this.$confirm('将删除刚才选中的图片，是否继续？', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          Object.keys(this.choosedList).forEach(key => {
            if (this.choosedList[key]) {
              this.$db.read().get('uploaded').removeById(key).write()
            }
          })
          this.choosedList = {}
          this.getGallery()
          const obj = {
            title: '操作结果',
            body: '删除成功'
          }
          const myNotification = new window.Notification(obj.title, obj)
          myNotification.onclick = () => {
            return true
          }
        }).catch(() => {
          return true
        })
      }
    },
    multiCopy () {
      if (Object.values(this.choosedList).some(item => item)) {
        let copyString = ''
        const style = this.$db.read().get('settings.pasteStyle').value() || 'markdown'
        Object.keys(this.choosedList).forEach(key => {
          if (this.choosedList[key]) {
            copyString += pasteStyle(style, this.$db.read().get('uploaded').getById(key).value().imgUrl) + '\n'
          }
        })
        const obj = {
          title: '批量复制链接成功',
          body: copyString
        }
        const myNotification = new window.Notification(obj.title, obj)
        this.$electron.clipboard.writeText(copyString)
        myNotification.onclick = () => {
          return true
        }
      }
    },
    toggleHandleBar () {
      this.handleBarActive = !this.handleBarActive
    }
  },
  beforeDestroy () {
    this.$electron.ipcRenderer.removeAllListeners('updateGallery')
  }
}
</script>
<style lang='stylus'>
.view-title
  color #eee
  font-size 20px
  text-align center
  margin 10px auto
  .sub-title
    font-size 14px
  .el-icon-caret-bottom
    cursor: pointer
    transition all .2s ease-in-out
    &.active
      transform: rotate(180deg)
.item-base
  background #2E2E2E
  text-align center
  margin-bottom 10px
  padding 5px 0
  cursor pointer
  font-size 13px
  transition all .2s ease-in-out
  &.copy
    cursor not-allowed
    background #49B1F5
    &.active
      cursor pointer
      background #1B9EF3
      color #fff
  &.delete
    cursor not-allowed
    background #F47466
    &.active
      cursor pointer
      background #F15140
      color #fff
.long-list
  width: calc(83.3333333% - 6px)
#gallery-view
  position relative
  .pull-right
    float right
  .gallery-list
    height 360px
    box-sizing border-box
    padding 8px 0
    overflow-y auto
    overflow-x hidden
    position absolute
    top: 38px
    transition all .2s ease-in-out .1s
    width 100%
    &.small
      height: 245px
      top: 152px
    &__img
      height 150px
      position relative
      margin-bottom 16px
    &__item
      width 100%
      height 120px
      background-size cover
      background-position 50% 50%
      background-repeat no-repeat
      transition all .2s ease-in-out
      cursor pointer
      margin-bottom 8px
      &-fake
        position absolute
        top 0
        left 0 
        opacity 0
        width 100%
        z-index -1
      &:hover
        background-color #49B1F5
        transform scale(1.1)
    &__tool-panel
      color #ddd
      i
        cursor pointer
        transition all .2s ease-in-out
        &.el-icon-document
          &:hover
            color #49B1F5
        &.el-icon-edit-outline
          &:hover
            color #69C282
        &.el-icon-delete
          &:hover
            color #F15140
  .handle-bar
    color #ddd
    .pic-bed-item
      @extend .item-base
      &:hover
        background #A4D8FA
        color #fff
      &.active
        background #49B1F5
        color #fff
  .el-input__inner
    border-radius 0
</style>