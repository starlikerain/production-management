<!--
 * @Author: 龚铱白
 * @Date: 2019-08-16 11:38:00
 * @LastEditors: 龚铱白
 * @LastEditTime: 2019-08-19 14:11:31
 * @Description: 
 -->
<template>
  <div>
    <el-form ref="form"
             :model="form"
             label-width="100px"
             inline
             class="eidtform"
             :rules="rules"
             status-icon>
      <div class="form-title f-fs18 f-tac f-mb30 f-mt15">提交批发退货单</div>
      <el-form-item label="退货日期"
                    prop="jhrq"
                    class="f4">{{form.jhrq}}</el-form-item>
      <el-form-item label="结算状态"
                    class="f4">{{jszt}}</el-form-item>
      <el-form-item label="支付方式"
                    class="f4">{{zffs}}</el-form-item>
      <el-form-item label="经办人"
                    prop="jbr"
                    class="f4">{{form.jbr}}</el-form-item>

      <!-- 以上是订单的基础信息 -->
      <div class="f-df f-mb10">
        <div class="f-f1">
          退货总金额：
          <span class="f-red">{{getTotalPrice}}元</span>
        </div>

      </div>

      <el-table :data="form.listDetail"
                empty-text="请选择单据"
                style="width: 100%;margin-bottom: 20px;">
        <el-table-column type="index"
                         label="序号"
                         width="50"></el-table-column>
        <el-table-column label="商品名称"
                         prop="agriProdName"></el-table-column>
        <el-table-column label="规格型号"
                         prop="model"></el-table-column>
        <el-table-column label="单价"
                         prop="zhrkdj"></el-table-column>
        <el-table-column label="可退数量"
                         prop="kthsl"></el-table-column>
        <el-table-column label="退货数量"
                         prop="thsl"></el-table-column>

        <el-table-column label="小计"
                         width="250"
                         prop="jhje">
          <template slot-scope="scope">
            {{isNaN(Number(Number(scope.row.zhrkdj)*Number(scope.row.thsl)).toFixed(2))?0.00:Number(Number(scope.row.zhrkdj)*Number(scope.row.thsl)).toFixed(2)}}
          </template>
        </el-table-column>

      </el-table>

      <div class="f-mb30"
           style="border-top: solid 1px #efefef;">
        <div style="line-height: 60px;">制单人 ：{{this.userInfo.yhxm}}</div>
      </div>

      <div class="f-tac f-mt20">
        <el-button round
                   type="primary"
                   @click="submitForm"
                   :disabled="disabled">{{disabled?'提交中':'提 交'}}
        </el-button>
        <el-button round
                   @click="closeForm(false)">取 消</el-button>
      </div>
    </el-form>

  </div>
</template>

<script>
import { mapGetters } from "vuex";
import { exec } from "common/tools";
import timeFormat from "common/tools/timeFormat";
import fetch from "fetch/axios/";

