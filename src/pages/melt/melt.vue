<template>
  <div>
    <el-breadcrumb separator-class="el-icon-arrow-right" class="crumb">
      <el-breadcrumb-item>冶炼记录</el-breadcrumb-item>
      <el-breadcrumb-item>{{castId}}号机组</el-breadcrumb-item>
    </el-breadcrumb>
    <Collapse>
      <el-form class="search_bar" :model="searchForm" :inline="true">
        <el-form-item label="冶炼日期：">
          <el-date-picker
            v-model="searchForm.date"
            type="daterange"
            :default-time="['00:00:00', '23:59:59']"
            :clearable="false"
            start-placeholder="开始日期"
            end-placeholder="结束日期">
          </el-date-picker>
        </el-form-item>
        <el-form-item label="冶炼人：">
          <el-input v-model="searchForm.melter" placeholder="请输入冶炼人姓名"></el-input>
        </el-form-item>
        <el-form-item label="材质：">
          <el-select v-model="searchForm.ribbonTypeName" placeholder="请选择">
            <el-option v-for="item in ribbonTypeList" :key="item.ribbonTypeId" :value="item.ribbonTypeName" :label="item.ribbonTypeName"></el-option>
          </el-select>
        </el-form-item>
        <el-form-item label="桶号：">
          <el-input v-model="searchForm.bucket" placeholder="请输入桶号"></el-input>
        </el-form-item>
        <el-form-item label="新料：">
          <el-input v-model="searchForm.newAlloyNumber" placeholder="请输入新料炉号"></el-input>
        </el-form-item>
        <el-form-item label="加工料：">
          <el-input v-model="searchForm.oldAlloyNumber" placeholder="请输入加工料炉号"></el-input>
        </el-form-item>
        <el-form-item>
          <el-button type="primary" icon="el-icon-search" @click="clickSearch">搜索</el-button>
          <el-button type="primary" icon="el-icon-refresh" @click="reset">重置</el-button>
        </el-form-item>
      </el-form>
    </Collapse>
    <div class="main_bd">
      <el-col class="table_hd">
        <el-button type="primary" icon="el-icon-plus" @click="createMelt" v-if="isAble">创建冶炼记录</el-button>
        <el-button type="primary" icon="el-icon-download" @click="exportExcel" v-if="userinfo.roleId === 1 || userinfo.roleId === 2 || userinfo.roleId === 3" class="pull_right">导出</el-button>
      </el-col>
      <el-table :data="tableData" stripe border style="width:100%" v-loading="loading" ref="table" :height="tableHeight">
        <el-table-column prop="createTime" label="冶炼日期" align="center" width="110px" :formatter="dateFormat" fixed></el-table-column>
        <el-table-column prop="ribbonTypeName" label="材质" align="center" width="80px" fixed></el-table-column>
        <el-table-column prop="furnace" label="炉号" align="center" width="170px" fixed></el-table-column>
        <el-table-column prop="bucket" label="桶号" align="center" width="50px"></el-table-column>
        <el-table-column prop="melter" label="冶炼人" align="center" width="70px"></el-table-column>
        <el-table-column prop="meltFurnace" label="冶炼炉" align="center" width="80px"></el-table-column>
        <el-table-column prop="newAlloyNumber" label="新料炉号" align="center" width="150px"></el-table-column>
        <el-table-column prop="newAlloyWeight" label="新料重量(kg)" align="center" width="110px"></el-table-column>
        <el-table-column prop="oldAlloyNumber" label="加工料炉号" align="center" width="150px"></el-table-column>
        <el-table-column prop="oldAlloyWeight" label="加工料重量(kg)" align="center" width="120px"></el-table-column>
        <el-table-column prop="mixAlloyNumber" label="回炉锭炉号" align="center" width="150px"></el-table-column>
        <el-table-column prop="mixAlloyWeight" label="回炉锭重量(kg)" align="center" width="120px"></el-table-column>
        <el-table-column prop="highNbNumber" label="高铌料炉号" align="center" width="150px"></el-table-column>
        <el-table-column prop="highNbWeight" label="高铌料重量(kg)" align="center" width="120px"></el-table-column>
        <el-table-column prop="Si" label="硅(g)" align="center" width="60px"></el-table-column>
        <el-table-column prop="Ni" label="镍(g)" align="center" width="60px"></el-table-column>
        <el-table-column prop="Cu" label="铜(g)" align="center" width="60px"></el-table-column>
        <el-table-column prop="BFe" label="硼铁(g)" align="center" width="70px"></el-table-column>
        <el-table-column prop="NbFe" label="铌铁(g)" align="center" width="70px"></el-table-column>
        <el-table-column prop="alloyTotalWeight" label="总重量(kg)" align="center" width="100px"></el-table-column>
        <el-table-column prop="alloyOutWeight" label="放钢重量(kg)" align="center" width="110px"></el-table-column>
        <el-table-column prop="alloyFixWeight" label="修正重量(kg)" align="center" width="110px"></el-table-column>
        <el-table-column prop="remark" label="备注" align="center" width="100px" :show-overflow-tooltip="true"></el-table-column>
        <el-table-column prop="updatedAt" label="更新时间" align="center" width="170px" :formatter="dateTimeFormat"></el-table-column>
        <el-table-column prop="updatePerson" label="更新者" align="center" width="70px"></el-table-column>
        <el-table-column label="操作" align="center" width="150px">
          <template slot-scope="scope">
            <el-button size="mini" type="primary" @click="edit(scope.row)" :disabled="userinfo.roleId !== 1 && userinfo.roleId !== 3 && userinfo.adminname !== scope.row.melter">修改</el-button>
            <el-button size="mini" type="danger" @click="del(scope.row)" v-if="userinfo.roleId === 1 || userinfo.roleId === 3">删除</el-button>
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
    <dialog-form v-if="dialogVisible" :dialogData="{ formType, dialogVisible, rowData, ribbonTypeList }" @close="closeHandler" @submit="submitHandler"></dialog-form>
  </div>
