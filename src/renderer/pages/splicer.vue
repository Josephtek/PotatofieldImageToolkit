<template>
  <el-tabs type="card" tab-position="left" id="splicer" @tab-click="clear">
    <el-tab-pane>
      <span slot="label" class="interactable"><i class="fas fa-image"></i> 导入图片</span>
      <div id="file-input" class="tab-content">
        <el-upload
          id="upload-dragger"
          action=""
          class="interactable"
          drag
          multiple
          :auto-upload="false"
          :on-change="handleFile"
          :show-file-list="false"
          :class="this.$store.state.splicer.fileList.length != 0 ? 'half' : ''">
          <i class="fas fa-image"></i>
          <div class="el-upload__text">将图片拖到此处，或<em>点击选择图片</em></div>
        </el-upload>
        <div v-if="this.$store.state.splicer.fileList.length != 0" id="file-list" class="interactable">
          <div id="list">
            <div
              v-for="(file, index) in this.$store.state.splicer.fileList"
              :key="index"
              class="file"
              @click="preview(index)">
              <div class="filename">{{ file.filename + '.' + file.ext }}</div>
              <div @click.stop="handleDelete(index)">
                <i class="fas fa-trash-alt delete"></i>
              </div>
            </div>
          </div>
          <div class="row">
            <el-button type="primary" size="mini" @click="clearConfirm" class="half-width-button interactable">清空列表</el-button>
            <el-button type="primary" size="mini" @click="edit" class="half-width-button interactable">进入拼图编辑器</el-button>
          </div>
        </div>
      </div>
    </el-tab-pane>
    <el-tab-pane>
      <span slot="label" class="interactable"><i class="fas fa-images"></i> 拼图模板库</span>
      <div id="templates" class="tab-content">
        <div id="container" v-if="this.$store.state.splicer.templates.length != 0">
          <div
            v-for="(template, index) in this.$store.state.splicer.templates.slice(templateListPage * 6 - 6, templateListPage * 6)"
            :key="index"
            class="template-container">
            <el-card class="card interactable">
              <div class="row">
                <div class="subtitle">{{ template.title }}</div>
              </div>
              <div class="text">外框宽度：{{ template.padding != 0 ? template.padding : '无外框' }}</div>
              <div class="text">图片间距：{{ template.spacing != 0 ? template.spacing : '无间距' }}</div>
              <v-clamp autoresize :max-lines="2" class="text">{{ template.text }}</v-clamp>
              <div class="row control-buttons">
                <div class="control-button interactable" @click="editTemplate(index + (templateListPage - 1) * 6)">
                  <span class="fa fa-edit"></span>
                  <div>编辑</div>
                </div>
                <div class="control-button interactable" @click="shareTemplate(index + (templateListPage - 1) * 6)">
                  <span class="fa fa-share-alt"></span>
                  <div>分享</div>
                </div>
                <div class="control-button interactable" @click="deleteTemplate(index + (templateListPage - 1) * 6)">
                  <span class="fa fa-trash-alt"></span>
                  <div>删除</div>
                </div>
              </div>
            </el-card>
          </div>
        </div>
        <div v-else id="empty-container">
          <div id="empty">
            <i class="far fa-folder-open"></i>
            <div>尚无已保存的模板</div>
          </div>
        </div>
        <div class="row">
          <el-pagination
            v-if="this.$store.state.splicer.templates.length > 6"
            class="interactable"
            small
            background
            layout="prev, pager, next"
            :pager-count="5"
            :page-size="6"
            :total="this.$store.state.splicer.templates.length"
            :current-page="templateListPage"
            :hide-on-single-page="true"
            @current-change="templateListPageChange">
          </el-pagination>
          <el-button type="primary" size="mini" @click="importTemplate" class="half-width-button interactable">导入拼图模板</el-button>
          <el-button type="primary" size="mini" @click="createTemplate" class="half-width-button interactable">创建拼图模板</el-button>
        </div>
      </div>
    </el-tab-pane>
    <el-tab-pane disabled>
      <span slot="label" id="control-button-holder">
        <div class="control-button interactable" @click="hide">
          <i class="fas fa-angle-double-down"></i>
          <div>最小化</div>
        </div>
        <div class="control-button interactable" @click="close">
          <span class="fas fa-sign-out-alt"></span>
          <div>退出</div>
        </div>
      </span>
    </el-tab-pane>
  </el-tabs>
