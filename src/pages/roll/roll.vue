<template>
  <div>
    <el-breadcrumb separator-class="el-icon-arrow-right" class="crumb">
      <el-breadcrumb-item>重卷记录</el-breadcrumb-item>
      <el-breadcrumb-item>{{castId}}号机组</el-breadcrumb-item>
    </el-breadcrumb>
    <Collapse>
      <el-form class="search_bar" :model="searchForm" :inline="true">
        <el-form-item label="生产日期：">
          <el-date-picker
            v-model="searchForm.date"
            type="daterange"
            :default-time="['00:00:00', '23:59:59']"
            :clearable="false"
            start-placeholder="开始日期"
            end-placeholder="结束日期">
          </el-date-picker>
        </el-form-item>
        <el-form-item label="喷带手：">
          <el-input v-model="searchForm.caster" placeholder="请输入喷带手姓名"></el-input>
        </el-form-item>
        <el-form-item label="炉号：">
          <el-input v-model="searchForm.furnace" placeholder="请输入炉号"></el-input>
        </el-form-item>
        <el-form-item label="重卷：">
          <el-input v-model="searchForm.roller" placeholder="请输入重卷人姓名"></el-input>
        </el-form-item>
        <el-form-item>
          <el-button type="primary" icon="el-icon-search" @click="clickSearch">搜索</el-button>
          <el-button type="primary" icon="el-icon-refresh" @click="reset">重置</el-button>
        </el-form-item>
      </el-form>
    </Collapse>
    <div class="main_bd">
      <el-col class="table_hd">
        <el-button type="primary" icon="el-icon-plus" @click="add" v-if="isAddable">创建重卷记录</el-button>
        <el-button type="primary" icon="el-icon-download" @click="exportExcel" v-if="userinfo.roleId === 1 || userinfo.roleId === 2 || userinfo.roleId === 3" class="pull_right">导出</el-button>
      </el-col>
      <el-table :data="tableData" stripe border style="width:100%" v-loading="loading" ref="table" :height="tableHeight"> 
        <el-table-column prop="furnace" label="炉号" align="center" min-width="170px"></el-table-column>
        <el-table-column prop="ribbonTypeName" label="材质" align="center" min-width="80px"></el-table-column>
        <el-table-column prop="ribbonWidth" label="规格" align="center"></el-table-column>
        <el-table-column prop="castDate" label="生产日期" align="center" :formatter="dateFormat" min-width="110px"></el-table-column>
        <el-table-column prop="caster" label="喷带手" align="center" min-width="100px"></el-table-column>
        <el-table-column prop="coilNumber" label="盘号" align="center" width="80px"></el-table-column>
        <el-table-column prop="diameter" label="外径(mm)" align="center" width="100px"></el-table-column>
        <el-table-column prop="coilWeight" label="重量(kg)" align="center" width="110px"></el-table-column>
        <el-table-column prop="rollMachine" label="机器编号" align="center" width="110px"></el-table-column>
        <el-table-column prop="roller" label="重卷人员" align="center" width="110px"></el-table-column>
        <el-table-column prop="createdAt" label="重卷日期" align="center" :formatter="rollDateFormat" min-width="110px"></el-table-column>
        <el-table-column label="是否平整" align="center" width="80px">
          <template slot-scope="scope">
            <span :class="{text_danger: scope.row.isFlat === '否'}">{{scope.row.isFlat}}</span>
          </template>
        </el-table-column>
        <el-table-column label="操作" align="center" width="150px">
          <template slot-scope="scope">
            <el-button size="mini" type="primary" @click="edit(scope.row)" v-if="isEditable" :disabled="scope.row.roller !== userinfo.adminname && userinfo.roleId !== 1">修改</el-button>
            <!-- <el-button size="mini" type="danger" @click="del(scope.row)">删除</el-button> -->
          </template>
        </el-table-column>
      </el-table>
      <el-pagination
        background
        layout="total,prev,pager,next"
        :total="pageConfig.total"
        :current-page.sync="pageConfig.current"
        :page-size="pageConfig.pageSize"
        @current-change="handleCurrentChange"></el-pagination>
    </div>
    <dialog-form v-if="dialogVisible" :dialogData="{ formType, dialogVisible, rowData }" @close="closeHandler" @submit="submitHandler"></dialog-form>
  </div>
</template>

<script>
import urlmap from '@/utils/urlmap';
import { dateFormat, debounce } from '@/utils/common';
import dialogForm from './components/dialogForm.vue';
import Collapse from '@/components/collapse.vue';
import qs from 'qs';

