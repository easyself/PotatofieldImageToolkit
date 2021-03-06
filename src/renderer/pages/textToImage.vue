<template>
  <div id="textToImage">
    <div id="header">
      <div id="title">富文本制图工具</div>
      <div id="minimize" class="control-button" @click="minimize">
        <object data="static/images/minimize.svg" type="image/svg+xml"></object>
      </div>
      <div id="close" class="control-button" @click="close">
        <object data="static/images/close.svg" type="image/svg+xml"></object>
      </div>
    </div>
    <el-tabs type="card" tab-position="top" id="content">
      <el-tab-pane>
        <span slot="label"><i class="fas fa-keyboard"></i> 输入内容</span>
        <div class="tab-content">
          <div id="editor">
            <ckeditor :editor="editor.editor" :config="editor.config" v-model="content"></ckeditor>
            <div class="row">
              <el-button type="primary" size="mini" @click="clear" class="bar-button">清空内容</el-button>
              <el-button type="primary" size="mini" @click="edit" class="bar-button">进入样式编辑器</el-button>
            </div>
          </div>
        </div>
      </el-tab-pane>
      <el-tab-pane>
        <span slot="label"><i class="fas fa-file-alt"></i> 样式模板库</span>
        <div id="templates" class="tab-content">
          <div id="container" v-if="this.$store.state.textToImage.templates.length != 0">
            <div
              v-for="(template, index) in this.$store.state.textToImage.templates.slice(templateListPage * 6 - 6, templateListPage * 6)"
              :key="index"
              class="template-container">
              <el-card class="card">
                <div class="row">
                  <div class="subtitle">{{ template.title }}</div>
                </div>
                <div class="text">外框宽度：{{ template.padding != 0 ? template.padding : '无外框' }}</div>
                <div class="text">背景颜色：
                  <div
                    class="color-sample"
                    :style="{
                      'background-color': template.backgroundColor
                    }"></div>
                </div>
                <v-clamp autoresize :max-lines="2" class="text">{{ template.text }}</v-clamp>
                <div class="row actions">
                  <div class="action" @click="editTemplate(index + (templateListPage - 1) * 6)">
                    <span class="fa fa-edit"></span>
                    <div>编辑</div>
                  </div>
                  <div class="action" @click="shareTemplate(index + (templateListPage - 1) * 6)">
                    <span class="fa fa-share-alt"></span>
                    <div>分享</div>
                  </div>
                  <div class="action" @click="deleteTemplate(index + (templateListPage - 1) * 6)">
                    <span class="fa fa-trash-alt"></span>
                    <div>删除</div>
                  </div>
                </div>
              </el-card>
            </div>
          </div>
          <div v-else id="empty-container">
            <div id="empty">
              <i class="fas fa-folder-open"></i>
              <div>尚无已保存的模板</div>
            </div>
          </div>
          <div class="row">
            <el-pagination
              v-if="this.$store.state.textToImage.templates.length > 6"
            
              small
              background
              layout="prev, pager, next"
              :pager-count="5"
              :page-size="6"
              :total="this.$store.state.textToImage.templates.length"
              :current-page="templateListPage"
              :hide-on-single-page="true"
              @current-change="templateListPageChange">
            </el-pagination>
            <el-button type="primary" size="mini" @click="importTemplate" class="bar-button">导入样式模板</el-button>
            <el-button type="primary" size="mini" @click="createTemplate" class="bar-button">创建样式模板</el-button>
          </div>
        </div>
      </el-tab-pane>
    </el-tabs>
  </div>
</template>

<script>
import { ipcRenderer, clipboard } from 'electron'
import '@ckeditor/ckeditor5-build-classic/build/translations/zh-cn'
import ClassicEditor from '@ckeditor/ckeditor5-build-classic'
import UploadAdapterPlugin from '../utils/EditorUploadAdapter'

const path = require('path')

