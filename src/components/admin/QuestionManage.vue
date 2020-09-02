<template>
  <div>
    <h2>问题管理</h2>
    <el-table :data="questionList">
      <el-table-column label="id" prop="id"></el-table-column>
      <el-table-column label="title" prop="title"></el-table-column>
      <!--      <el-table-column label="text" prop="text"></el-table-column>-->
      <el-table-column label="score" prop="score"></el-table-column>
      <!--      <el-table-column label="idSchema" prop="idSchema"></el-table-column>-->
      <el-table-column label="opt">
        <template slot-scope="scope">
          <el-button size="mini" @click="handleAnswer(scope.$index, scope.row)">Answer</el-button>
          <el-button size="mini" @click="handleStatistics(scope.$index, scope.row)">Statistics</el-button>
          <el-button size="mini" @click="handleDeleteQuestion(scope.$index, scope.row)">Delete</el-button>
        </template>
      </el-table-column>
    </el-table>

    <el-form ref="newQuestionForm" :model="newQuestionForm" label-position="left" v-show="showQuestionForm">
      <el-form-item label="title">
        <el-input v-model="newQuestionForm.title"></el-input>
      </el-form-item>

      <el-form-item label="text">
        <el-input v-model="newQuestionForm.text"></el-input>
      </el-form-item>

      <el-form-item label="score">
        <el-input v-model="newQuestionForm.score"></el-input>
      </el-form-item>

      <el-form-item label="database">

        <el-select v-model="newQuestionForm.idSchema" placeholder="请选择一个数据库">
          <el-option v-for="item in databaseOptions" :key="item.value" :label="item.label" :value="item.value">
          </el-option>
        </el-select>
      </el-form-item>

      <el-form-item>
        <el-button type="primary" @click="addQuestion">提交</el-button>
      </el-form-item>

    </el-form>

    <div v-show="showStats">
      <h2>Statistics of {{ currentQuestionTitle }}</h2>
      <h4>Correct Rate:{{ currentQuestionCorrectRate }}</h4>
      <el-table :data="stats">
        <el-table-column label="Submit ID" prop="submit_id"></el-table-column>
        <el-table-column label="SID" prop="sid"></el-table-column>
        <el-table-column label="Name" prop="sname"></el-table-column>
        <el-table-column label="Correctness" prop="correctness"></el-table-column>
        <el-table-column label="Answer">
          <template slot-scope="scope">
          <el-button size="mini" @click="handleCheckResult(scope.$index, scope.row)">Check</el-button>
          </template>
        </el-table-column>
      </el-table>
    </div>

    <div v-show="showResult">
      <h4>Student Answer:</h4>
      <p>{{studentAnswer}}</p>
      <h4>Correct Answer:</h4>
      <p>{{correctAnswer}}</p>
      <h4>Student Result:</h4>
      <p>{{studentResult}}</p>
    </div>
  </div>
</template>

<script>
export default {
  name: "QuestionManage",
  data() {
    return {
      questionList: [],
      stats: [],
      newQuestionForm: {
        idSchema: '',
        title: '',
        text: '',
        score: '',
      },
      databaseList: [],
      databaseOptions: [],
      showQuestionForm: true,
      showStats: false,
      showResult: false,
      currentQuestionId: -1,
      currentQuestionTitle: '',
      currentQuestionCorrectRate: '0',
      currentSubmitId: -1,
      studentAnswer:'',
      correctAnswer:'',
      studentResult:''
    }
  },
  mounted() {
    this.getQuestionList()
  },
  methods: {
    addQuestion() {
      this.newQuestionForm['session'] = this.$store.getters.Token
      this.$axios.post('/question', this.newQuestionForm).then(res => {
        this.$message.success('成功')
        this.getQuestionList()
        this.newQuestionForm.idSchema = ''
        this.newQuestionForm.score = ''
        this.newQuestionForm.text = ''
        this.newQuestionForm.title = ''
      }).catch(res => {
        this.$message.error(res.toString())
      })

    },
    getQuestionList() {
      this.getDatabaseList()
      this.questionList = []
      this.$axios.get('/question', {headers: {'session': this.$store.getters.Token}}).then(res => {
        this.questionList = res.data['data']
      }).catch(res => {
        this.$message.error(res.toString())
      })
    },
    handleAnswer(index, row) {
      this.$router.push({
        name: 'AnswerManage',
        params: {
          'idQuestion': row['id'],
          'row': row
        }
      })
    },
    handleDeleteQuestion(index, row) {
      this.$axios.delete('/question/' + row.id, {
        data: {
          'session': this.$store.getters.Token,
        }
      }).then(res => {
        this.$message.success('done')
        this.getQuestionList()
      }).catch(res => {
        this.$message.error(res.toString())
      })
    },
    handleStatistics(index, row) {
      if (this.showStats === true && this.currentQuestionId === row.id) {
        this.showStats = false;
        this.showQuestionForm = true;
        this.currentQuestionId = -1;
        return;
      }
      this.showQuestionForm = false;
      this.showStats = true;
      this.currentQuestionId = row.id;
      this.currentQuestionTitle = row.title;
      this.$axios.get('/question/' + this.currentQuestionId + '/correct_rate',
          {headers: {'session': this.$store.getters.Token}}).then(
          res => {
            this.currentQuestionCorrectRate = (res.data['rate'] * 100).toFixed(1) + '%';
          }
      ).catch(res => {
        this.$message.error(res.toString())
      });
      this.$axios.get('/question/' + this.currentQuestionId + '/stats',
          {headers: {'session': this.$store.getters.Token}}).then(
              res => {
                this.stats = res.data['data'];
              }
      )
    },
    handleCheckResult(index, row) {
      if (this.showResult === true && this.currentSubmitId === row.submit_id) {
        this.showResult = false;
        this.currentSubmitId = -1;
        return;
      }
      this.$axios.get('/check_result/'+ row.submit_id, {headers: {'session': this.$store.getters.Token}}).then(
          res => {
            this.studentAnswer = res.data['answer'];
            this.correctAnswer = res.data['correct'];
            this.studentResult = res.data['result'];
          }
      );
      this.currentSubmitId = row.submit_id;
      this.showResult = true;
    },
    getDatabaseList() {
      this.databaseList = []
      this.databaseOptions = []
      this.$axios.get('/schema', {headers: {'session': this.$store.getters.Token}}).then(res => {
        this.databaseList = res.data['data']
        for (var i = 0; i < this.databaseList.length; i++) {
          this.databaseOptions.push({
            'value': this.databaseList[i]['id'],
            'label': this.databaseList[i]['name']
          })
        }

      }).catch(res => {
        this.$message.error(res.toString())
      })
    },
  }
}
</script>

<style scoped>

</style>