export default {
  data () {
    const commonValidate = {
      required: true,
      message: "必填项",
      trigger: "blur"
    };
    return {
      disabled: false,
      selNcpVisable: false,
      supNameDialogVisable: false,
      totalPrice: "",
      addTime: timeFormat.one(new Date()),
      selIds: [],
      djBean: "",
      jszt: "",
      zffs: "",
      rules: {
        jbr: commonValidate,
        khmc: commonValidate,
        jhrq: commonValidate,
        prodDate: commonValidate
      },
      form: {
        id: "",
        khid: "",
        khmc: "",
        khdz: "",
        xgdjh: "",
        xgdjid: "",
        jzzt: "",
        jbr: "", //联系人
        jhdh: "",//退货单号
        listDetail: []
      }
    };
  },
  props: {
    id: {
      type: [String, Number],
      default: ""
    }
  },
  computed: {
    ...mapGetters({
      userInfo: "getUserInfo"
    }),
    getTotalPrice: function () {
      var array = this.form.listDetail || [];
      var totalPrice = 0;
      for (var item of array) {
        if (!isNaN(Number(item.zhrkdj) * Number(item.thsl))) {
          totalPrice += Number(item.zhrkdj) * Number(item.thsl);
        } else {
          totalPrice += 0;
        }
      }
      this.totalPrice = totalPrice;
      return totalPrice.toFixed(2);
    }
  },
  created () {
    this.getOrderInfo();
  },
  methods: {
    getOrderInfo () {
      fetch({
        url: "/nzsyTh/getThdById/",
        method: "get",
        data: {
          thdId: this.id
        }
      })
        .then(res => {
          res.bean ? (this.form = res.bean) : "";
          this.form.listDetail = res.bean.thmx;
          this.form.xgdjh = res.bean.xgdjh;
          this.form.xgdjId = res.bean.xgdjId;
          this.form.jhdh = res.bean.jhdh;
          for (var i = 0; i < this.form.listDetail.length; i++) {
            console.log(this.form.listDetail[i])
            this.form.listDetail[i].zhrkdj = this.form.listDetail[i].dj;
            this.form.listDetail[i].kthsl = this.form.listDetail[i].dqkthsl;
          }
          const jzzt = res.bean.jzzt;
          if (jzzt == 1) {
            this.jszt = "已结算";
          } else {
            this.jszt = "未结算";
          }
          const jzfs = res.bean.jzfs;
          if (jzfs == 1) {
            this.zffs = "现金";
          } else if (jzfs == 2) {
            this.zffs = "支付宝";
          } else if (jzfs == 3) {
            this.zffs = "微信";
          } else if (jzfs == 4) {
            this.zffs = "银联";
          }

        })
        .catch(() => {
          this.loading = false;
        });
    },
    submitForm () {
      for (var i = 0; i < this.form.listDetail.length; i++) {
        if (this.form.listDetail[i].thsl > this.form.listDetail[i].dqkthsl) {
          this.$message.error("存在超额退货商品！");
          this.disabled = false;
          return false;
        }
      }
      fetch({
        url: '/nzsyTh/updateThdById/',
        method: 'post',
        data: {
          thdId: this.id,
          zt: "1"
        }
      }).then(() => {
        this.$message.success("操作成功！");
        this.closeForm(true);
        this.disabled = false;
      }).catch(() => {
        this.$message.error("操作失败！");
        this.disabled = false;
      });
    },
    // submitForm () {
    //   this.$refs.form.validate(valid => {
    //     if (valid) {
    //       if (!this.form.listDetail.length) {
    //         this.$message.error("请选择单据！");
    //         return false;
    //       }
    //       //防止二次提交
    //       this.disabled = true;
    //       //用户参数
    //       var isValid = 1;
    //       let qyid = this.userInfo.bizId;
    //       let cjrid = this.userInfo.bizUserId;
    //       let cjrmc = this.userInfo.yhxm;
    //       var mxIds = [];
    //       var thNums = [];
    //       this.form.listDetail.forEach(item => {
    //         if (
    //           item.xgdjid == "" ||
    //           item.thsl == ""
    //         ) {
    //           isValid = 2;
    //           return;
    //         }
    //         if (item.thsl > item.kthsl) {
    //           console.log("退货数量===" + item.thsl);
    //           console.log("可退货数量===" + item.kthsl);
    //           isValid = 3; //isValid = 3 -> 判断输入的收款金额不能大于应收金额
    //           return;
    //         }
    //         if (this.id != 0 && this.id != null && this.id != undefined) {
    //           mxIds.push(item.xgmxId);
    //         } else {
    //           mxIds.push(item.id);
    //         }
    //         thNums.push(item.thsl);
    //       });
    //       if (isValid == 2) {
    //         this.$message.warning("请完善订单商品信息");
    //         this.disabled = false;
    //         return;
    //       }
    //       if (isValid == 3) {
    //         this.$message.warning("退货数量不应该大于可退货数量");
    //         this.disabled = false;
    //         return false;
    //       }
    //       let params = {
    //         id: this.id,
    //         qyid: qyid + "",
    //         jhdh: this.form.jhdh + "",
    //         khid: this.form.khid + "",
    //         khmc: this.form.khmc + "",
    //         jhrq: this.form.jhrq + "",
    //         jbr: this.form.jbr,
    //         thlx: "1",
    //         zdr: cjrmc,
    //         xgdjId: this.form.xgdjid + "",
    //         xgdjh: this.form.xgdjh + "",
    //         zt: "1",
    //         cjrid: cjrid + "",
    //         cjrmc: cjrmc + "",
    //         je: this.totalPrice + "",
    //         mxIds: String(mxIds),
    //         thNums: String(thNums),
    //         jzzt: this.form.jzzt + "",
    //         jzfs: this.form.jzfs + "",
    //       };
    //
    //       fetch({
    //         method: "post",
    //         url: "/nzsyTh/saveTh/",
    //         data: params
    //       })
    //         .then(() => {
    //           this.$message.success("操作成功！");
    //           this.closeForm(true);
    //           this.disabled = false;
    //         })
    //         .catch(() => {
    //           this.$message.error("操作失败！");
    //           this.disabled = false;
    //         });
    //     } else {
    //       this.$message.error("请按要求填写");
    //       return false;
    //     }
    //   });
    // },

    closeForm (fresh = false) {
      this.$emit("closeForm", fresh);
    },
    close (isRefresh = true) {
      this.$emit("close", isRefresh);
    }
  },
}
</script>


<style scoped lang="scss">
.eidtform {
  // width:100%;
  margin: 0 30px;
  box-sizing: border-box;
}

.f4 {
  width: calc(25% - 15px);
  box-sizing: border-box;
}

.f2 {
  width: calc(50% - 15px);
  box-sizing: border-box;
}
</style>

<style>
.eidtform .el-form-item .el-form-item__content {
  width: calc(100% - 100px);
}

.eidtform .el-date-editor.el-input,
.eidtform .el-date-editor.el-input__inner {
  width: 100%;
}
</style>
