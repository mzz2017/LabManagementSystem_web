<template>
  <div class="wrapper">
    <div id="nornal-window" v-show="editMode===false">
      <article class="article">
        <header class="article-info clearfix">
          <h1 class="title">{{newsInfo.title}}</h1>
          <p class="article-author" data-hc-id="408158643ed564c72fa092180000">By
            <router-link :to="{path:'/profile',query:{uid: newsInfo.author.uid}}" :title="newsInfo.author.name"
                         target="_self" itemprop="author">{{newsInfo.author.name}}
            </router-link>
          </p>
          <p class="article-time">
            <time :datetime="newsInfo.datetime" itemprop="datePublished" class="el-icon-date"> 发表于 {{newsInfo.datetime}}
            </time>
          </p>
        </header>
        <div class="article-content" v-html="newsInfo.content" ref="content"></div>
      </article>
    </div>
    <div id="edit-window" v-if="editMode">
      <quill-editor
        v-model="editContent" style="height: 90%">
      </quill-editor>
    </div>
    <CollapsePanel class="sidebar-right"
                   v-if="identity===3 || newsInfo.author.name===username"
                   @click.native="handleOnMouseClick(false,'title');handleOnMouseClick(false,'type')"
                   :show-close=true @onclose="handleOnClickClosePanel">
      <div class="panelContent">
        <div class="panel-row" @click="handleOnMouseClick(true,'title')">
          <p class="panel-para" v-if="panelEditable.title===false">
            <label class="labelfont">标题</label>{{panelInfo.title}}
          </p>
          <el-input class="panel-para"
                    size="mini"
                    v-model="panelInfo.title"
                    v-else>
            <label slot="prefix" class="panel-label">标题</label>
          </el-input>
        </div>
        <div class="panel-row" @click="handleOnMouseClick(true,'type')">
          <p class="panel-para" v-if="panelEditable.type===false">
            <label class="labelfont">分类</label>{{newsMap[panelInfo.type]}}
          </p>
          <div style="display: flex; height:35px" v-else>
            <label class="labelfont">分类</label>
            <Select v-model="panelInfo.type" style="width:125px;height:35px;">
              <Option v-for="(key,val) in newsMap" :value="val" :key="key">{{ key }}</Option>
            </Select>
          </div>
        </div>
        <div class="bottom-operaton">
          <el-button size="mini" @click="handleClickSaveBtn">保存修改</el-button>
          <el-button size="mini" :type="editMode?'danger':''" @click="handleClickEditBtn">{{editMode?'退出编辑':'编辑文章'}}
          </el-button>
        </div>
      </div>
    </CollapsePanel>
  </div>
</template>

