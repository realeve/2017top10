<template>
  <div>
    <x-header></x-header>
    <toast v-model="toast.show" :type="toast.type">{{ toast.text }}</toast>
    <div class="vote" v-for="(item,i) in checkList" :key="item.id">
      <p class="title">{{i | chinese}}、{{item.title}}</p>
      <card>
        <div slot="content" class="card-content">
          <p class="desc">{{item.content}}</p>
          <div class="weui-cell switch">
            <span>投它一票</span>
            <span>{{progress}}</span>
            <input type="checkbox" class="weui-switch" v-model="valueList[i]" @on-change="checkMaxVotes(i)">
          </div>
        </div>
      </card>
    </div>

    <p style="margin-top:20px;" v-show="myChecked.length">我的选择</p>
    <div class="weui-cells weui-cells_checkbox">
      <label class="weui-cell weui-check_label" v-for="(item,i) in myChecked" :key="item.id">
        <div class="weui-cell__bd">
          <p>{{i+1}}.{{item.title}}</p>
        </div>
      </label>
    </div>

    <div class="submit">
      <x-button :disabled="maxnum!=sport.maxTickets" @click.native="submit" type="primary">提交数据</x-button>
      <x-button @click.native="addInfo" type="default">填写个人信息</x-button>
    </div>

    <x-footer/>
  </div>

</template>

<script>
import { Sticky, Group, Cell, Card, Divider, Toast, XButton } from "vux";
import XFooter from "./Footer";
import XHeader from "./Header";
import _checkList from "../js/checkList";
import util from "../js/common";
import md5 from "md5";
import { mapState } from "vuex";

import moment from "moment";

export default {
  components: {
    Sticky,
    Group,
    Cell,
    Divider,
    Card,
    Toast,
    XButton,
    XHeader,
    XFooter
  },
  data() {
    return {
      valueList: [],
      toast: {
        show: false,
        text: "",
        type: ""
      },
      voteNum: [],
      checkList: _checkList, //util.randomArr(_checkList),
      time: new Date().getTime(),
      signature: ""
    };
  },
  computed: {
    ...mapState(["cdnUrl", "sport", "userInfo"]),
    maxnum() {
      let count = this.valueList.filter(item => item);
      return count.length;
    },
    openid() {
      return this.userInfo.openid;
    },
    curTimeStamp() {
      return (new Date().getTime() / 1000).toFixed(0);
    },
    progress() {
      return this.maxnum + "/" + this.sport.maxTickets;
    },
    myChecked() {
      let arr = [];
      this.checkList.forEach((item, i) => {
        if (this.valueList[i]) {
          arr.push(item);
        }
      });
      return arr;
    },
    isSportEnd() {
      return moment().format("YYYY-MM-DD") > this.sport.endDate;
    }
  },
  filters: {
    chinese(i) {
      return [
        "一",
        "二",
        "三",
        "四",
        "五",
        "六",
        "七",
        "八",
        "九",
        "十",
        "十一",
        "十二",
        "十三",
        "十四",
        "十五"
      ][i];
    }
  },
  methods: {
    getSignature() {
      this.time = new Date().getTime();
      this.signature = md5(
        btoa(
          this.sport.salt + this.time + this.openid + this.openid.toUpperCase()
        )
      );
    },
    showToast(settings) {
      this.toast.text = settings.text;
      this.toast.type = settings.type;
      this.toast.show = true;
      setTimeout(() => {
        this.toast.show = false;
      }, 1500);
    },
    checkMaxVotes(i) {
      if (this.maxnum > this.sport.maxTickets) {
        this.showToast({
          text: "请勿超过" + this.sport.maxTickets + "票",
          type: "warn"
        });
      }
    },
    getOriginIdx(newIdx) {
      return this.checkList[newIdx].id;
    },
    submit() {
      this.getSignature();
      let arr = [];
      this.valueList.forEach((item, i) => {
        if (item) {
          arr.push(i);
        }
      });
      let addStr = arr
        .map(item => this.getOriginIdx(item))
        .sort((a, b) => a - b);
      let params = {
        s: "/addon/Api/Api/addVoteInfo",
        sid: this.sport.id,
        timestamp: this.time,
        signature: this.signature,
        addstr: addStr.join(","),

        openid: this.userInfo.openid,
        nickname: this.userInfo.nickname,
        sex: this.userInfo.sex,
        city: this.userInfo.city,
        province: this.userInfo.province,
        country: this.userInfo.country,
        headimgurl: this.userInfo.headimgurl
      };

      let url = this.cdnUrl;
      this.$http
        .jsonp(url, {
          params
        })
        .then(res => {
          if (!res.ok) {
            this.showToast({
              text: "数据提交失败",
              type: "warn"
            });
            return;
          }
          var data = res.data;
          if (data.status > "0") {
            this.showToast({
              text: data.title,
              type: "success"
            });
            // 跳转提交用户信息
            setTimeout(() => {
              this.$router.push("/info");
            }, 500);
          } else {
            this.showToast({
              text: data.title,
              type: "warn"
            });
          }
        })
        .catch(e => {
          console.log(e);
        });
    },
    getStep() {
      let url = this.cdnUrl;
      let params = {
        s: "/addon/Api/Api/isSetUserInfo",
        openid: this.openid,
        sid: this.sport.id
      };
      this.$http
        .jsonp(url, {
          params
        })
        .then(res => {
          var data = res.data;
          if (data.status > 1) {
            this.$router.push("/info");
          } else if (this.isSportEnd) {
            this.$router.push("message");
          }
        })
        .catch(e => {
          console.log(e);
        });
    },
    // auth() {
    //   if (this.userInfo.openid == null) {
    //     this.$router.push("/follow");
    //     return false;
    //   }
    //   return true;
    // },
    init() {
      // if (!this.$store.state.isDebug) {
      //   let passed = this.auth();
      //   if (!passed) {
      //     return;
      //   }
      // }
      this.getStep();
      this.valueList = new Array(_checkList.length).fill(false);
    },
    addInfo() {
      this.$router.push("/info");
    }
  },
  created() {
    this.init();
  }
};
</script>

<style scoped lang="less">
@import "../assets/css/switch.css";

.title {
  color: #e84543;
  font-size: 20px;
  font-weight: bold;
  text-align: left;
  padding: 0 15px;
}

.switch {
  margin-top: 10px;
  display: flex;
  justify-content: space-between;
}

.card-content {
  margin: 10px 15px 0px 15px;
  .desc {
    padding-top: 15px;
    font-size: 16px;
    line-height: 1.6em;
    text-indent: 2.3em;
    letter-spacing: 0.15em;
    text-align: left;
  }
}

.vote {
  margin-bottom: 25px;
}

.submit {
  margin: 10px 15px 25px 15px;
}
.weui-cell__bd p {
  text-align: left;
}
</style>