export default {
  name: 'textToImage',
  data () {
    return {
      editor: {
        editor: ClassicEditor,
        config: {
          language: 'zh-cn',
          heading: {
            options: [
              { model: 'paragraph', title: 'Paragraph', class: 'ck-heading_paragraph' },
              { model: 'heading1', view: 'h2', title: 'Heading 1', class: 'ck-heading_heading1' },
              { model: 'heading2', view: 'h3', title: 'Heading 2', class: 'ck-heading_heading2' }
            ]
          },
          image: {
            toolbar: []
          },
          toolbar: ['heading', 'imageUpload', 'bold', 'italic', 'blockQuote', 'bulletedList', 'numberedList', 'undo', 'redo'],
          extraPlugins: [UploadAdapterPlugin]
        }
      },
      content: '',
      templateListPage: 1,
      templateTitle: ''
    }
  },
  methods: {
    minimize() {
      ipcRenderer.send('minimize')
    },
    close() {
      ipcRenderer.send('close')
      this.$destroy()
    },
    clear() {
      this.content = ''
    },
    templateListPageChange(page) {
      this.templateListPage = page
    },
    edit() {
      if (this.content == '') {
        this.$dialog({
          type: 'warning',
          text: '请您输入内容！'
        })
      } else {
        this.$store.dispatch('textToImage/contentAssign', this.content).then(() => {
          this.$router.replace('/textToImage/editor')
        })
      }
    },
    editTemplate(index) {
      this.$router.replace('/textToImage/template?index=' + String(index))
    },
    shareTemplate(index) {
      clipboard.writeText(btoa(encodeURI(JSON.stringify({
        type: 'textToImageTemplate',
        content: this.$store.state.textToImage.templates[index]
      }))))
      this.$dialog({
        type: 'success',
        title: '成功',
        text: '已成功将样式模板复制到剪贴板。'
      })
    },
    deleteTemplate(index) {
      this.$dialog({
        type: 'warning',
        title: '操作确认',
        text: '确定要删除这个模板吗？',
        showCancel: true,
        confirmFunction: () => {
          if (this.templateListPage != 1) {
            if (this.templateListPage == Math.ceil(this.$store.state.textToImage.templates.length / 6)) {
              if (this.$store.state.textToImage.templates.length % 6 == 1) {
                this.templateListPage -= 1
              }
            }
          }
          this.$store.dispatch('textToImage/templateDelete', index)
        }
      })
    },
    importTemplate() {
      try {
        let template = JSON.parse(decodeURI(atob(clipboard.readText())))
        if (template.type != 'textToImageTemplate') {
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
                    title: '请输入样式模板标题',
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
              for (let i = 0; i < this.$store.state.textToImage.templates.length; i++) {
                if (title == this.$store.state.textToImage.templates[i].title) {
                  this.$dialog({
                    type: 'warning',
                    title: '存在同名模板',
                    text: '您需要将新导入的模板重命名，才能将其保存。',
                    showCancel: true,
                    confirmFunction: () => {
                      this.$dialog({
                        title: '请输入样式模板标题',
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
              this.$store.dispatch('textToImage/templatePush', template)
              this.$dialog({
                type: 'success',
                title: '成功',
                text: '样式模板导入成功。'
              })
            }
          }
          checkName(template.title)
        }
      } catch (e) {
        this.$dialog({
          type: 'error',
          title: '导入失败',
          text: '未能从您的剪贴板中读取到样式模板信息！'
        })
      }
    },
    createTemplate() {
      this.$router.replace('/textToImage/template?index=-1')
    }
  }
}
</script>

<style lang="scss">
#textToImage {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;

  button {
    font-family: var(--main-font);
  }

  #header {
    padding-left: 20px;
    padding-right: 20px;
    box-sizing: border-box;
    flex-basis: 40px;
    background-color: var(--dark-gray);
    display: flex;
    align-items: center;
    z-index: 3000;
    -webkit-app-region: drag;

    #title {
      color: var(--white);
      font-size: 16px;
      flex-grow: 1;
    }

    .control-button {
      -webkit-app-region: no-drag;
      width: 20px;
      height: 20px;
      margin-left: 5px;
      margin-right: 5px;
      border-radius: 10px;
      position: relative;
      display: flex;
      justify-content: center;
      align-items: center;

      object {
        width: 50%;
        color: var(--white);
      }

      &:first-child {
        margin-left: 0;
      }

      &:last-child {
        margin-right: 0;
      }

      &::after {
        content: '';
        position: absolute;
        width: 100%;
        height: 100%;
        left: 0;
        top: 0;
        border-radius: 50%;
        transition: 0.2s;
      }

      &:hover::after {
        background-color: rgba(0, 0, 0, 0.1);
      }
    }

    #minimize {
      background-color: var(--success-green);
    }

    #close {
      background-color: var(--warning-red);
    }
  }

  #content {
    height: 0;
    flex-grow: 1;
    display: flex;
    flex-direction: column;
    
    .el-tabs__header {
      margin: 0;
      
      .el-tabs__nav-scroll {
        background-color: var(--main-color);
        
        .el-tabs__nav {
          border: 0;
          
          .el-tabs__item {
            width: 150px;
            height: 50px;
            line-height: 50px;
            text-align: center;
            border: 0;
            transition: 0.2s;
            
            &.is-active {
              background-color: var(--white);
              color: var(--main-color);
              cursor: default;
            }
            
            &:not(.is-active) {
              color: var(--white);
              position: relative;
            }
            
            &:not(.is-active)::after {
              content: '';
              position: absolute;
              width: 100%;
              height: 100%;
              left: 0;
              top: 0;
              transition: 0.2s;
            }

            &:not(.is-active):hover::after {
              background-color: rgba(0, 0, 0, 0.1);
            }
          }
        }
      }
    }
    
    .el-tabs__content {
      flex-grow: 1;
      
      .el-tab-pane {
        width: 100%;
        height: 100%;
        
        .tab-content {
          width: 100%;
          height: 100%;
          padding: 20px;
          box-sizing: border-box;
          display: flex;
          justify-content: space-between;
        }
      }
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
      
      .control {
        width: 60%;
      }
      
      &:first-child {
        margin-top: 0;
      }
      
      &:last-child {
        margin-bottom: 0;
      }
    }
    
    .bar-button {
      width: 0;
      height: 28px;
      flex-grow: 1;
      box-sizing: border-box;
      border: none;
      padding-left: 0;
      padding-right: 0;
      margin-left: 5px;
      margin-right: 5px;
      
      &:first-child {
        margin-left: 0;
      }
      
      &:last-child {
        margin-right: 0;
      }
    }
    
    .wrapper {
      width: 100%;
      height: 100%;
      display: flex;
      flex-direction: column;
    }
    
    .el-input-group {
      display: flex;
    }
    
    .el-input-group__prepend {
      width: fit-content;
      display: flex;
      justify-content: center;
      align-items: center;
    }
      
    #editor {
      width: 100%;
      height: 100%;
      display: flex;
      flex-direction: column;
      justify-content: space-between;
      overflow: hidden;
      
      .ck-editor {
        height: 0;
        flex-grow: 1;
        display: flex;
        flex-direction: column;
        
        .ck-editor__main {
          flex-grow: 1;
          overflow: hidden;
          
          .ck-editor__editable {
            height: 100%;
            box-sizing: border-box;
            transition: 0.2s;
            
            blockquote {
              font-style: normal;
            }
            
            &:hover {
              border-color: var(--main-color);
            }
            
            &:focus {
              border-color: var(--main-color);
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
        
        .ck-toolbar {
          background-color: var(--white);
        }
      }
    }
      
    #templates {
      flex-direction: column;
      
      #container {
        width: 100%;
        height: 0;
        flex-grow: 1;
        display: flex;
        flex-wrap: wrap;
        
        .template-container {
          width: calc(100% / 3);
          height: 50%;
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
              
              .actions {
                width: 100%;
                flex-grow: 1;
                align-items: flex-end;
                
                .action {
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
}
</style>
