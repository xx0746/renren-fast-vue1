<template>
  <div class="mod-user">
    <el-form :inline="true" :model="dataForm" @keyup.enter.native="getDataList()">
      <el-form-item>
        <el-date-picker
          style="width: 170px"
          :clearable="false"
          v-model="dataForm.weekvalue"
          :format="dataForm.year +'-' +dataForm.month +'月第' + dataForm.week + '周'"
          @change="weekChange"
          value-format="yyyy-MM-dd"
          class="inp"
          size="medium"
          type="week"
          placeholder="请选择"
          :picker-options="{'firstDayOfWeek': 1}"
          end-placeholder
        ></el-date-picker>
      </el-form-item>
      <el-form-item >
        <!--v-if="visible"-->
        <el-button type="primary" @click="addHandle()" :disabled="dataForm.disableFlag">新增</el-button>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" @click="submitHandle()" :disabled="dataForm.disableFlag">提交</el-button>
      </el-form-item>
      <div style="margin-bottom: 10px">
        <span>({{this.weekStart}}日至{{this.weekEnd}}日)</span>
        <span style="font-size: 1px">温馨提示：1.审批中的工时，无法操作。2.新增或者删除工时后请点击提交，否则无法保存。</span>
      </div>
    </el-form>

    <el-table
      :data="dataList"
      border
      v-loading="dataListLoading"
      :header-cell-style="{background:'#eef1f6',color:'#606266'}">
      <el-table-column label="序号" type="index" header-align="center" align="center" width="50"></el-table-column>
      <el-table-column label="工作类型" header-align="center" align="center" >
        <template slot-scope="scope">
          <span>{{scope.row.oneName}}-{{scope.row.twoName}}</span>
        </template>
      </el-table-column>
      <el-table-column label="使用工时" prop="ut" header-align="center" align="center"></el-table-column>
      <el-table-column label="工时状态"  header-align="center" align="center">
        <template slot-scope="scope">
          <span v-if="scope.row.status == '0'">待审批</span>
          <span v-if="scope.row.status == '1'">审批中</span>
          <span v-if="scope.row.status == '2'">审批完成</span>
          <span v-if="scope.row.status == '-1'">待提交</span>
        </template>
      </el-table-column>
      <el-table-column label="项目编码" header-align="center" align="center">
        <template slot-scope="scope">{{dataForm.projectCode}}</template>
      </el-table-column>
      <el-table-column label="项目名称" header-align="center" align="center">
        <template slot-scope="scope">{{dataForm.projectName}}</template>
      </el-table-column>

      <el-table-column v-if="visible" label="操作" header-align="center" align="center" width="50">
        <template slot-scope="scope">
          <el-button type="text" size="small" @click="deleteHandle(scope.row.oneId, scope.row.twoId)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>
    <el-pagination
      v-if="!visible"
      @size-change="sizeChangeHandle"
      @current-change="currentChangeHandle"
      :current-page="pageIndex"
      :page-sizes="[10, 20, 50, 100]"
      :page-size="pageSize"
      :total="totalPage"
      layout="total, sizes, prev, pager, next, jumper">
    </el-pagination>
    <!-- 弹窗,  新增-->
    <write-add ref="writeAdd" @refreshDataList="putDataList"></write-add>
  </div>
</template>

