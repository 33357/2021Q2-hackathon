<!--
 * @Author: 33357
 * @Date: 2021-05-15 13:38:11
 * @LastEditTime: 2021-05-16 13:47:01
 * @LastEditors: 33357
-->
<template>
  <div class="home">
    <loading v-show="topIsLoading"></loading>

    <send-window
      v-if="showSendWindow"
      @closeSendWindow="closeSendWindow"
      :commentBlogId="commentBlogId"
    ></send-window>

    <div class="card" v-for="(item, index) in followBlogIds" :key="index">
      <div v-if="$store.state.blogs[item] !== undefined">
        <div class="card-main">
          <header class="card-header">
            <!-- <div
            class="header-bg"
            v-if="typeof item.mblog.cardid !== 'undefined'"
          ></div> -->
            <a class="avatar">
              <div class="avatar-wrapper border-around-1px">
                <img
                  class="avatar-img"
                  v-if="$store.state.avatars[$store.state.blogs[item].person]"
                  :src="$store.state.avatars[$store.state.blogs[item].person]"
                />
                <!-- <i
                  class="iconfont"
                  :class="calculateVerifiedClass(item.mblog.user.verified_type)"
                ></i> -->
              </div>
            </a>
            <div class="user-info">
              <a class="user-name txt-l txt-cut"
              v-if="$store.state.blogs[item]">{{
                formatName($store.state.blogs[item].person)
              }}</a>
              <div class="publish-data txt-xs">
                <span class="publish-created-at">{{
                  formatTime($store.state.blogs[item].createDate)
                }}</span>
                <span
                  class="publish-source"
                  v-if="$store.state.tokenList[$store.state.blogs[item].group]"
                  >来自{{
                    " " +
                    $store.state.tokenList[$store.state.blogs[item].group]
                      .symbol
                  }}</span
                >
              </div>
            </div>
          </header>
          <section class="card-body">
            <p
              class="default-content"
              v-html="$store.state.blogs[item].content"
            ></p>
            <div
              class="retweet"
              v-if="$store.state.blogs[item].repostBlogId !== 0"
            >
              <p v-if="$store.state.blogs[$store.state.blogs[item].repostBlogId]">
                <a class="retweet-user"
                  >@{{
                    formatName(
                      $store.state.blogs[$store.state.blogs[item].repostBlogId]
                        .person
                    )
                  }}</a
                >：{{
                  $store.state.blogs[$store.state.blogs[item].repostBlogId]
                    .content
                }}
              </p>
            </div>
          </section>
        </div>
        <footer
          class="card-footer border-1px border-top-1px txt-s no-text-select"
        >
          <a class="forward" @click.prevent="openSendWindow(item)">
            <i class="iconfont icon-forward"></i>
            {{ $store.state.blogs[item].commentBlogIds.length }}
          </a>
          <i class="separate-line"></i>
          <a
            class="like"
            @click.prevent="likeIt($event, item)"
            :class="{ liked: $store.state.favorites.indexOf(item) !== -1 }"
          >
            <i class="iconfont icon-like"></i>
          </a>
        </footer>
      </div>
      <transition
        name="like"
        v-on:before-enter="beforeLikeEnter"
        v-on:enter="likeEnter"
        v-on:after-enter="afterLikeEnter"
      >
        <div class="like-animation-wrapper" v-show="showLikeAnimationWrapper">
          <i class="iconfont icon-like"></i>
        </div>
      </transition>
    </div>
    <loading v-show="bottomIsLoading"></loading>
    <div
      class="content-tip no-text-select"
      v-show="noMore"
      @click="updateContent()"
    >
      <span>没有更多微博了QAQ，点我刷新看看吧！</span>
      <a class="iconfont icon-refresh"></a>
    </div>
  </div>
</template>

<script>
import loading from "../../components/loading/loading.vue";
import sendWindow from "../../components/sendWindow/sendWindow.vue";
import { formatTime, formatName } from "../../const/func";