</template>

<script>
const { ipcRenderer, clipboard } = require('electron')
const ReadDirectory = require('../utils/ReadDirectory')
const path = require('path')

import EXIF from 'exif-js'

export default {
  name: 'splicer',
  data () {
    return {
      errorList: [],
      templateListPage: 1,
      errorLog: null,
      templateTitle: ''
    }
  },
  methods: {
    hide() {
      ipcRenderer.send('minimize')
    },
    close() {
      ipcRenderer.send('close')
      this.$store.dispatch('splicer/fileListEmpty')
      this.$destroy()
    },
    clear() {
      this.$store.dispatch('splicer/fileListEmpty')
      this.errorList = []
      this.errorLog = null
    },
    clearConfirm() {
      this.$dialog({
        type: 'warning',
        title: '操作确认',
        text: '将清除您已导入的图片，确定执行操作吗？',
        showCancel: true,
        confirmFunction: () => {
          this.clear()
        }
      })
    },
    handleFile(file) {
      let ext = file.name.substring(file.name.lastIndexOf(".") + 1, file.name.length).toLowerCase()
      let filename = file.name.substring(0, file.name.lastIndexOf("."))
      let filepath = path.dirname(file.raw.path)
      if (['jpg', 'jpeg', 'png'].indexOf(ext) != -1) {
        this.$store.dispatch('splicer/fileListPush', {
          fullpath: file.raw.path,
          filename: filename,
          ext: ext
        })
      } else {
        this.errorList.push(filename + '.' + ext)
        if (this.errorLog) {
          this.errorLog.change({
            content: this.$createElement('div', null, this.errorList.map((filename) => {
              return this.$createElement('p', {
                style: {
                  'line-height': '24px',
                  'font-size': '12px',
                  'width': '100%',
                  'overflow': 'hidden',
                  'text-overflow': 'ellipsis',
                  'white-space': 'nowrap',
                  'text-indent': '0'
                }
              }, filename)
            }))
          })
        } else {
          this.errorLog = this.$dialog({
            type: 'warning',
            title: '部分图片导入失败',
            text: '下列图片导入失败，请您检查文件格式。但已导入的图片不受影响，您仍可以继续处理列表中显示的已导入图片。',
            content: this.$createElement('div', null, this.errorList.map((filename) => {
              return this.$createElement('p', {
                style: {
                  'line-height': '24px',
                  'font-size': '12px',
                  'width': '100%',
                  'overflow': 'hidden',
                  'text-overflow': 'ellipsis',
                  'white-space': 'nowrap',
                  'text-indent': '0'
                }
              }, filename)
            })),
            confirmFunction: () => {
              this.errorList = []
              this.errorLog = null
            }
          })
        }
      }
    },
    handleDelete(index) {
      this.$store.dispatch('splicer/fileListDelete', index)
    },
    preview(index) {
      let dialog = this.$dialog({
        title: '图像预览',
        text: '正在生成预览',
        showConfirm: false
      })
      let url = this.$store.state.splicer.fileList[index].fullpath
      let image = document.createElement('img')
      image.src = url
      image.onload = () => {
        EXIF.getData(image, () => {
          EXIF.getAllTags(image)
          let orientation = EXIF.getTag(image, 'Orientation')
          if (orientation == 3 || orientation == 6 || orientation == 8) {
            let canvas = document.createElement('canvas')
            let width, height, x, y, rotation
            if (orientation == 3) {
              width = image.width
              height = image.height
              x = -width
              y = -height
              rotation = 180
            } else if (orientation == 6) {
              width = image.height
              height = image.width
              x = 0
              y = -width
              rotation = 90
            } else {
              width = image.height
              height = image.width
              x = -height
              y = 0
              rotation = 270
            }
            canvas.height = height
            canvas.width = width
            let context = canvas.getContext("2d")
            context.rotate(rotation * Math.PI / 180)
            context.drawImage(image, x, y)
            dialog.change({
              title: '图像预览',
              text: '',
              showConfirm: true,
              content: this.$createElement('img', {
                'attrs': {
                  'id': 'preview-image'
                }
              }),
              onShowFunction: () => {
                canvas.style.width = '100%'
                canvas.style.display = 'block'
                let previewImage = document.getElementById('preview-image')
                previewImage.parentNode.replaceChild(canvas, previewImage)
              }
            })
          } else {
            dialog.change({
              title: '图像预览',
              text: '',
              showConfirm: true,
              content: this.$createElement('img', {
                'attrs': {
                  'src': url
                }
              })
            })
          }
        })
      }
    },
    templateListPageChange(page) {
      this.templateListPage = page
    },
    edit() {
      ipcRenderer.send('open', {
        title: '拼图编辑器',
        path: '#/splicer/editor',
        modal: true,
        height: 720,
        width: 1000
      })
    },
    editTemplate(index) {
      ipcRenderer.send('open', {
        title: '拼图模板编辑器',
        path: '#/splicer/template?index=' + String(index),
        modal: true,
        height: 600,
        width: 1000
      })
    },
    shareTemplate(index) {
      clipboard.writeText(btoa(encodeURI(JSON.stringify({
        type: 'splicerTemplate',
        content: this.$store.state.splicer.templates[index]
      }))))
      this.$dialog({
        type: 'success',
        title: '成功',
        text: '已成功将拼图模板复制到剪贴板。'
      })
    },
    deleteTemplate(index) {
      this.$dialog({
        type: 'warning',
        title: '操作确认',
        text: '确定要删除这个模板吗？',
        showCancel: true,
        confirmFunction: () => {
          this.$store.dispatch('splicer/templateDelete', index)
        }
      })
    },
    importTemplate() {
      try {
        let template = JSON.parse(decodeURI(atob(clipboard.readText())))
        if (template.type != 'splicerTemplate') {
          throw false
        } else {
          template = template.content
          let checkName = (title) => {
            if (title == '') {
              this.$dialog({
                type: 'error',
                title: '错误',
                text: '请输入模板标题，否则无法导入该模板。',
                showCancel: true,
                confirmFunction: () => {
                  this.$dialog({
                    title: '请输入拼图模板标题',
                    content: this.$createElement('div', {
                      'class': 'el-input el-input--mini'
                    }, [
                      this.$createElement('input', {
                        'dom-props': {
                          value: this.templateTitle,
                        },
                        'on': {
                          input: (event) => {
                            this.templateTitle = event.target.value
                          }
                        },
                        'class': 'el-input__inner',
                        'style': {
                          'font-family': 'var(--main-font)'
                        }
                      })
                    ]),
                    showCancel: true,
                    confirmFunction: () => {
                      checkName(this.templateTitle)
                      this.templateTitle = ''
                    },
                    cancelFunction: () => {
                      this.templateTitle = ''
                    }
                  })
                }
              })
            } else {
              for (let i = 0; i < this.$store.state.splicer.templates.length; i++) {
                if (title == this.$store.state.splicer.templates[i].title) {
                  this.$dialog({
                    type: 'warning',
                    title: '存在同名模板',
                    text: '您需要将新导入的模板重命名，才能将其保存。',
                    showCancel: true,
                    confirmFunction: () => {
                      this.$dialog({
                        title: '请输入拼图模板标题',
                        content: this.$createElement('div', {
                          'class': 'el-input el-input--mini'
                        }, [
                          this.$createElement('input', {
                            'dom-props': {
                              value: this.templateTitle,
                            },
                            'on': {
                              input: (event) => {
                                this.templateTitle = event.target.value
                              }
                            },
                            'class': 'el-input__inner',
                            'style': {
                              'font-family': 'var(--main-font)'
                            }
                          })
                        ]),
                        showCancel: true,
                        confirmFunction: () => {
                          checkName(this.templateTitle)
                          this.templateTitle = ''
                        },
                        cancelFunction: () => {
                          this.templateTitle = ''
                        }
                      })
                    }
                  })
                  return
                }
              }
              template.title = title
              this.$store.dispatch('splicer/templatePush', template)
              this.$dialog({
                type: 'success',
                title: '成功',
                text: '拼图模板导入成功。'
              })
            }
          }
          checkName(template.title)
        }
      } catch (e) {
        this.$dialog({
          type: 'error',
          title: '导入失败',
          text: '未能从您的剪贴板中读取到拼图模板信息！'
        })
      }
    },
    createTemplate() {
      ipcRenderer.send('open', {
        title: '拼图模板编辑器',
        path: '#/splicer/template?index=-1',
        modal: true,
        height: 600,
        width: 1000
      })
    }
  }
}
</script>

