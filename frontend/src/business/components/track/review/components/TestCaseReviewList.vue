<template>
  <el-card class="table-card" v-loading="result.loading">
    <template v-slot:header>
      <ms-table-header :is-tester-permission="true" :condition.sync="condition"
                       @search="initTableData" @create="testCaseReviewCreate"
                       :create-tip="$t('test_track.review.create_review')"
                       :title="$t('test_track.review.test_review')"/>
    </template>

    <el-table
      border
      class="adjust-table"
      :data="tableData"
      @filter-change="filter"
      @sort-change="sort"
      @row-click="intoReview">
      <template v-for="(item, index) in tableLabel">
        <el-table-column
          v-if="item.id=='name'"
          prop="name"
          :label="$t('test_track.review.review_name')"
          show-overflow-tooltip
          :key="index">
        </el-table-column>
        <el-table-column
          v-if="item.id=='reviewer'"
          prop="reviewer"
          :label="$t('test_track.review.reviewer')"
          show-overflow-tooltip
          :key="index">
        </el-table-column>
        <el-table-column
          v-if="item.id=='projectName'"
          prop="projectName"
          :label="$t('test_track.review.review_project')"
          show-overflow-tooltip
          :key="index">
        </el-table-column>
        <el-table-column
          v-if="item.id=='creatorName'"
          prop="creatorName"
          :label="$t('test_track.review.review_creator')"
          show-overflow-tooltip
          :key="index"
        >
        </el-table-column>
        <el-table-column
          v-if="item.id=='status'"
          prop="status"
          column-key="status"
          :label="$t('test_track.review.review_status')"
          show-overflow-tooltip
          :key="index"
        >
          <template v-slot:default="scope">
          <span class="el-dropdown-link">
            <plan-status-table-item :value="scope.row.status"/>
          </span>
          </template>
        </el-table-column>
        <el-table-column
          v-if="item.id=='createTime'"
          prop="createTime"
          :label="$t('commons.create_time')"
          show-overflow-tooltip
          :key="index"
        >
          <template v-slot:default="scope">
            <span>{{ scope.row.createTime | timestampFormatDate }}</span>
          </template>
        </el-table-column>
        <el-table-column
          v-if="item.id=='endTime'"
          prop="endTime"
          :label="$t('test_track.review.end_time')"
          show-overflow-tooltip
          :key="index">
          <template v-slot:default="scope">
            <span>{{ scope.row.endTime | timestampFormatDate }}</span>
          </template>
        </el-table-column>
      </template>
      <el-table-column
        min-width="100"
        :label="$t('commons.operating')">
        <template slot="header">
          <header-label-operate @exec="customHeader"/>
        </template>
        <template v-slot:default="scope">
          <ms-table-operator :is-tester-permission="true" @editClick="handleEdit(scope.row)"
                             @deleteClick="handleDelete(scope.row)">
          </ms-table-operator>
        </template>
      </el-table-column>
      <header-custom ref="headerCustom" :initTableData="initTableData" :optionalFields=headerItems
                     :type=type></header-custom>
    </el-table>

    <ms-table-pagination :change="initTableData" :current-page.sync="currentPage" :page-size.sync="pageSize"
                         :total="total"/>
    <ms-delete-confirm :title="$t('test_track.review.delete')" @delete="_handleDelete" ref="deleteConfirm"/>

  </el-card>
</template>

<script>
import MsDeleteConfirm from "../../../common/components/MsDeleteConfirm";
import MsTableOperator from "../../../common/components/MsTableOperator";
import MsTableOperatorButton from "../../../common/components/MsTableOperatorButton";
import MsDialogFooter from "../../../common/components/MsDialogFooter";
import MsTableHeader from "../../../common/components/MsTableHeader";
import MsCreateBox from "../../../settings/CreateBox";
import MsTablePagination from "../../../common/pagination/TablePagination";
import {
  checkoutTestManagerOrTestUser,
  getCurrentProjectID, getCurrentUser,
  getCurrentWorkspaceId
} from "../../../../../common/js/utils";
import {_filter, _sort, getLabel} from "@/common/js/tableUtils";
import PlanStatusTableItem from "../../common/tableItems/plan/PlanStatusTableItem";
import {Test_Case_Review} from "@/business/components/common/model/JsonData";
import {TEST_CASE_LIST, TEST_CASE_REVIEW_LIST} from "@/common/js/constants";
import HeaderCustom from "@/business/components/common/head/HeaderCustom";
import HeaderLabelOperate from "@/business/components/common/head/HeaderLabelOperate";

