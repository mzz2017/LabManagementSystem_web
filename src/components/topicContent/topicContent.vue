<template>
  <div class="page-wrapper">
    <div class="content">
      <div class="wrapper">
        <div id="nornal-window" v-show="editMode===false">
          <article class="article">
            <header class="article-info clearfix">
              <div class="author-info-block">
                <router-link class="avatar" :to="{path:'/profile',query:{uid:topicInfo.author.uid}}">
                  <img v-lazy="topicInfo.authorAvatar">
                </router-link>
                <div class="author-info-box">
                  <div>
                    <router-link :to="{path:'/profile',query:{uid:topicInfo.author.uid}}" class="name">
                      {{topicInfo.author.name}}
                    </router-link>
                  </div>
                  <time :datetime="topicInfo.datetime" itemprop="datePublished" class="publish-time">
                    {{rTime(topicInfo.datetime)}}
                  </time>
                </div>
              </div>
              <h1 class="title">{{topicInfo.title}}</h1>
            </header>
            <div class="article-content" v-html="topicInfo.content" ref="content"></div>
          </article>
        </div>
        <div id="edit-window" v-if="editMode">
          <quill-editor
            v-model="editContent" style="height: 90%">
          </quill-editor>
        </div>
        <CollapsePanel class="sidebar-right"
                       v-if="identity===3 || topicInfo.author.name===username"
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
                <label class="labelfont">主题</label>{{panelInfo.type}}
              </p>
              <el-input class="panel-para"
                        size="mini"
                        v-model="panelInfo.type"
                        v-else>
                <label slot="prefix" class="panel-label">主题</label>
              </el-input>
            </div>
            <div class="bottom-operaton">
              <el-button size="mini" @click="handleClickSaveBtn">保存修改</el-button>
              <el-button size="mini" :type="editMode?'danger':''" @click="handleClickEditBtn">{{editMode?'退出编辑':'编辑文章'}}
              </el-button>
            </div>
          </div>
        </CollapsePanel>
      </div>
    </div>
    <div class="content" id="comment">
      <div class="wrapper">
        <div class="comment-form author-info-block">
          <div class="avatar">
            <img v-lazy="personalInfo.avatar">
          </div>
          <div class="comment-box">
            <el-input
              type="textarea"
              :autosize="{ minRows: 2}"
              resize="none"
              placeholder="说说你的看法"
              v-model="commentContent"
              @focus="handleOnCommentInputFocus">
            </el-input>
            <div class="commit-submit" v-show="commentExtend">
              <el-button size="medium" type="primary" class="comment-submit-btn"
                         @click="handleOnClickCommentSubmitButton">评论
              </el-button>
            </div>
          </div>
        </div>

        <ul class="comment-list">
          <li class="comment-list-item" v-for="(item, index) in commentList">
            <router-link class="avatar-small" :to="{path:'/profile',query:{uid:item.user.uid}}">
              <img v-lazy="item.avatar">
            </router-link>
            <div class="content-box">
              <router-link :to="{path:'/profile',query:{uid:item.user.uid}}" class="comment-name">{{item.user.name}}</router-link>
              <div class="comment-content">
                <span class="comment-content-span">{{item.content}}</span>
              </div>
              <div class="comment-action">
                <button :class="{'comment-action-item btn-like':true,'icon-liked':item.liked}"
                        @click="handleOnClickLikeButton(index)">
                  <i class="el-icon-caret-top"></i>
                  <span>{{item.like}}</span>
                </button>
                <div class="comment-action-item">
                  {{rTime(item.time)}}
                </div>
              </div>
            </div>
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script>
  // import 'common/katex/katex.min'
  import axios from 'axios'
  import store from 'store/store'
  import {mapState} from 'vuex'
  import CollapsePanel from 'base/collapsePanel/collapsePanel'
  import {Option, Select} from 'iview'

  import relativeTime from '../../common/js/relativeTime'

  // import ElInputCus from 'base/inputCus/inputCus'
  export default {
    store,
    name: "topicContent",
    components: {
      CollapsePanel,
      Select,
      Option
    },
    data() {
      return {
        // 利用vue-router传参
        topicType: '',
        topicInfo: {
          title: '',
          author: {
            name: '',
            uid: 1,
          },
          authorAvatar: '',
          datetime: '',
          content: ''
        },
        personalInfo: {
          avatar: ''
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
        editContent: null,
        commentExtend: false,
        commentContent: '',
        commentList: []
      }
    },
    methods: {
      loadData() {
        //获取话题文章内容
        axios({
          url: apiRootPath + 'topic/topicDetail',
          method: 'get',
          params: {
            topicId: this.$route.params.topicid
          }
        }).then((response) => {
          if (response.data.code === "SUCCESS" && response.data.data !== null) {
            // console.log(response.data.data)
            this.topicInfo = response.data.data
            this.panelInfo.type = response.data.data.type
            this.panelInfo.title = response.data.data.title
            this.editMode = this.$route.query.hasOwnProperty('mode') && this.$route.query.mode === '1'
            document.title = `${response.data.data.title} - ${this.defaultTitle}`
            if (this.editMode) {
              this.editContent = this.topicInfo.content
              this.setEditorHeight()
            }
            else {
              this.editContent = null
            }
          } else {
            process.env.NODE_ENV === "development" && console.log(response.data)
            this.$router.replace({name: 'error404'})
          }
          //获取到之后重新定位过来(#comment)
          if (location.hash.charAt(0) === '#')
            setTimeout(() => {
              document.getElementById(location.hash.substring(1)).scrollIntoView()
            }, 200)
        }).catch((error) => {
          if (location.hash.charAt(0) === '#')
            setTimeout(() => {
              document.getElementById(location.hash.substring(1)).scrollIntoView()
            }, 200)
          process.env.NODE_ENV === "development" && console.log(error)
          // this.$router.replace({name: 'error404'})
        })

        //获取评论
        axios({
          url: apiRootPath + 'topic/commentDetail',
          method: 'get',
          params: {
            topicId: this.$route.params.topicid
          }
        }).then((response) => {
          if (response.data.code === 'SUCCESS') {
            this.commentList = response.data.data
          }
        }).catch((error) => {
          console.log(error)
        })

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
          topicId: this.$route.params.topicid,
          title: this.panelInfo.title,
          type: this.panelInfo.type
        }
        if (this.editMode) {
          postData['content'] = this.editContent
        }
        // console.log(this.newsInfo.title)
        axios({
          url: apiRootPath + 'topic/editTopic',
          method: 'post',
          data: postData
        }).then((response) => {
          if (response.data.code === 'SUCCESS') {
            this.$message({
              type: 'success',
              message: "文章已成功保存！"
            })
            this.topicInfo.title = this.panelInfo.title
            this.topicInfo.type = this.panelInfo.type
            if (this.editMode)
              this.topicInfo.content = this.editContent
            this.$router.replace(`/topic/${this.$route.params.topicid}`)
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
            url: apiRootPath + 'topic/deleteTopic',
            method: 'post',
            data: {
              topicId: this.$route.params.topicid
            }
          }).then((response) => {
            if (response.data.code === 'SUCCESS') {
              this.$message({
                type: 'success',
                message: '删除成功!'
              });
              this.$router.push({path: '/topic'})
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
      },
      handleOnCommentInputFocus() {
        this.commentExtend = true
      },
      handleOnClickLikeButton(index) {
        let plus
        if (this.commentList[index].liked) {
          plus = false
          this.commentList[index].like--;
        }
        else {
          plus = true
          this.commentList[index].like++;
        }
        this.commentList[index].liked = !this.commentList[index].liked;
        axios({
          url: apiRootPath + 'topic/setCommentLiked',
          method: 'post',
          data: {
            "topicId": this.$route.params.topicid,
            "commentId": this.commentList[index].commentId
          }
        }).then((response) => {
          if (response.data.code === 'SUCCESS') {
            //success
          }
          else {
            this.commentList[index].liked = !this.commentList[index].liked;
            if (plus) {
              this.commentList[index].like--;
            }
            else {
              this.commentList[index].like++;
            }
            this.$message.warning(response.data.message)
          }
        }).catch((error) => {
          this.$message.warning('阿哦，出现了一点问题')
          this.commentList[index].liked = !this.commentList[index].liked;
          if (plus) {
            this.commentList[index].like--;
          }
          else {
            this.commentList[index].like++;
          }
          console.log(error)
        })
      },
      rTime(t) {
        return relativeTime(t)
      },
      handleOnClickCommentSubmitButton() {
        axios({
          url: apiRootPath + 'topic/addTopicComment',
          method: 'post',
          data: {
            "topicId": parseInt(this.$route.params.topicid),
            "content": this.commentContent
          }
        }).then((response) => {
          if (response.data.code === 'SUCCESS') {
            this.loadData()
            this.$message.success('评论成功！')
            this.commentContent = ""
          } else {
            this.$message.warning(`评论失败，错误提示: ${response.data.message}`)
          }
        }).catch((error) => {
          this.$message.error('发布失败，请检查网络连接！')
          process.env.NODE_ENV === 'development' && console.log(error)
        })
      }
    },
    created() {
      this.loadData()
    },
    mounted() {
      // console.log('editmode:', this.editMode, this.$route)
      // katex.render("L_{0m}^{k+1}=\\min\\{L_{01}^k+l_{1m},L_{02}^k+l_{2m},L_{03}^k+l_{3m},...,L_{0(n-1)}^k+l_{(n-1)m}\\}", this.$refs.content);
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
        return this.$route.params.topicid
      },
      ...mapState(['defaultTitle']),
      query() {
        return this.$route.query
      },
      identity() {
        //0:,1:学生,2:老师,3:管理员
        let identity = window.localStorage.getItem('identity')
        return identity ? identity : 0
      },
      username() {
        let username = window.localStorage.getItem('username')
        return username ? username : '未登录'
      }
    },
    watch: {
      watchNeeds() {
        this.loadData()
      },
      query(val) {
        this.editMode = this.$route.query.hasOwnProperty('mode') && this.$route.query.mode === '1'
        this.editContent = this.editContent !== null ? this.editContent : this.topicInfo.content
        this.setEditorHeight()
      }
    }
  }
</script>

<style lang="scss" scoped>
  @import '../../common/css/topicContent.scss';
</style>
<style lang="stylus">
  @import '../../common/css/article/style.styl';
</style>
<style lang="scss">
  //UI框架自定义修正
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

  .comment-form {
    .el-textarea__inner {
      overflow: hidden !important;
    }
    .comment-submit-btn {
      background-color: #007fff !important;
      border-radius: 2px !important;
      &:hover {
        background-color: #0371df !important;
      }
    }
  }
</style>