<style lang="scss">
#splicer {
  width: 100%;
  height: 100%;
  
  button {
    font-family: var(--main-font);
  }
  
  .el-tabs__header {
    margin-right: 0;
    
    .el-tabs__nav-scroll {
      background-color: var(--dark-gray);
      
      .el-tabs__nav {
        border: 0;
        height: 100%;
        display: flex;
        flex-direction: column;
        
        .el-tabs__item {
          width: 150px;
          height: 50px;
          line-height: 50px;
          color: var(--light-gray);
          text-align: center;
          border: 0;
          transition: 0.2s;
          
          #control-button-holder {
            width: 100%;
            display: flex;
            justify-content: space-between;
            align-items: center;
            padding: 20px;
            box-sizing: border-box;
            
            .control-button {
              font-size: 12px;
              line-height: initial;
              cursor: pointer;
              transition: 0.2s;
              
              svg {
                font-size: 20px;
                margin: 5px;
              }
              
              &:hover {
                color: var(--white);
              }
              
              &:active {
                filter: brightness(0.9);
              }
            }
          }
          
          &.is-active {
            background-color: var(--white);
            color: var(--main-color);
            cursor: default;
          }
          
          &.is-disabled {
            flex-grow: 1;
            padding: 0;
            display: flex;
            align-items: flex-end;
          }
          
          &:hover:not(.is-disabled):not(.is-active) {
            color: var(--white);
          }
          
          &:active:not(.is-disabled):not(.is-active) {
            filter: brightness(0.9);
          }
        }
      }
    }
  }
  
  .el-tabs__content {
    height: 100%;
  }
  
  .el-tab-pane {
    width: 100%;
    height: 100%;
  }
  
  .tab-content {
    width: 100%;
    height: 100%;
    padding: 20px;
    box-sizing: border-box;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  
  .row {
    width: 100%;
    flex-shrink: 0;
    margin-top: 10px;
    margin-bottom: 10px;
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    align-items: center;
    
    .controller {
      width: 60%;
    }
    
    &:first-child {
      margin-top: 0;
    }
    
    &:last-child {
      margin-bottom: 0;
    }
  }
  
  .half-width-button {
    width: calc(50% - 5px);
  }
    
  #file-input {
    flex-direction: row;
    
    #upload-dragger {
      width: 100%;
      height: 100%;
      transition: 0.5s;
      
      &.half {
        width: calc(50% - 5px);
      }
      
      .el-upload {
        width: 100%;
        height: 100%;
        
        .el-upload-dragger {
          width: 100%;
          height: 100%;
          display: flex;
          flex-direction: column;
          justify-content: center;
          align-items: center;
          
          svg {
            font-size: 40px;
            margin: 14px;
          }
        }
      }
    }
    
    #file-list {
      width: calc(50% - 5px);
      height: 100%;
      display: flex;
      flex-direction: column;
      justify-content: space-between;
      
      #list {
        width: 100%;
        flex-grow: 1;
        background-color: var(--white-gray);
        box-sizing: border-box;
        border-radius: 6px;
        border-color: var(--light-gray);
        border-style: solid;
        border-width: 1px;
        overflow-y: auto;
        overflow-x: hidden;
        
        .file {
          height: 28px;
          width: 100%;
          line-height: 24px;
          font-size: 12px;
          padding-left: 5px;
          padding-right: 5px;
          box-sizing: border-box;
          background-color: var(--white);
          border-bottom-color: var(--light-gray);
          border-bottom-style: solid;
          border-bottom-width: 1px;
          display: flex;
          justify-content: space-between;
          align-items: center;
          transition: 0.2s;
          
          &:hover {
            background-color: var(--white-gray);
          }
          
          .filename {
            overflow: hidden;
            white-space: nowrap;
            text-overflow: ellipsis;
            flex-grow: 1;
            padding-right: 10px;
          }
          
          .delete {
            color: var(--light-gray);
            cursor: pointer;
            transition: 0.2s;
            
            &:hover {
              color: var(--warning-red);
            }
          }
        }
        
        &::-webkit-scrollbar {
          width: 10px;
        }
            
        &::-webkit-scrollbar-track {
          border-radius: 5px;
          background-color: var(--transparent);
          
          &:hover {
            background-color: var(--white-gray);
          }
        }
        
        &::-webkit-scrollbar-thumb {
          border-radius: 5px;
          background-color: var(--light-gray);
          transition: 0.2s;
          
          &:hover {
            background-color: var(--gray);
          }
        }
      }
    }
  }
    
  #templates {
    flex-direction: column;
    
    #container {
      width: 100%;
      flex-grow: 1;
      display: flex;
      flex-wrap: wrap;
      
      .template-container {
        width: calc(100%/3);
        height: 210px;
        box-sizing: border-box;
        padding: 10px;
        
        .card {
          width: 100%;
          height: 100%;
          color: var(--dark-gray);
          
          .el-card__body {
            width: 100%;
            height: 100%;
            box-sizing: border-box;
            display: flex;
            flex-direction: column;
            
            .subtitle {
              width: 100%;
              overflow: hidden;
              white-space: nowrap;
              text-overflow: ellipsis;
            }
            
            .control-buttons {
              width: 100%;
              flex-grow: 1;
              align-items: flex-end;
              
              .control-button {
                display: flex;
                flex-direction: column;
                justify-content: center;
                align-items: center;
                cursor: pointer;
                font-size: 12px;
                width: 32px;
                transition: 0.2s;
                
                svg {
                  font-size: 20px;
                  margin: 5px;
                }
                
                &:hover {
                  color: var(--main-color);
                }
                
                &:active {
                  filter: brightness(0.9);
                }
              }
            }
          }
          
          &:hover {
            transform: scale(1.05);
          }
        }
      }
    }
    
    #empty-container {
      width: 100%;
      flex-grow: 1;
      display: flex;
      justify-content: center;
      align-items: center;
      
      #empty {
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
        font-size: 14px;
        
        svg {
          font-size: 40px;
          margin: 14px;
        }
      }
    }
    
    .el-pagination {
      padding: 0;
      margin-right: 10px;
      
      li {
        min-width: 24px;
        height: 28px;
        line-height: 28px;
      }
      
      .btn-prev {
        width: 24px;
        height: 28px;
        line-height: 28px;
        margin-left: 0;
      }
      
      .btn-next {
        width: 24px;
        height: 28px;
        line-height: 28px;
        margin-right: 0;
      }
    }
  }
}
</style>