export default {
  name: "follow",
  data() {
    return {
      //要使用实例的属性，需要在这初始化声明
      hasTopTip: false,
      topTip: {},
      weiboContent: {},
      pagePos: 0,
      topIsLoading: true,
      bottomIsLoading: false,
      noMore: false,
      noNew: false,
      showLikeAnimationWrapper: false,
      clickedLikeBtnPos: {
        pageX: 0,
        pageY: 0,
      },
      showSendWindow: false,
      commentBlogId: 0,
      followBlogIds: [],
      favorites: this.$store.state.favorites,
    };
  },
  components: {
    loading,
    "send-window": sendWindow,
  },
  created() {
    this.$http
      .get("apis/weibo-content?targetCursor=1", { id: 0 })
      .then((res) => {
        /*res.body.data和res.data.data，哪一个才是最佳实践？
         * .body 是vue-resource 的API，res.data未知？*/
        if (res.body.errorNum !== 0) {
          console.log("Get data error at created()!");
          return;
        }
        this.weiboContent = res.data.data; //微博的所有内容

        let tempTopTip = res.data.data.card_group[0];
        if (tempTopTip.mod_type === "mod/clientTopTips") {
          this.topTip = res.data.data.card_group.shift();
          this.hasTopTip = true;
        }
        this.bottomIsLoading = true;
        setTimeout(() => {
          //故意推迟，以展示加载动画效果
          this.topIsLoading = false;
        }, 600);
      });
    this.getFollowBlogIds();
    this.addScrollEvent();
  },
  methods: {
    formatTime(date) {
      return formatTime(date);
    },
    formatName(date) {
      return formatName(date);
    },

    async getFollowBlogIds() {
      let arr;
      if (
        this.$store.state.follows.indexOf(this.$store.state.walletAddress) == -1
      ) {
        arr = [this.$store.state.walletAddress, ...this.$store.state.follows];
      } else {
        arr = this.$store.state.follows;
      }
      arr.forEach(async (address) => {
        const res = await this.$store.state.web3.routerFunc.getPersonBlogIds(
          address
        );
        if (res.length != 0) {
          res.forEach(async (blogId) => {
            this.$store.dispatch("getBlog", blogId);
            if (this.followBlogIds.indexOf(blogId) == -1) {
              this.followBlogIds.push(blogId);
              this.updateFollowBlogIds();
            }
          });
        }
      });
    },
    updateFollowBlogIds() {
      this.followBlogIds.sort((a, b) => {
        return b - a;
      });
    },
    calculateVerifiedClass: function (verifiedType) {
      let tempOutcome = "";
      switch (verifiedType) {
        case -1:
          tempOutcome = "no-verified";
          break;
        case 0:
          tempOutcome = "icon-yellow-v";
          break;
        case 1:
          tempOutcome = "icon-blue-v";
          break;
      }
      return tempOutcome;
    },
    openPicViewer(targetPicUrl) {
      //        this.$store.commit('disableBodyScroll');
      this.$store.commit("openPicViewer", { targetPicUrl: targetPicUrl });
    },
    addScrollEvent() {
      // 监听滚动事件，判断是否接近页面，触发加载旧微博
      /*可以通过以下两句分别对比使用事件防抖前后的事件触发次数：*/
      //window.addEventListener('scroll', this.handleScroll)
      window.addEventListener(
        "scroll",
        this.myDebounce(this.handleScroll, 500)
      );
    },
    handleScroll() {
      let pageHeight = Math.max(
        document.documentElement.clientHeight,
        document.body.scrollHeight,
        document.documentElement.scrollHeight,
        document.body.offsetHeight,
        document.documentElement.offsetHeight
      );

      let windowScrollHeight = window.scrollY || window.pageYOffset;
      if (windowScrollHeight + window.innerHeight > pageHeight - 100) {
        this.hideAppAddTip();
        this.getContent();
      }
    },
    getContent() {
      let nextCursor = this.weiboContent.next_cursor;
      if (nextCursor !== -1) {
        let targetUrl = "/apis/weibo-content" + "?targetCursor=" + nextCursor;
        this.$http.get(targetUrl).then((res) => {
          if (res.body.errorNum === 0) {
            //把已有的微博和加载的旧微博合并
            this.weiboContent.card_group = this.weiboContent.card_group.concat(
              res.data.data.card_group
            );
            // 更新下一个目标的id，用于后续更新
            this.weiboContent.next_cursor = res.data.data.next_cursor;
          } else {
            console.log("getContent() error!");
          }
        });
      } else {
        this.noMore = true;
        this.bottomIsLoading = false;
      }
    },
    updateContent() {
      this.topIsLoading = true;
      this.hideAppAddTip();
      this.scrollToTop();

      let previousCursor = this.weiboContent.previous_cursor;
      if (previousCursor !== -1) {
        let targetUrl =
          "/apis/weibo-content" + "?targetCursor=" + previousCursor;
        this.$http.get(targetUrl).then((res) => {
          if (res.body.errorNum === 0) {
            // console.log('res.data.data.card_group = ', res.data.data.card_group)
            //把新的微博和已有的微博合并
            let _this = this;
            this.weiboContent.card_group = res.data.data.card_group.concat(
              _this.weiboContent.card_group
            );
            this.weiboContent.previous_cursor = res.data.data.previous_cursor;
            //故意延迟消失，以展示效果
            setTimeout(() => {
              this.topIsLoading = false;
            }, 500);
          } else {
            console.log("updateContent() error!");
          }
        });
      } else {
        setTimeout(() => {
          this.noNew = true;
          this.topIsLoading = false;
        }, 500);
        setTimeout(() => {
          this.noNew = false;
        }, 3000);
      }
    },
    scrollToTop() {
      window.scrollTo(0, 0);
    },
    myDebounce(func, wait) {
      let timeout;
      return function () {
        clearTimeout(timeout);
        timeout = setTimeout(func, wait);
        /*一个事件发生wait毫秒后，不再触发该事件，才执行相应的处理函数。*/
      };
    },
    likeIt(e, item) {
      // 保存点击位置，用于设置动画块的起始位置：
      this.clickedLikeBtnPos.pageX = e.pageX - parseInt(window.scrollX);
      this.clickedLikeBtnPos.pageY = e.pageY - parseInt(window.scrollY);
      const index = this.$store.state.favorites.indexOf(item);
      if (index === -1) {
        //显示点赞动画：
        this.showLikeAnimationWrapper = true;
        // 点赞数+1，并设置为已经赞了的状态
        /* vm.$set( target, key, value ) 用于设置对象的属性。
          如果对象是响应式的，确保属性被创建后也是响应式的，
          同时 **“触发视图更新” **。
          这个方法主要用于避开 Vue 不能检测属性被添加的限制。*/
        this.$store.state.favorites.push(item);
      } else {
        this.$store.state.favorites.splice(index, 1);
      }
    },
    beforeLikeEnter(el) {
      /*在动画块呈现之前，将其位置设置到点赞开始的位置上：*/
      el.style.transform = "scale(0.1)";
      el.style.top = this.clickedLikeBtnPos.pageY + "px";
      el.style.left = this.clickedLikeBtnPos.pageX + "px";
    },
    likeEnter(el, done) {
      /* eslint-disable no-unused-vars*/
      /*TODO:Figure out 很奇怪！必须要触发一次重绘才能实现位置移动的过渡效果？？？*/
      let rf = el.offsetHeight;
      el.style.transform = "scale(1)";
      el.style.top = "190px";
      el.style.left = "50%";
      done();
    },
    afterLikeEnter(el) {
      //动画结束后，隐藏该动画块（需要enter钩子中调用done()）：
      this.showLikeAnimationWrapper = false;
    },
    hideAppAddTip() {
      this.$emit("hideAppAddTip");
    },
    preventBgScroll() {
      // setTimeout(0) to avoid flash when the page scroll to top
      setTimeout(() => {
        this.$store.commit("storePagePos");
        this.$store.commit("disableBodyScroll");
      }, 0);
    },
    allowBgScroll() {
      setTimeout(() => {
        this.$store.commit("enableBodyScroll");
        this.$store.commit("restorePagePos");
      }, 0);
    },
    openSendWindow(blogId) {
      /*TODO:BUG Prevent scrolling when overlaid. (Headache!!!)
        Read the 23 in ../../notes/Summary-of-experience for detail.*/
      this.commentBlogId = Number(blogId);
      console.log(this.commentBlogId);
      this.preventBgScroll();
      this.showSendWindow = true;
    },
    closeSendWindow() {
      this.allowBgScroll();
      this.showSendWindow = false;
      this.getFollowBlogIds();
    },
  },
  computed: {
    switchPicViewer() {
      return this.$store.state.switchPicViewer;
    },
  },
};
</script>