</template>

<script>
import urlmap from '@/utils/urlmap';
import { dateFormat, dateTimeFormat, debounce } from '@/utils/common';
import dialogForm from './components/dialogForm.vue';
import Collapse from '@/components/collapse.vue';
import { mapState, mapActions } from 'vuex';
import qs from 'qs';

export default {
  name: 'melt',
  components: {
    dialogForm, Collapse
  },
  data () {
    return {
      userinfo: {},
      isAble: false,
      castId: 6,
      searchForm: {
        melter: '',
        ribbonTypeName: '',
        bucket: '',
        newAlloyNumber: '',
        oldAlloyNumber: '',
        date: []
      },
      loading: false,
      tableData: [],
      dialogVisible: false,
      formType: 'create',
      rowData: {},
      pageConfig: {
        total: 1,
        current: 1,
        pageSize: 10
      },
      tableHeight: 100
    }
  },
  computed: {
    ...mapState([
      'ribbonTypeList'
    ])
  },
  // 动态路由匹配
  beforeRouteUpdate(to, from, next) {
    this.castId = to.params.castId;
    this.getTableData();
    // 判断当前用户角色是否有操作权限
    this.isAble = this.setIsAble();    
    next();
  },
  created () {
    this.castId = this.$route.params.castId;
    this.userinfo = JSON.parse(localStorage.getItem('userinfo'));
    // 判断当前用户角色是否有操作权限
    this.isAble = this.setIsAble();
    this.getTableData();
    this.getRibbonTypeList();  
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
    ...mapActions([
      'getRibbonTypeList'
    ]),
    dateFormat(row, column) {
      return dateFormat(row.createTime);
    },
    dateTimeFormat(row, column) {
      return dateTimeFormat(row.updatedAt);
    },
    setIsAble() {
      // 8：6#机组冶炼人员，10：7#机组冶炼人员，12：8#机组冶炼人员，14：9#机组冶炼人员
      if (this.userinfo.roleId == 8 || this.userinfo.roleId == 10 || this.userinfo.roleId == 12 || this.userinfo.roleId == 14) {
        return true;
      } else { // 其他
        return false;
      }
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
      this.searchForm = { melter: '', ribbonTypeName: '', bucket: '', newAlloyNumber: '', oldAlloyNumber: '', date: [] };
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
        melter: this.searchForm.melter,
        ribbonTypeName: this.searchForm.ribbonTypeName,
        bucket: this.searchForm.bucket,
        newAlloyNumber: this.searchForm.newAlloyNumber,
        oldAlloyNumber: this.searchForm.oldAlloyNumber
      };
      Object.assign(params, _params);
      this.$http('get', urlmap.queryMelt, params).then(data => {
        this.pageConfig.total = data.count;
        this.pageConfig.pageSize = data.limit;
        this.tableData = data.list;
      }).catch((err) => {
        console.log(err);
      }).finally(() => {
        this.loading = false;
      });
    },
    createMelt() {
      this.dialogVisible = true;
      this.formType = 'create';
    },
    edit(row) {
      this.rowData = row;
      this.dialogVisible = true;
      this.formType = 'edit';
    },
    del(row) {
      const { meltId, furnace } = row;
      this.$confirm(`删除后数据无法恢复，确定删除 ${furnace} 吗？`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http('delete', urlmap.delMelt, {meltId}).then(data => {
          this.getTableData();
        }).catch(error => {
          console.log(error);
        });
      }).catch(() => {});
    },
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
    exportExcel() {
      const params = {
        castId: this.castId,
        startTime: this.searchForm.date[0],
        endTime: this.searchForm.date[1],
        melter: this.searchForm.melter,
        ribbonTypeName: this.searchForm.ribbonTypeName,
        bucket: this.searchForm.bucket,
        newAlloyNumber: this.searchForm.newAlloyNumber,
        oldAlloyNumber: this.searchForm.oldAlloyNumber
      };
      const url = `${urlmap.exportMelt}?${qs.stringify(params)}`;
      window.open(url);
    }
  }
}
</script>

<style lang="scss" scoped>

</style>

