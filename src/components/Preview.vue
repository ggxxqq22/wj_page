<!--
程序名：问卷填写页面
功能：用户打开问卷链接对问卷进行填写
-->
<template>
  <div class="display">
    <div class="content">
      <h3>{{title}}</h3>
      <div class="top" v-if="desc!='' && !flag">
        {{desc}}
      </div>
      <el-button type="primary" style="margin-top: 25px;" @click="() => {flag = !flag}" v-if="!flag">开始填写</el-button>
      <template v-if="flag">
        <el-card class="box-card" v-for="(item,index) in pageParams.pageArr[num-1]">
          <div slot="header" class="clearfix">
            <div class="questionTitle">
              <!--显示必填标识-->
              <span style="color: #F56C6C;">
              <span v-if="item.must">*</span>
              <span v-else>&nbsp;</span>
            </span>
              {{(item.trueNum)+'.'+item.title}}
            </div>
          </div>
          <!--单选题展示-->
          <div class="text item" v-if="item.type=='radio' && item.title !== 'divide'" v-for="optionItem in item.options">
            <el-radio v-model="item.radioValue" :label="optionItem.id" style="margin: 5px;">{{ optionItem.title }}</el-radio>
          </div>
          <!--排序题展示-->
          <template v-if="item.type=='rank'">
            <draggable v-model="item.rankValue"  chosenClass="chosen" forceFallback="true" animation="300">
              <transition-group>
                <div class="rank_item" v-for="(element,ind) in item.rankValue" :key="element.id">{{(ind+1) + '. ' + element.title}}</div>
              </transition-group>
            </draggable>
          </template>
          <!--多选题展示-->
          <el-checkbox-group v-if="item.type=='checkbox'" v-model="item.checkboxValue">
            <div class="text item"  v-for="optionItem in item.options">
              <el-checkbox :label="optionItem.id" style="margin: 5px;">{{ optionItem.title }}</el-checkbox>
            </div>
          </el-checkbox-group>
          <!--填空题展示-->
          <el-input
            v-if="item.type=='text'"
            type="textarea"
            :rows="item.row"
            v-model="item.textValue"
            resize="none">
          </el-input>
          <!--矩阵量表题展示-->
          <div v-if="item.type === 'nps'" class="nps">
            <el-row>
              <el-col :span="8">
                &nbsp;
              </el-col>
              <el-col v-for="(lev, i) in item.level.split(',')" :key="i" :span="calcSpan(item.level)" class="nps_radio">
                {{ lev }}
              </el-col>
            </el-row>
            <el-row v-for="(row, rowIndex) in item.options" :key="rowIndex" class="nps_row">
              <el-col :span="8" class="nps_title">
                {{ row.title }}
              </el-col>
              <el-col v-for="(lev, i) in item.level.split(',')" :key="i" :span="calcSpan(item.level)" class="nps_radio">
                <el-radio :label="i" v-model="item.npsValue[rowIndex]">&nbsp;</el-radio>
              </el-col>
            </el-row>
          </div>
        </el-card>
        <el-button type="primary" style="margin: 5px;" @click="submit" :loading="submitLoading" v-if="num === pageSize">{{submitText}}</el-button>
        <el-pagination
          background
          hide-on-single-page
          layout="prev, pager, next"
          class="page"
          :current-page.sync="num"
          :page-size="1"
          :total="pageParams.pageSize">5io
        </el-pagination>
      </template>
      <div class="bottom">
        <el-link type="info" href="/index">ZJUCSS问卷&nbsp;提供技术支持</el-link>
      </div>
    </div>
  </div>