<style scoped lang="stylus">
.top-tip {
  width: 100%;
  background-color: #fed;

  .to-top-tip {
    height: 2.75em;
    display: flex;
    align-items: center;
    padding-left: 0.75rem;

    .icon-hot {
      color: #ff8200;
      margin-top: -5px;
      font-size: 22px;
    }

    .top-tip-content {
      font-size: 0.875rem;
      color: #FF8200;
      padding: 0 0.6875rem;
      line-height: 1.5rem;

      .iconfont {
        margin-left: 0.5rem;
        font-size: 0.775rem;
      }
    }
  }
}

.card-main:active {
  background-color: #ebebeb;
}

.card-header {
  display: flex;

  .header-bg {
    background-image: url('../../../static/img/card-header-bg-headline.png');
    width: 100%;
    height: 60px;
    background-repeat: no-repeat;
    background-position: top right;
    background-size: 100% auto;
    position: absolute;
    top: -4px;
    left: 0;
  }

  .avatar {
    margin: 0.75rem 0 0.5rem 0.75rem;
    display: flex;
    position: relative;

    .avatar-img {
      width: 2.125rem;
      height: 2.125rem;
      border-radius: 50%;
      vertical-align: top;
      display: block;
    }

    .no-verified {
      display: none;
    }

    .icon-yellow-v, .icon-blue-v {
      position: absolute;
      right: -0.125rem;
      bottom: -0.125rem;
    }
  }
}