<script>
  // import 'common/katex/katex.min'
  import axios from 'axios'
  import store from 'store/store'
  import {mapState} from 'vuex'
  import CollapsePanel from 'base/collapsePanel/collapsePanel'
  import {Option, Select} from 'iview'


  // import ElInputCus from 'base/inputCus/inputCus'
  export default {
    store,
    name: "newsContent",
    components: {
      CollapsePanel,
      Select,
      Option
    },
    data() {
      return {
        // 利用vue-router传参
        newsType: '',
        newsInfo: {
          title: '',
          author: {
            uid: 0,
            name: ''
          },
          datetime: '',
          content: ''
        },
        panelEditable: {
          title: false,
          type: false
        },
        panelInfo: {
          title: '',
          type: ''
        },
        clickTimer: {
          title: null,
          type: null
        },
        editMode: false,
        editContent: null
      }
    },
    methods: {
      loadData() {
        let cfg = {
          url: apiRootPath + 'news/assignationArticle',
          method: 'get',
          params: {
            newsId: this.$route.params.newsid
          }
        }
        axios(cfg)
          .then((response) => {
            if (response.data.code === "SUCCESS" && response.data.data !== null) {
              if (response.data.data.type !== this.$route.params.newstype)
                this.$router.replace({name: 'error404'})
              this.newsInfo = response.data.data
              this.panelInfo.type = response.data.data.type
              this.panelInfo.title = response.data.data.title
              this.editMode = this.$route.query.hasOwnProperty('mode') && this.$route.query.mode === '1'
              document.title = `${response.data.data.title} - ${this.defaultTitle}`
              if (this.editMode) {
                this.editContent = this.newsInfo.content
                this.setEditorHeight()
              }
              else {
                this.editContent = null
              }
            } else {
              process.env.NODE_ENV === "development" && console.log(response.data)
              // this.$router.replace({name: 'error404'})
              this.$router.replace({name: 'error404'})
            }
          })
          .catch((error) => {
            process.env.NODE_ENV === "development" && console.log(error)
            // this.$router.replace({name: 'error404'})
          })

        // this.editMode = false
        // this.editContent = null

      },
      handleOnMouseEnterPanel(enter, component) {
        this.panelEditable[component] = enter
      },
      handleOnMouseClick(state, component) {
        if (this.clickTimer[component] && state === false) {
          return
        }
        // console.log(state, component[component])
        clearTimeout(this.clickTimer[component])
        this.clickTimer[component] = setTimeout(() => {
          this.panelEditable[component] = state
          this.clickTimer[component] = null
        }, 10)
      },
      handleClickSaveBtn() {
        let postData = {
          newsId: this.$route.params.newsid,
          title: this.panelInfo.title,
          type: this.panelInfo.type
        }
        if (this.editMode) {
          postData['content'] = this.editContent
        }
        // console.log(this.newsInfo.title)
        axios({
          url: apiRootPath + 'news/modifyNews',
          method: 'post',
          data: postData
        }).then((response) => {
          if (response.data.code === 'SUCCESS') {
            this.$message({
              type: 'success',
              message: "文章已成功保存！"
            })
            this.newsInfo.title = this.panelInfo.title
            this.newsInfo.type = this.panelInfo.type
            if (this.editMode)
              this.newsInfo.content = this.editContent
            this.$router.replace(`/news/${this.panelInfo.type}/${this.$route.params.newsid}`)
          } else {
            this.$message.warning(`保存失败，错误提示: ${response.data.message}`)
          }
        }).catch((error) => {
          this.$message.warning("保存失败，请检查网络连接！")
          process.env.NODE_ENV === 'development' && console.log(error)
        })
      },
      handleClickEditBtn() {
        this.editMode = !this.editMode
        this.$router.push({path: this.$route.fullPath, query: {mode: this.editMode ? '1' : '0'}})
      },
      handleOnClickClosePanel() {
        this.$confirm('此操作将删除这篇文章, 是否继续?', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          axios({
            url: apiRootPath + 'news/deleteNews',
            method: 'post',
            data: {
              newsId: this.$route.params.newsid
            }
          }).then((response) => {
            if (response.data.code === 'SUCCESS') {
              this.$message({
                type: 'success',
                message: '删除成功!'
              });
              this.$router.push({path: '/news'})
            } else {
              this.$message.warning(`删除失败，错误提示: ${response.data.message}`)
            }
          }).catch((error) => {
            this.$message.error('删除失败，请检查网络连接！')
            process.env.NODE_ENV === 'development' && console.log(error)
          })
        }).catch((error) => {
          this.$message({
            type: 'info',
            message: '已取消删除'
          })
          console.log(error)
        })
      },
      setEditorHeight() {
        // document.getElementById()
        this.$nextTick(() => {
          let e = document.getElementById('edit-window')
          if (e)
            e.style.height = `${window.innerHeight - 200}px`;
        })
      }
    },
    mounted() {
      // console.log('editmode:', this.editMode, this.$route)
      // katex.render("L_{0m}^{k+1}=\\min\\{L_{01}^k+l_{1m},L_{02}^k+l_{2m},L_{03}^k+l_{3m},...,L_{0(n-1)}^k+l_{(n-1)m}\\}", this.$refs.content);
      this.loadData()
      // console.log(this.newsMap)
      var timer = null


      window.onresize = () => {
        if (!this.editMode)
          return
        clearTimeout(timer)
        timer = setTimeout(() => {
          this.setEditorHeight()
        }, 100)
      }
    },
    computed: {
      watchNeeds() {
        return this.$route.params.newstype + this.$route.params.newsid
      },
      ...mapState(['defaultTitle', 'newsMap']),
      query() {
        return this.$route.query
      },
      identity() {
        //管理员返回3
        if (localStorage.getItem('admin') === '1')
          return 3
        let identity = localStorage.getItem('identity')
        return identity ? parseInt(identity) : 0
      },
      username() {
        return localStorage.getItem('username')
      }
    },
    watch: {
      '$route'() {
        this.loadData()
      }
    }

  }
</script>

<style lang="scss" scoped>
  @import '../../common/css/newsContent';
</style>
<style lang="stylus">
  @import '../../common/css/article/style.styl';
</style>
<style>
  @import '../../common/katex/katex.min.css';

  .el-input__inner {
    padding-left: 42px !important;
  }

  .ivu-select-single {
    height: 35px !important;
    display: flex;
    align-items: center;
  }

  .ivu-select-selected-value, .ivu-select-selected-value {
    height: 28px !important;
  }

  .ivu-select-selection {
    height: 28px !important;
    width: 125px;
  }
</style>
