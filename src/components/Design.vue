<!--
程序名：问题设计页面
功能：对问卷中问题的添加、编辑、删除
-->
<template>
  <div class="Design" v-loading="loading" element-loading-text="加载中...">
    <h3>{{ title }}</h3>
    <div class="top" v-if="desc!=''">
      {{ desc }}
    </div>
    <el-card class="box-card" v-for="(item,index) in detail" style="margin: 10px;" :key="index">
      <div slot="header" class="clearfix" v-if="item.type !== 'page'">
        <div class="questionTitle">
          <!--显示必填标识-->
          <span style="color: #F56C6C;">
            <span v-if="item.must">*</span>
            <span v-else>&nbsp;</span>
          </span>
          <span style="color: black;margin-right: 3px;">{{ (questionNum[index] + 1) + '.' }}</span>
          {{ item.title }}
        </div>
        <div style="float: right;">
          <el-button style="padding: 2px" type="text" @click="editorQuestion(item)" v-if="item.title !== 'divide'">编辑
          </el-button>
          <el-button style="padding: 2px;color: #F56C6C" type="text" @click="deleteQuestion(index)">删除</el-button>
        </div>
      </div>
      <!--单选题展示-->
      <div class="text item" v-if="item.type=='radio'" v-for="(option,index) in item.options" :key="index">
        <el-radio v-model="item.radioValue" :label="index" style="margin: 5px;">{{ option.title }}</el-radio>
      </div>
      <!--排序题展示-->
      <draggable v-model="item.rankValue"  chosenClass="chosen" forceFallback="true" animation="300">
        <transition-group>
          <div class="rank_item" v-for="(element,ind) in item.rankValue" :key="element.id">{{(ind+1) + '. ' + element.title}}</div>
        </transition-group>
      </draggable>
      <!--分页展示-->
      <div v-if="item.type === 'page'" class="page">
        <span>分页线</span>
        <el-button style="padding: 2px;color: #F56C6C" type="text" @click="deleteQuestion(index)">删除</el-button>
      </div>
      <!--多选题展示-->
      <el-checkbox-group v-if="item.type=='checkbox'" v-model="item.checkboxValue">
        <div class="text item" v-for="(option,index) in item.options" :key="index">
          <el-checkbox :label="index" style="margin: 5px;">{{ option.title }}</el-checkbox>
        </div>
      </el-checkbox-group>
      <!--填空题展示-->
      <el-input
        v-if="item.type=='text'"
        type="textarea"
        :rows="item.row"
        resize="none"
        v-model="item.textValue">
      </el-input>
      <!--矩阵量表题展示-->
      <template v-if="item.type === 'nps'">
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
      </template>
    </el-card>
    <el-button icon="el-icon-circle-plus" @click="addQuestion" style="margin-top: 10px;">添加题目</el-button>
    <el-button icon="el-icon-circle-plus" @click="addPage" style="margin-top: 10px;">添加分页</el-button>
    <br><br><br><br><br>
    <!--添加题目弹窗-->
    <el-dialog :title="dialogTitle" :visible.sync="dialogShow" :close-on-click-modal="false" class="dialog">
      <el-form ref="form" :model="willAddQuestion" label-width="80px">
        <!--题目类型-->
        <el-form-item label="题目类型" style="width: 100%;">
          <el-select v-model="willAddQuestion.type" placeholder="请选择题目类型" @change="typeChange">
            <el-option
              v-for="item in allType"
              :key="item.value"
              :label="item.label"
              :value="item.value">
            </el-option>
          </el-select>
        </el-form-item>
        <!--是否必填-->
        <el-form-item label="是否必填" style="width: 100%;">
          <el-checkbox v-model="willAddQuestion.must">必填</el-checkbox>
        </el-form-item>
        <!--题目标题-->
        <el-form-item label="题目标题" style="width: 100%;">
          <el-input v-model="willAddQuestion.title" placeholder="请输入标题"></el-input>
        </el-form-item>
        <!--分类-->
        <template v-if="willAddQuestion.type=='radio'||willAddQuestion.type=='checkbox'">
          <el-form-item :label="'选项'+(index+1)" v-for="(item,index) in willAddQuestion.options" :key="index">
            <el-row>
              <el-col :span="16">
                <el-input v-model="item.title" placeholder="请输入选项名" style="width: 90%;"></el-input>
              </el-col>
              <el-col :span="8">
                <el-button type="danger" plain class="" @click="deleteOption(index)">删除选项</el-button>
              </el-col>
            </el-row>

          </el-form-item>
          <el-button type="primary" plain class="addOptionButton" @click="addOption">新增选项</el-button>
        </template>
        <template v-if="willAddQuestion.type=='text'">
          <el-form-item label="填空">
            <el-input type="textarea" :rows="willAddQuestion.row" style="width: 80%" resize="none"></el-input>
          </el-form-item>
          <el-form-item label="行数">
            <el-input-number v-model="willAddQuestion.row" :min="1" :max="10" label="描述文字"></el-input-number>
          </el-form-item>
        </template>
        <template v-if="willAddQuestion.type=='nps'">
          <el-form-item label="量表等级" style="width: 100%;">
            <el-input v-model="willAddQuestion.level" placeholder="请输入量表等级，以半角逗号「,」隔开。示例：非常不同意,比较不同意,比较同意,同意,非常同意" @change="$forceUpdate()"></el-input>
          </el-form-item>
          <el-form-item :label="'问题'+(index+1)" v-for="(item,index) in willAddQuestion.options" :key="index">
            <el-row>
              <el-col :span="16">
                <el-input v-model="item.title" placeholder="请输入子问题名" style="width: 90%;"></el-input>
              </el-col>
              <el-col :span="8">
                <el-button type="danger" plain class="" @click="deleteOption(index)">删除问题</el-button>
              </el-col>
            </el-row>
          </el-form-item>
          <el-button type="primary" plain class="addOptionButton" @click="addOption">新增问题</el-button>
        </template>
        <template v-if="willAddQuestion.type=='rank'">
          <el-form-item :label="'排序项'+(index+1)" v-for="(item,index) in willAddQuestion.options" :key="index">
            <el-row>
              <el-col :span="16">
                <el-input v-model="item.title" placeholder="请输入排序项" style="width: 90%;"></el-input>
              </el-col>
              <el-col :span="8">
                <el-button type="danger" plain class="" @click="deleteOption(index)">删除排序项</el-button>
              </el-col>
            </el-row>

          </el-form-item>
          <el-button type="primary" plain class="addOptionButton" @click="addOption">新增排序项</el-button>
        </template>
        <!--<el-collapse v-if="willAddQuestion.type=='radio'" class="more-options">-->
        <!--  <el-collapse-item title="更多选项">-->
        <!--    <el-form-item label="随机选项">-->
        <!--      <el-checkbox>勾选此选项后该问题的每个选项将不按预定顺序出现，但统计结果中顺序不变</el-checkbox>-->
        <!--    </el-form-item>-->
        <!--    <el-form-item label="随机场景">-->
        <!--      <el-checkbox>勾选此选项后该问题将解析标题中的 <strong>^{"可能语句1"\"可能语句2"}</strong> 结构,随机下发不同的标题内容</el-checkbox>-->
        <!--    </el-form-item>-->
        <!--  </el-collapse-item>-->
        <!--</el-collapse>-->
      </el-form>
      <br>
      <div style="width: 100%;text-align: right">
        <el-button style="margin-left: 10px;" @click="dialogShow=false">取消</el-button>
        <el-button type="primary" style="margin-left: 10px;" @click="checkAddQuestion">完成</el-button>
      </div>
    </el-dialog>
  </div>
