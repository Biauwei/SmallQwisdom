<template>
  <div class="page">
    <template v-if="roleType != 3">
      <div class="page-hd">
        <div class="button-sp-area flex" size-17>
          <a href="javascript:void(0);" id="showDatePicker" @click="popupShow = true">
            <span>{{ className }}</span>
            <van-icon name="arrow-down" size="16px"></van-icon>
          </a>
        </div>
      </div>
    </template>
    <div class="page-bd">
      <!-- -->
      <van-popup v-model="popupShow" position="bottom">
        <div class="popup-class">
          <div class="cells">
            <div class="cell popup-box" v-for="(p, index) in classList" :key="index">
              <div class="cell-hd">
                <img src="@/assets/kong.png" width="54" height="54" />
              </div>
              <div class="cell-bd pl-20">
                <p>{{ p.className }}</p>
              </div>
              <div class="cell-ft">
                <van-radio-group v-model="classId">
                  <van-radio
                    :name="p.classId"
                    checked-color="#92cd36"
                    @click="handleClassConfirm(p)"
                  ></van-radio>
                </van-radio-group>
              </div>
            </div>
          </div>
        </div>
      </van-popup>
      <template v-if="roleType == 2">
        <qxRelease url="/fresh/add" />
      </template>
      <!-- list -->
      <van-list
        v-model="loading"
        :finished="finished"
        :immediate-check="false"
        :offset="100"
        @load="onLoad"
      >
        <van-swipe-cell
          ref="swipeCell"
          :right-width="60"
          v-for="(fresh, index) in list"
          :key="index"
          :disabled="roleType == 3"
          :on-close="onClose(fresh, index)"
        >
          <van-cell-group>
            <figure class="figure figure-skin-two" @click="go(fresh)">
              <div class="figure-bd">
                <div class="figure-info">
                  <figcaption size-18 class="text-ellipsis">{{ fresh.title }}</figcaption>
                  <div class="metedata flex">
                    <time class="time">{{ fresh.postTime }}</time>
                  </div>
                  <div
                    class="figure-thumb-medium"
                    v-if="fresh.topImage"
                    :style="{backgroundImage: `url(${fresh.topImage})`}"
                  ></div>
                  <p class="line-clamp">{{ fresh.textContent | brReplace }}</p>
                </div>
              </div>
              <div class="figure-ft">
                <div class="figure-icon">
                  <van-icon name="eye-o" size="16px"></van-icon>
                  <b>{{ fresh.classReadCount }}</b>
                </div>
                <div class="figure-icon" v-if="fresh.classCommentCount">
                  <van-icon name="comment-o" size="16px"></van-icon>
                  <b>{{ fresh.classCommentCount }}</b>
                </div>
              </div>
            </figure>
          </van-cell-group>
          <span slot="right" style="line-height: 80px;">删除</span>
        </van-swipe-cell>
      </van-list>
      <div class="empty" v-if="empty">
        <img src="@/assets/kong.png" alt />
        <p>暂无新鲜速报</p>
      </div>
    </div>
  </div>
</template>
<script>
import service from "@/api";
import qxRelease from "@/components/Release";
import classList from "@/mixins/classList";
export default {
  name: "fresh",
  mixins: [classList],
  components: {
    qxRelease
  },
  data() {
    return {
      empty: false,
      loading: false,
      finished: false,
      popupShow: false,
      className: this.$store.state.user.info.className,
      classId: parseInt(this.$store.state.user.info.classId),
      totalPage: 1, //总页数
      query: {
        openId: this.$store.state.user.info.openId,
        classId: this.$store.state.user.info.classId,
        studentId: this.$store.state.user.info.studentId,
        page: 1,
        pageSize: 10
      },
      roleType: this.$store.state.user.info.roleType,
      list: []
    };
  },
  methods: {
    onLoad() {
      if (this.query.page < this.totalPage) {
        //加载数据
        this.query.page += 1;
        service.freshQuery(this.query).then(res => {
          if (res.errorCode === 0) {
            let list = res.data.data;
            this.totalPage = res.data.totalPage;
            this.query.page = res.data.page;
            // 加载状态结束
            this.loading = false;
            for (let i = 0; i < list.length; i++) {
              this.list.push(list[i]);
            }
          }
        });
      } else {
        // 数据全部加载完成
        console.log("数据全部加载完成");
        this.loading = false;
        this.finished = true;
      }
    },
    onClose(fresh, index) {
      return (clickPosition, instance) => {
        switch (clickPosition) {
          case "right":
            this.$dialog
              .confirm({
                title: "提示",
                message: "确定删除该条速报吗？"
              })
              .then(async () => {
                let obj = {
                  openId: this.query.openId,
                  freshId: fresh.freshId
                };
                //删除速报
                let res = await service.deleteFresh(obj);
                if (res.errorCode === 0) {
                  instance.close();
                  this.list.splice(index, 1);
                }
              })
              .catch(() => {
                instance.close();
              });
            break;
        }
      };
    },
    go(fresh) {
      let { freshId, classId, studentId } = fresh;
      this.$router.push({
        path: "/fresh/show",
        query: {
          freshId,
          classId,
          studentId
        }
      });
    },
    //选择班级
    handleClassConfirm(obj) {
      this.popupShow = false;
      this.className = obj.className;
      this.query.page = 1;
      this.query.classId = obj.classId;
      //当切换班级时，重新设置为没有全部加载完成
      this.finished = false;
      this.freshQuery(this.query);
    },
    //删除速报
    async deleteFresh(params = {}) {
      let res = await service.deleteFresh(params);
      if (res.errorCode === 0) {
      }
    },
    //速报列表查询
    async freshQuery(params = {}) {
      let res = await service.freshQuery(params);
      if (res.errorCode === 0) {
        this.popupShow = false;
        this.query.page = res.data.page;
        this.totalPage = res.data.totalPage;
        this.list = res.data.data || [];
        if (this.list.length) {
          this.empty = false;
        } else {
          this.empty = true;
        }
      } else {
        this.$toast(`${res.errorMsg}`);
      }
    }
  },
  mounted() {
    this.freshQuery(this.query);
  }
};
</script>
<style lang="less" scoped>
.page-hd {
  margin-bottom: 20px;
  background-color: #fff;
}
</style>