export default {
  name: 'melt',
  components: {
    dialogForm, Collapse
  },
  data () {
    return {
      userinfo: {},
      isAddable: false,
      isEditable: false,
      castId: 6,
      searchForm: {
        caster: '',
        furnace: '',
        roller: '',
        date: []
      },
      loading: false,
      tableData: [],
      dialogVisible: false,
      formType: 'add',
      rowData: {},
      pageConfig: {
        total: 1,
        current: 1,
        pageSize: 10
      },
      tableHeight: 200
    }
  },
  // 动态路由匹配
  beforeRouteUpdate(to, from, next) {
    this.castId = to.params.castId;
    this.getTableData();
    // 判断当前用户角色是否有操作权限
    this.isAddable = this.setIsAddable(); 
    this.isEditable = this.setIsEditable(); 
    next();
  },
  created () {
    this.castId = this.$route.params.castId;
    this.userinfo = JSON.parse(localStorage.getItem('userinfo'));
    // 判断当前用户角色是否有操作权限
    this.isAddable = this.setIsAddable(); 
    this.isEditable = this.setIsEditable(); 
    this.getTableData();    
  },
  mounted () {
    const self = this;
    self.$nextTick(() => {
      // self.tableHeight = window.innerHeight - self.$refs.table.$el.getBoundingClientRect().top;
      self.tableHeight = window.innerHeight - 100;
    });
    window.onresize = debounce(() => {
      // self.tableHeight = window.innerHeight - self.$refs.table.$el.getBoundingClientRect().top;
      self.tableHeight = window.innerHeight - 100;
    }, 1000)
  },
  methods: {
    dateFormat(row, column) {
      return dateFormat(row.castDate);
    },
    rollDateFormat(row, column) {
      return dateFormat(row.createdAt);
    },
    clickSearch() {
      // 重置当前页码为1
      const params = {
        current: 1
      };
      this.pageConfig.current = 1;
      this.getTableData(params);
    },
    reset() {
      this.searchForm = { caster: '', furnace: '', roller: '', date: [] };
      const params = {
        current: 1
      };
      this.pageConfig.current = 1;
      this.getTableData(params);
    },
    getTableData(params = {}) {
      const _params = {
        castId: this.castId,
        startTime: this.searchForm.date[0],
        endTime: this.searchForm.date[1],
        caster: this.searchForm.caster,
        furnace: this.searchForm.furnace,
        roller: this.searchForm.roller
      };
      Object.assign(params, _params);
      this.$http('get', urlmap.queryMeasure, params).then(data => {
        this.pageConfig.total = data.count;
        this.pageConfig.pageSize = data.limit;
        this.tableData = data.list;
      }).catch((err) => {
        console.log(err);
      }).finally(() => {
        this.loading = false;
      });
    },
    add() {
      this.dialogVisible = true;
      this.formType = 'add';
    },
    edit(row) {
      this.rowData = row;
      this.dialogVisible = true;
      this.formType = 'edit';
    },
    // del(row) {
    //   const { _id, furnace } = row;
    //   this.$confirm(`确定删除 ${furnace} 吗？`, '提示', {
    //     confirmButtonText: '确定',
    //     cancelButtonText: '取消',
    //     type: 'warning'
    //   }).then(() => {
    //     this.$http('delete', urlmap.delMeasure, {_id}).then(data => {
    //       this.getTableData();
    //     }).catch(error => {
    //       console.log(error);
    //     });
    //   }).catch(() => {});
    // },
    closeHandler() {
      this.dialogVisible = false;
    },
    submitHandler() {
      if (this.formType === 'edit') {
        this.dialogVisible = false;
      }
      this.pageConfig.current = 1;
      this.getTableData();
    },
    handleCurrentChange(val) {
      const params = {
        current: val
      };
      this.getTableData(params);
    },
    setIsAddable() {
      if (this.userinfo.roleId == 4 || this.userinfo.roleId == 15) { // 重卷人员可修改
        return true;
      } else { // 其他
        return false;
      }
    },
    setIsEditable() {
      if (this.userinfo.roleId == 4 || this.userinfo.roleId == 15 || this.userinfo.roleId == 1) { // 重卷人员 或者厂长 可修改
        return true;
      } else { // 其他
        return false;
      }
    },
    exportExcel() {
      const params = {
        castId: this.castId,
        startTime: this.searchForm.date[0],
        endTime: this.searchForm.date[1],
        caster: this.searchForm.caster,
        furnace: this.searchForm.furnace,
        roller: this.searchForm.roller
      };
      const url = `${urlmap.exportRoll}?${qs.stringify(params)}`;
      window.open(url);
    }
  }
}
</script>