</template>
<script>
import {designOpera} from './api'
import draggable from 'vuedraggable'
export default {
  components: {
    draggable,
  },
  data() {
    return {
      loading: false,//页面加载中
      dialogShow: false,
      dialogTitle: '',
      detail: [],
      wjId: 0,
      title: '',
      desc: '',
      willAddQuestion: {
        id: 0,
        type: '',
        title: '',
        options: [
          {
            title: '',//选项标题
            id: 0//选项id
          }
        ],
        row: 1,
        must: true,//是否必填
      },
      allType: [
        {
          value: 'radio',
          label: '单选题',
        },
        {
          value: 'checkbox',
          label: '多选题',
        },
        {
          value: 'text',
          label: '填空题',
        },
        {
          value: 'nps',
          label: '矩阵量表'
        },
        {
          label: '排序题',
          value: 'rank'
        }
      ],
      levelType: [
        "不满意，比较不满意，一般，比较满意，"
      ]
    }
  },
  methods: {
    //计算量表span
    stopFunc() {
      alert('停止编辑')
    },
    calcSpan(level) {
      return parseInt(16 / level.split(',').length)
    },
    //初始化问卷所有问题
    init(wjId, title, desc) {
      this.wjId = wjId;
      this.title = title;
      this.desc = desc;
      this.getQuestionList();
    },
    //获取问题列表(问卷内容)
    getQuestionList() {
      this.detail = [];
      this.loading = true;
      designOpera({
        opera_type: 'get_question_list',
        username: 'test',
        wjId: this.wjId,
      })
        .then(data => {
          this.detail = data.detail;
          console.log(this.questionNum)
          console.log('---------')
          console.log(this.detail)
          console.log('---------')
          this.loading = false;
        })
    },
    //点击添加问题按钮
    addQuestion() {
      if (this.wjId == 0 || this.wjId == null) {
        this.$message({
          type: 'error',
          message: '清先创建问卷!'
        });
        return;
      }
      this.dialogTitle = '添加题目';
      this.willAddQuestion = {
        id: 0,
        type: '',
        title: '',
        options: [
          {
            title: '',//选项标题
            id: 0//选项id
          }
        ],
        row: 1,
        must: true,//是否必填，
      };
      this.dialogShow = true;
    },
    addPage() {
      if (this.wjId == 0 || this.wjId == null) {
        this.$message({
          type: 'error',
          message: '清先创建问卷!'
        });
        return;
      }
      designOpera({
        opera_type: 'add_question',
        username: 'test',
        wjId: this.wjId,
        questionId: 0,
        title: 'page',
        type: 'page',
        options: [],
        row: 1,
        must: true,
      })
        .then(data => {
          if (data.code == 0) {
            this.dialogShow = false;
            this.$message({
              type: 'success',
              message: '保存成功!'
            });
            this.getQuestionList();
          } else {
            this.dialogShow = false;
            this.$message({
              type: 'error',
              message: data.msg
            });
          }
          this.willAddQuestion = {
            id: 0,
            type: '',
            title: '',
            options: [''],
            row: 1,
            must: false,
          };
        });
    },
    //删除问题
    deleteQuestion(index) {
      this.$confirm('确定删除此题目?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        designOpera({
          opera_type: 'delete_question',
          username: 'test',
          questionId: this.detail[index].id,
        })
          .then(data => {
            console.log(data);
            if (data.code == 0) {
              this.detail.splice(index, 1);
              this.$message({
                type: 'success',
                message: '删除成功!'
              });
            } else {
              this.$message({
                type: 'error',
                message: data.msg
              });
            }
          })
      });

    },
    //确认添加/保存题目
    checkAddQuestion() {
      //添加保存问题
      designOpera({
        opera_type: 'add_question',
        username: 'test',
        wjId: this.wjId,
        questionId: this.willAddQuestion.id,
        title: this.willAddQuestion.title,
        type: this.willAddQuestion.type,
        options: this.willAddQuestion.options,
        row: this.willAddQuestion.row,
        must: this.willAddQuestion.must,
        level: this.willAddQuestion.level || ''
      })
        .then(data => {
          console.log(data);
          if (data.code == 0) {
            this.dialogShow = false;
            this.$message({
              type: 'success',
              message: '保存成功!'
            });
            this.getQuestionList();
          } else {
            this.dialogShow = false;
            this.$message({
              type: 'error',
              message: data.msg
            });
          }
          this.willAddQuestion = {
            id: 0,
            type: '',
            title: '',
            options: [''],
            row: 1,
            must: true,
            level: '无'
          };
        });
    },
    //点击编辑问题按钮
    editorQuestion(item) {
      this.willAddQuestion.title = item.title;
      this.willAddQuestion.type = item.type;
      this.willAddQuestion.options = JSON.parse(JSON.stringify(item.options));
      this.willAddQuestion.text = item.text;
      this.willAddQuestion.row = item.row;
      this.willAddQuestion.must = item.must;
      this.willAddQuestion.id = item.id;
      this.willAddQuestion.level = item.level;
      this.dialogTitle = '编辑问题';
      this.dialogShow = true;
    },
    //添加选项
    addOption() {
      this.willAddQuestion.options.push({
        title: '',
        id: 0,
      });
    },
    //删除选项
    deleteOption(index) {
      this.willAddQuestion.options.splice(index, 1);
    },
    //切换问题类型
    typeChange(value) {
      console.log(value);
      this.willAddQuestion.type = value;
      this.willAddQuestion.text = '';
      this.row = 1;
    },
  },
  computed: {
    questionNum() {
      const res = []
      let i = 0
      this.detail.forEach((item, index) => {
        if (item.type !== 'page') {
          res[index] = i
          i++
        } else {
          res[index] = i
        }
      })
      return res
    }
  }
}
</script>
<style scoped>
.Design {

}

.Design .dialog {
  text-align: left;
}

.Design .questionTitle {
  display: inline-block;
  width: 80%;
  font-size: 16px;
  color: #303133;
}

.Design .addOptionButton {
  display: inline-block;
  margin-left: 80px;
}

.box-card {
  width: 100%;
  text-align: left;
}

.Design .top {
  color: #606266;
  margin-left: 20px;
  padding: 0 10px 10px 10px;
  border-bottom: 3px solid #409EFF;
  font-size: 15px;
  line-height: 22px;
  text-align: left;
}

.more-options {
  margin: 20px 0 20px 10px;
}

.page {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-left: 7px;
  padding-bottom: 10px;
  border-bottom: 3px solid #409EFF;
  margin-bottom: -5px;
}
.el-row {
  margin-bottom: 10px;
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
</style>