</template>
<script>
import {answerOpera} from './api'
import draggable from 'vuedraggable'
export default {
  components: {
    draggable,
  },
  data(){
    return{
      num: 0,
      flag: false,
      dialogShow:false,
      dialogTitle:'',
      dialogType:1,//1添加 2修改
      oldItem:null,//编辑中问题的对象
      willAddQuestion:{
        type:'',
        title:'',
        options:[''],
        text:'',
        row:1,
      },
      allType:[
        {
          value:'radio',
          label:'单选题',
        },
        {
          value:'checkbox',
          label:'多选题',
        },
        {
          value:'text',
          label:'填空题',
        },
        {
          value:'nps',
          label:'矩阵量表题',
        },
        {
          value: 'rank',
          label: '排序题'
        }
      ],
      title:'',
      desc:'',
      detail:[],
      startTimestamp:0,//填写问卷开始时间戳 毫秒
      submitLoading:false,//提交按钮 加载中状态
      submitText:'提交',//提交按钮文字
    }
  },
  computed: {
    pageSize() {
      let res = 1
      for(let i=0;i<this.detail.length;i++) {
        if(this.detail[i].type === 'page' && i < this.detail.length - 1) {
          res++
          while(this.detail[i+1] && this.detail[i+1].type === 'page') {
            i++
          }
        }
      }
      return res
    },
    pageParams() {
      const res = {
        pageSize: 1,
        pageArr: []
      }
      let index = 0
      for(let i=0;i<this.detail.length;i++) {
        if(this.detail[i].type === 'page') {
          if(i !== this.detail.length - 1) {
            index++
            res.pageSize++
          }
        } else {
          this.detail[i].trueNum = i - index + 1
          if(!res.pageArr[index]) res.pageArr[index] = [this.detail[i]]
          else res.pageArr[index].push(this.detail[i])
        }
      }
      return res
    }
  },
  mounted(){
    var wjId=this.$route.params.id;
    answerOpera({
      opera_type:'get_info',
      wjId:wjId,
      username:'test'//增加登录验证后不需传递（后端从session获取）
    })
      .then(data=>{
        //console.log(data);
        if(data.code==0){
          this.title=data.title;
          this.desc=data.desc;
          this.detail=data.detail;
          let len = this.detail.length
          for(let i=0;i<len;i++) {
            // 删除重复的page
            if(this.detail[i] && this.detail[i].type === 'page' && this.detail[i+1] && this.detail[i+1].type === 'page') {
              this.detail.splice(i, 1)
              i--
            }
          }
          document.title=data.title;
          console.log('--------')
          console.log(this.detail)
        }
        else{
          this.$message({
            type: 'error',
            message: data.msg
          });
        }
      })
    this.startTimestamp=new Date().getTime();//时间戳 毫秒
  },
  methods:{
    calcSpan(level) {
      return parseInt(16 / level.split(',').length)
    },
    //提交问卷
    submit(){
      this.$router.push({path:'/thankyou'});//跳到欢迎页
    }
  }
}
</script>
<style scoped>
.nps {
  font-size: 14px;
}
.display{
  text-align: center;
  padding: 20px;
}
.page {
  margin-top: 30px;
}
.divide {
  border: 3px solid #409EFF;
}
.display .top{
  color: #606266;
  padding: 20px 10px 10px 10px;
  border-bottom: 3px solid #409EFF;
  font-size: 15px;
  line-height: 22px;
  text-align: left;
}
.display .content{
  width: 100%;
  max-width: 800px;
  display: inline-block;
  text-align: center;
}
.display .box-card{
  text-align: left;
  width: 100%;
  margin:10px 0 10px 0;
}
.display .bottom{
  margin: 20px 10px 20px 10px;
  color: #909399;
}
.display a:link,a:visited,a:active {
  color: #909399;
  text-decoration:none;
}
.rank_item {
  padding: 6px;
  background-color: #fdfdfd;
  border: solid 1px #eee;
  margin-bottom: 10px;
  cursor: move;
}
/*选中样式*/
.chosen {
  border: solid 2px #3089dc !important;
}
.nps_title {
  padding-right: 15px !important;
}
.nps_radio {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100% !important;
  line-height: 1;
  align-self: center;
}
.nps_row {
  display: flex;

}
.nps_row + .nps_row {
  margin-top: 10px;
}
.nps_row >>> .el-radio__label {
  display: none;
}
</style>
