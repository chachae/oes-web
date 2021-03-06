<template>
  <div class="app-container">
    <div class="filter-container">
      <el-upload
        :on-error="handleError"
        list-type="file"
        :limit="1"
        action="http://localhost:8301/system/user/import"
        :headers="headers"
      >
        <el-button plain type="primary">点击上传</el-button>
        <div slot="tip" class="el-upload__tip">图片不能超过1MB</div>
      </el-upload>
    </div>
    <div class="filter-container">
      <el-input v-model="queryParams.username" :placeholder="$t('table.user.username')" class="filter-item search-item" />
      <el-input v-model="queryParams.fullName" placeholder="真实姓名" class="filter-item search-item" />
      <el-input v-model="queryParams.deptName" :placeholder="$t('table.user.dept')" class="filter-item search-item" />
      <el-date-picker
        v-model="queryParams.timeRange"
        :range-separator="null"
        :start-placeholder="$t('table.user.createTime')"
        value-format="yyyy-MM-dd"
        class="filter-item search-item date-range-item"
        type="daterange"
      />
      <el-button class="filter-item" type="primary" @click="search">
        {{ $t('table.search') }}
      </el-button>
      <el-button class="filter-item" type="warning" @click="reset">
        {{ $t('table.reset') }}
      </el-button>
      <el-dropdown v-has-any-permission="['user:add','user:delete','user:reset','user:export']" trigger="click" class="filter-item">
        <el-button>
          {{ $t('table.more') }}<i class="el-icon-arrow-down el-icon--right" />
        </el-button>
        <!-- 工具条 -->
        <el-dropdown-menu slot="dropdown">
          <el-dropdown-item v-has-permission="['user:add']" @click.native="add">{{ $t('table.add') }}</el-dropdown-item>
          <el-dropdown-item v-has-permission="['user:delete']" @click.native="batchDelete">{{ $t('table.delete') }}</el-dropdown-item>
          <el-dropdown-item v-has-permission="['user:reset']" @click.native="resetPassword">{{ $t('table.resetPassword') }}</el-dropdown-item>
          <el-dropdown-item v-has-permission="['user:export']" @click.native="exportExcel">{{ $t('table.export') }}</el-dropdown-item>
        </el-dropdown-menu>
      </el-dropdown>
    </div>
    <!-- 表单 -->
    <el-table
      ref="table"
      :key="tableKey"
      :data="list"
      border
      fit
      style="width: 100%;"
      @selection-change="onSelectChange"
      @sort-change="sortChange"
    >
      <el-table-column type="selection" align="center" width="40px" />
      <el-table-column :label="$t('table.user.username')" prop="username" :show-overflow-tooltip="true" align="center" min-width="120px">
        <template slot-scope="scope">
          <span>{{ scope.row.username }}</span>
        </template>
      </el-table-column>
      <el-table-column label="真实姓名" :show-overflow-tooltip="true" align="center" min-width="100px">
        <template slot-scope="scope">
          <span>{{ scope.row.fullName }}</span>
        </template>
      </el-table-column>
      <el-table-column
        :label="$t('table.user.sex')"
        :filters="[{ text: $t('common.sex.male'), value: '0' }, { text: $t('common.sex.female'), value: '1' }, { text: $t('common.sex.secret'), value: '2' }]"
        :filter-method="filterSex"
        class-name="status-col"
      >
        <template slot-scope="{row}">
          <el-tag :type="row.sex | sexFilter">
            {{ transSex(row.sex) }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column :label="$t('table.user.dept')" align="center" min-width="100px">
        <template slot-scope="scope">
          <span>{{ scope.row.deptName }}</span>
        </template>
      </el-table-column>
      <el-table-column
        :label="$t('table.user.status')"
        :filters="[{ text: $t('common.status.valid'), value: '1' }, { text: $t('common.status.invalid'), value: '0' }]"
        :filter-method="filterStatus"
        class-name="status-col"
      >
        <template slot-scope="{row}">
          <el-tag :type="row.status | statusFilter">
            {{ row.status === '1' ? $t('common.status.valid') : $t('common.status.invalid') }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column :label="$t('table.user.createTime')" prop="createTime" align="center" min-width="180px" sortable="custom">
        <template slot-scope="scope">
          <span>{{ scope.row.createTime }}</span>
        </template>
      </el-table-column>
      <el-table-column :label="$t('table.operation')" align="center" min-width="150px" class-name="small-padding fixed-width">
        <template slot-scope="{row}">
          <i v-hasPermission="['user:view']" class="el-icon-view table-operation" style="color: #87d068;" @click="view(row)" />
          <i v-hasPermission="['user:update']" class="el-icon-setting table-operation" style="color: #2db7f5;" @click="edit(row)" />
          <i v-hasPermission="['user:delete']" class="el-icon-delete table-operation" style="color: #f50;" @click="singleDelete(row)" />
          <el-link v-has-no-permission="['user:view','user:update','user:delete']" class="no-perm">
            {{ $t('tips.noPermission') }}
          </el-link>
        </template>
      </el-table-column>
    </el-table>
    <!-- 分页 -->
    <pagination v-show="total>0" :total="total" :page.sync="pagination.num" :limit.sync="pagination.size" @pagination="search" />
    <!-- 编辑 -->
    <user-edit
      ref="edit"
      :dialog-visible="dialog.isVisible"
      :title="dialog.title"
      @success="editSuccess"
      @close="editClose"
    />
    <!-- 查看 -->
    <user-view
      ref="view"
      :dialog-visible="userViewVisible"
      @close="viewClose"
    />
  </div>
</template>

<script>
import { getToken } from '@/utils/auth'
import Pagination from '@/components/Pagination'
import UserEdit from './Edit'
import UserView from './View'
import UserAPI from '@/api/system/user'

export default {
  name: 'UserManage',
  components: { Pagination, UserEdit, UserView },
  filters: {
    // 性别 tag 转换
    sexFilter(status) {
      const map = {
        0: '',
        1: 'success',
        2: 'info'
      }
      return map[status]
    },
    // 状态 tag 转换
    statusFilter(status) {
      const map = {
        0: 'danger',
        1: 'success'
      }
      return map[status]
    }
  },
  data() {
    return {
      headers: {
        Authorization: `bearer ${getToken()}`
      },
      dialog: {
        isVisible: false,
        title: ''
      },
      userViewVisible: false,
      tableKey: 0,
      loading: false,
      list: null,
      total: 0,
      queryParams: {},
      sort: {},
      selection: [],
      pagination: {
        size: 10,
        num: 1
      }
    }
  },
  computed: {
    currentUser() {
      return this.$store.state.account.user
    }
  },
  mounted() {
    this.fetch()
  },
  methods: {
    handleError(response, file, fileList) {
      this.$message({
        type: 'error',
        message: JSON.parse(response.message).message
      })
    },
    // 转换性别数字表示
    transSex(sex) {
      switch (sex) {
        case '0':
          return this.$t('common.sex.male')
        case '1':
          return this.$t('common.sex.female')
        default:
          return this.$t('common.sex.secret')
      }
    },
    filterStatus(value, row) {
      return row.status === value
    },
    filterSex(value, row) {
      return row.sex === value
    },
    viewClose() {
      this.userViewVisible = false
    },
    editClose() {
      this.dialog.isVisible = false
    },
    editSuccess() {
      this.search()
    },
    onSelectChange(selection) {
      this.selection = selection
    },
    search() {
      this.fetch({
        ...this.queryParams,
        ...this.sort
      })
    },
    reset() {
      this.queryParams = {}
      this.sort = {}
      this.$refs.table.clearSort()
      this.$refs.table.clearFilter()
      this.search()
    },
    // 导出用户数据的 Excel 文档
    exportExcel() {
      this.$download('system/user/excel', {
        pageSize: this.pagination.size,
        pageNum: this.pagination.num,
        ...this.queryParams
      }, `user_${new Date().getTime()}.xlsx`)
    },
    // 打开增加用户模态框
    add() {
      this.dialog.title = this.$t('common.add')
      this.dialog.isVisible = true
    },
    // 用户密码重置
    resetPassword() {
      if (!this.selection.length) {
        this.$message({
          message: this.$t('tips.noDataSelected'),
          type: 'warning'
        })
        return
      }
      this.$confirm(this.$t('tips.confirmRestPassword'), this.$t('common.tips'), {
        confirmButtonText: this.$t('common.confirm'),
        cancelButtonText: this.$t('common.cancel'),
        type: 'warning'
      }).then(() => {
        const userNames = []
        this.selection.forEach((u) => {
          userNames.push(u.username)
        })
        UserAPI.resetPassword(userNames).then(() => {
          this.$message({
            message: this.$t('tips.resetPasswordSuccess'),
            type: 'success'
          })
          this.clearSelections()
        })
      }).catch(() => {
        this.clearSelections()
      })
    },
    // 单个删除
    singleDelete(row) {
      this.$refs.table.toggleRowSelection(row, true)
      this.batchDelete()
    },
    // 批量删除
    batchDelete() {
      if (!this.selection.length) {
        this.$message({
          message: this.$t('tips.noDataSelected'),
          type: 'warning'
        })
        return
      }
      let contain = false
      this.$confirm(this.$t('tips.confirmDelete'), this.$t('common.tips'), {
        confirmButtonText: this.$t('common.confirm'),
        cancelButtonText: this.$t('common.cancel'),
        type: 'warning'
      }).then(() => {
        const userIds = []
        this.selection.forEach((u) => {
          if (u.userId === this.currentUser.userId) {
            contain = true
            return
          }
          userIds.push(u.userId)
        })
        if (contain) {
          this.$message({
            message: this.$t('tips.containCurrentUser'),
            type: 'warning'
          })
          this.clearSelections()
        } else {
          this.delete(userIds)
        }
      }).catch(() => {
        this.clearSelections()
      })
    },
    clearSelections() {
      this.$refs.table.clearSelection()
    },
    view(row) {
      this.$refs.view.setUser(row)
      this.userViewVisible = true
    },
    // 删除用户
    delete(userIds) {
      this.loading = true
      UserAPI.del(userIds).then(() => {
        this.$message({
          message: this.$t('tips.deleteSuccess'),
          type: 'success'
        })
        this.search()
      })
    },
    // 编辑用户
    edit(row) {
      let roleId = []
      if (row.roleId && typeof row.roleId === 'string') {
        roleId = row.roleId.split(',')
        row.roleId = roleId
      }
      UserAPI.get(row.userId).then((r) => {
        row.deptIds = r.data.data
        this.$refs.edit.setUser(row)
        this.dialog.title = this.$t('common.edit')
        this.dialog.isVisible = true
      })
    },
    fetch(params = {}) {
      params.pageSize = this.pagination.size
      params.pageNum = this.pagination.num
      if (this.queryParams.timeRange) {
        params.createTimeFrom = this.queryParams.timeRange[0]
        params.createTimeTo = this.queryParams.timeRange[1]
      }
      this.loading = true
      this.$get('system/user', { ...params }).then((r) => {
        const data = r.data.data
        this.total = data.total
        this.list = data.rows
        this.loading = false
      })
    },
    sortChange(val) {
      this.sort.field = val.prop
      this.sort.order = val.order
      this.search()
    }
  }
}
</script>
<style lang="scss" scoped>
</style>