export default {
  name: "TestCaseReviewList",
  components: {
    HeaderLabelOperate,
    HeaderCustom,
    MsDeleteConfirm,
    MsTableOperator,
    MsTableOperatorButton,
    MsDialogFooter,
    MsTableHeader,
    MsCreateBox,
    MsTablePagination,
    PlanStatusTableItem
  },
  data() {
    return {
      type: TEST_CASE_REVIEW_LIST,
      headerItems: Test_Case_Review,
      tableLabel: Test_Case_Review,
      result: {},
      condition: {},
      tableData: [],
      isTestManagerOrTestUser: false,
      currentPage: 1,
      pageSize: 10,
      total: 0,
      statusFilters: [
        {text: this.$t('test_track.plan.plan_status_prepare'), value: 'Prepare'},
        {text: this.$t('test_track.plan.plan_status_running'), value: 'Underway'},
        {text: this.$t('test_track.plan.plan_status_completed'), value: 'Completed'}
      ],
    }
  },
  watch: {
    '$route'(to) {
      if (to.path.indexOf("/track/review/all") >= 0) {
        this.initTableData();
      }
    }
  },
  created() {
    this.isTestManagerOrTestUser = checkoutTestManagerOrTestUser();
    this.initTableData();
  },
  methods: {
    customHeader() {
      this.$refs.headerCustom.open(this.tableLabel)
    },

    initTableData() {
      getLabel(this, TEST_CASE_REVIEW_LIST);
      let lastWorkspaceId = getCurrentWorkspaceId();
      this.condition.workspaceId = lastWorkspaceId;
      if (!getCurrentProjectID()) {
        return;
      }
      this.condition.projectId = getCurrentProjectID();
      this.result = this.$post("/test/case/review/list/" + this.currentPage + "/" + this.pageSize, this.condition, response => {
        let data = response.data;
        this.total = data.itemCount;
        this.tableData = data.listObject;
        for (let i = 0; i < this.tableData.length; i++) {
          let path = "/test/case/review/project";
          this.$post(path, {id: this.tableData[i].id}, res => {
            let arr = res.data;
            let projectIds = arr.filter(d => d.id !== this.tableData[i].projectId).map(data => data.id);
            this.$set(this.tableData[i], "projectIds", projectIds);
          });
          this.$post('/test/case/review/reviewer', {id: this.tableData[i].id}, res => {
            let arr = res.data;
            let reviewer = arr.map(data => data.name).join("、");
            let userIds = arr.map(data => data.id);
            this.$set(this.tableData[i], "reviewer", reviewer);
            this.$set(this.tableData[i], "userIds", userIds);
          })
        }
      });
    },
    intoReview(row) {
      this.$router.push('/track/review/view/' + row.id);
    },
    testCaseReviewCreate() {
      if (!getCurrentProjectID()) {
        this.$warning(this.$t('commons.check_project_tip'));
        return;
      }
      this.$emit('openCaseReviewEditDialog');
    },
    handleEdit(caseReview) {
      this.$emit('caseReviewEdit', caseReview);
    },
    statusChange() {

    },
    handleDelete(caseReview) {
      this.$refs.deleteConfirm.open(caseReview);
    },
    _handleDelete(caseReview) {
      let reviewId = caseReview.id;
      this.$get('/test/case/review/delete/' + reviewId, () => {
        this.initTableData();
        this.$success(this.$t('commons.delete_success'));
      });
    },
    filter(filters) {
      _filter(filters, this.condition);
      this.initTableData();
    },
    sort(column) {
      _sort(column, this.condition);
      this.initTableData();
    },
  }
}
</script>

<style scoped>

</style>