<script>
  import WriteAdd from './writeAdd'

  export default {
    data () {
      return {
        visible: false,
        dataForm: {
          disableFlag: true,
          weekvalue: '',
          year: '',
          month: '',
          week: '',
          projectId: '',
          projectCode: '',
          projectName: ''
        },
        utstatus: 0,
        weekStart: '',
        weekEnd: '',
        dataList: [],
        optionWorks: [],
        pageIndex: 1,
        pageSize: 10,
        totalPage: 0,
        dataListLoading: false
      }
    },
    components: {
      WriteAdd
    },
    activated () {
      this.dataForm.projectId = this.$route.params.projectId
      this.dataForm.projectCode = this.$route.params.projectCode
      this.dataForm.projectName = this.$route.params.projectName
      this.getDay()
      this.getDataList()
      this.getWorkTypeList()
    },
    methods: {
      isADD () {
        if (this.utstatus === 0 || this.utstatus === undefined) {
          this.dataForm.disableFlag = false
          this.visible = true
        } else {
          this.dataForm.disableFlag = true
          this.visible = false
        }
      },
      // 获取数据列表
      getDataList () {
        if (this.dataForm.projectId === '' || this.dataForm.projectId === undefined) {
          this.$message.info('未找到对应的项目，无法填报工时')
          return
        }
        this.dataListLoading = true
        this.$http({
          url: this.$http.adornUrl('/utWrite/writeList'),
          method: 'post',
          params: this.$http.adornParams({
            'current': this.pageIndex,
            'size': this.pageSize,
            'year': this.dataForm.year,
            'month': this.dataForm.month,
            'week': this.dataForm.week,
            /* 'type': '0', */
            'projectId': this.dataForm.projectId
          })
        }).then(({data}) => {
          if (data && data.code === 0) {
            this.dataList = data.dataList
            this.totalPage = data.total
            this.utstatus = data.status
            this.isADD()
          } else {
            this.dataList = []
            this.totalPage = 0
          }
          this.dataListLoading = false
        })
      },
      // 每页数
      sizeChangeHandle (val) {
        this.pageSize = val
        this.pageIndex = 1
        this.getDataList()
      },
      // 当前页
      currentChangeHandle (val) {
        this.pageIndex = val
        this.getDataList()
      },
      putDataList (nodesLable, nodesValue, workTime) {
        // 如果该工作类型已添加，先删除再保存
        this.dataList.some((item, i) => {
          if (item.oneId === nodesValue[0] && item.twoId === nodesValue[1]) {
            this.dataList.splice(i, 1)
            return true
          }
        })
        // 保存
        this.dataList.push({
          'oneId': nodesValue[0],
          'oneName': nodesLable[0],
          'twoId': nodesValue[1],
          'twoName': nodesLable[1],
          'status': '-1',
          'ut': workTime
        })
      },
      deleteHandle (oneId, twoId) {
        this.dataList.some((item, i) => {
          if (item.oneId === oneId && item.twoId === twoId) {
            this.dataList.splice(i, 1)
            return true
          }
        })
        /* this.$confirm(`确定删除吗?`, '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.dataList.some((item, i) => {
            if (item.oneId === oneId && item.twoId === twoId) {
              this.dataList.splice(i, 1)
              return true
            }
          })
        }).catch(() => {
        }) */
      },
      // 新增
      addHandle () {
        this.$refs.writeAdd.init(this.projectId,
            this.dataForm.weekvalue.substring(0, 7) + '月第' + this.dataForm.week + '周（' +
            this.weekStart + '日至' + this.weekEnd + '日)', this.optionWorks)
      },
      getWorkTypeList () {
        if (this.dataForm.projectId === '' || this.dataForm.projectId === undefined) {
          // this.$message.info('未找到对应的项目，无法填报工时')
          return
        }
        this.$http({
          url: this.$http.adornUrl('/workType/getWorkTypeList'),
          method: 'post',
          data: this.$http.adornData({
            'current': 1,
            'size': 100
          })
        }).then(({data}) => {
          if (data && data.code === 0) {
            /* data.dataList.some((item, i) => {
              item.children.some((item1, i) => {
                delete item1.children
              })
            }) */
            this.optionWorks = data.dataList
          }
        })
      },
      submitHandle () {
        if (this.dataList.length <= 0) {
          this.$message.info('请先添加工作类型及工时')
          return
        }
        this.$confirm(`提交后无法修改，确定提交吗?`, '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.$http({
            url: this.$http.adornUrl(`/utWrite/writeAdd`),
            method: 'post',
            data: this.$http.adornData({
              'year': this.dataForm.year,
              'month': this.dataForm.month,
              'week': this.dataForm.week,
              'projectId': this.dataForm.projectId,
              'dataList': this.dataList
            })
          }).then(({data}) => {
            if (data && data.code === 0) {
              this.$message({
                message: '操作成功',
                type: 'success',
                duration: 1500,
                onClose: () => {
                  // this.$message.info('提交完成')
                  this.getDataList()
                }
              })
            } else {
              this.$message.error(data.msg)
            }
          })
        })
      },
      weekChange (val) {
        if (val) {
          /* if (new Date(val).getTime() > new Date().getTime()) {
            this.$message.info('不能大于当前周')
            return
          } */
          this.dataForm.week = this.getWeekInMonth(new Date(val))
          this.getDataList()
        }
      },
      getDay () {
        let date = new Date()
        let year = date.getFullYear()
        let month = date.getMonth() + 1
        let day = date.getDate()
        if (month < 10) {
          month = '0' + month
        }
        if (day < 10) {
          day = '0' + day
        }
        let nowDay = year + '-' + month + '-' + day
        this.dataForm.week = this.getWeekInMonth(new Date(nowDay))
        this.dataForm.weekvalue = year + '-' + month + '-' + day
      },

      // 根据日期判断是月的第几周
      getWeekInMonth (t) {
        if (t === undefined || t === '' || t === null) {
          t = new Date()
        } else {
          let now = new Date(t)
          let nowTime = now.getTime()
          let day = now.getDay() || 7
          let oneDayTime = 24 * 60 * 60 * 1000
          let MondayTime = nowTime - (day - 1) * oneDayTime// 显示周一
          let SundayTime = nowTime + (7 - day) * oneDayTime// 显示周日
          let weekStartDate = new Date(MondayTime)
          this.weekStart = this.formatDate(new Date(MondayTime))
          this.weekEnd = this.formatDate(new Date(SundayTime))
          this.dataForm.year = weekStartDate.getFullYear() + ''
          let month = weekStartDate.getMonth() + 1
          if (month < 10) {
            month = '0' + month
          }
          this.dataForm.month = month + ''
          let date = weekStartDate.getDate()
          // eslint-disable-next-line no-unused-vars
          weekStartDate.setDate(1)
          let d = weekStartDate.getDay()
          let fisrtWeekend = d
          if (d === 0) {
            fisrtWeekend = 1
          } else {
            fisrtWeekend = 7 - d + 1
          }
          if (date <= fisrtWeekend) {
            return '1'
          } else {
            return 1 + Math.ceil((date - fisrtWeekend) / 7) + ''
          }
        }
      },
      formatDate (date) {
        let myyear = date.getFullYear()
        let mymonth = date.getMonth() + 1
        let myweekday = date.getDate()
        if (mymonth < 10) {
          mymonth = '0' + mymonth
        }
        if (myweekday < 10) {
          myweekday = '0' + myweekday
        }
        return myyear + '-' + mymonth + '-' + myweekday
      }
    }
  }
</script>