.user-info {
  max-width: 16rem;
  display: flex;
  justify-content: center;
  flex-direction: column;
  padding: 0.6875rem 0.6875rem 0;
  line-height: 1rem;

  .user-name {
    color: #333;
  }

  .publish-data {
    color: #929292;

    .publish-source {
      padding-left: 0.5rem;
    }
  }
}

.card-operate {
  position: absolute;
  top: 0;
  right: 0;
  width: 3.75rem;
  height: 3.375rem;

  &:active {
    background-color: #ebebeb;
  }

  .icon-down-arrow {
    position: absolute;
    top: 0.5rem;
    right: 0.5rem;
  }
}

.card-body {
  padding: 0.25rem 0.75rem 0.625rem;
  line-height: 1.35rem;
}

.pic-list {
  margin-top: 0.4375rem;
  width: 15.5rem;
  overflow: hidden;

  & li {
    float: left;
  }

  & img {
    width: 4.75rem;
    height: 4.75rem;
    margin-right: 0.25rem;
  }
}

.single-pic {
  margin-top: 0.3125rem;
  max-height: 12.5rem;
  overflow: hidden;
  text-align: left;

  & img {
    width: 11.25rem;
    vertical-align: top;
  }
}

.retweet {
  padding: 0.625rem;
  margin: 0.4375rem -0.75rem -0.75rem;
  background-color: #f7f7f7;
}

.card-footer {
  width: 100%;
  display: flex;
  align-items: center;
  background-image: -webkit-gradient(linear, left top, left bottom, color-stop(0%, #dadada), color-stop(50%, #dadada), color-stop(50%, rgba(0, 0, 0, 0)));
  background-image: -webkit-linear-gradient(top, #dadada 0, #dadada 50%, rgba(0, 0, 0, 0) 50%);
  background-image: linear-gradient(to bottom, #dadada 0, #dadada 50%, rgba(0, 0, 0, 0) 50%);
  /* 从上到下，在1px的高度之中，从#dadada开始，直到0.5px时，将剩余的下半部分0.5px设置为透明，
  从而实现视觉上的0.5px。 */
  -webkit-background-size: 100% 1px;
  background-size: 100% 1px;
  background-repeat: no-repeat;
  background-position: top;

  & > a {
    color: #929292;
    flex: 1;
    text-align: center;
    padding: 0.375rem 0;
    display: inline-block;
    height: 1.5rem;
    line-height: 1.5rem;

    &:active {
      background-color: #ebebeb;
    }
  }
}

.liked {
  color: red !important;
}

.like-animation-wrapper {
  width: 50px;
  height: 50px;
  line-height: 50px;
  text-align: center;
  color: #ff8200;
  background-color: #f5f5f5;
  border: 1px solid #eee;
  border-radius: 50%;
  position: fixed;
  transition: all 0.3s linear;
  margin-left: -25px;

  .icon-like {
    font-size: 32px;
  }
}

.content-tip {
  width: 100%;
  text-align: center;
  min-height: 50px;
  margin-bottom: 0.5625rem;

  &:active {
    background-color: #ebebeb;
  }

  span {
    display: inline-block;
    font-size: 0.75rem;
    line-height: 19px;
    color: #7c7c7c;
    margin: 14px 0;
  }
}

.like-enter-to {
  // like动画最后一帧，即结束时，给图标添加上转动的动画
  animation: like-rotate 0.8s 0.4s;
}

@keyframes like-rotate {
  0% {
    transform: rotate(30deg);
  }

  20% {
    transform: rotate(-30deg);
  }

  40% {
    transform: rotate(30deg);
  }

  60% {
    transform: rotate(0deg);
  }
}

@media (min-width: 600px) {
  // m.weibo.com没有很完美的解决微博卡片标题背景挂件比例失调的问题，
  // 只是在大屏幕下粗暴的隐藏掉了。
  // 相比之下，我认为下面这个方法虽然多了一些代码，但至少没有因噎废食。
  .card-header .header-bg {
    background-size: 50%;
  }
}
</style>