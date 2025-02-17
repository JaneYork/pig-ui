<!--
  -    Copyright (c) 2018-2025, lengleng All rights reserved.
  -
  - Redistribution and use in source and binary forms, with or without
  - modification, are permitted provided that the following conditions are met:
  -
  - Redistributions of source code must retain the above copyright notice,
  - this list of conditions and the following disclaimer.
  - Redistributions in binary form must reproduce the above copyright
  - notice, this list of conditions and the following disclaimer in the
  - documentation and/or other materials provided with the distribution.
  - Neither the name of the pig4cloud.com developer nor the names of its
  - contributors may be used to endorse or promote products derived from
  - this software without specific prior written permission.
  - Author: lengleng (wangiegie@gmail.com)
  -->

<template>
  <div class="user">
    <basic-container>
      <avue-crud
        :option="option"
        ref="crud"
        v-model="form"
        :page.sync="page"
        @on-load="getList"
        @size-change="sizeChange"
        @current-change="currentChange"
        :table-loading="listLoading"
        @search-change="handleFilter"
        @refresh-change="handleRefreshChange"
        @row-update="update"
        @row-save="create"
        :before-open="handleOpenBefore"
        :data="list"
      >
        <template slot="menuLeft">
          <el-button
            v-if="permissions.sys_user_add"
            class="filter-item"
            @click="$refs.crud.rowAdd()"
            type="primary"
            icon="el-icon-edit"
            >添加
          </el-button>
          <el-button
            v-if="permissions.sys_user_import_export"
            class="filter-item"
            plain
            type="primary"
            size="small"
            icon="el-icon-upload"
            @click="$refs.excelUpload.show()"
            >导入
          </el-button>
          <el-button
            v-if="permissions.sys_user_import_export"
            class="filter-item"
            plain
            type="primary"
            size="small"
            icon="el-icon-download"
            @click="exportExcel"
            >导出
          </el-button>
        </template>
        <template slot="username" slot-scope="scope">
          <span>{{ scope.row.username }}</span>
        </template>
        <template slot="role" slot-scope="scope">
          <span v-for="(role, index) in scope.row.roleList" :key="index">
            <el-tag>{{ role.roleName }} </el-tag>&nbsp;&nbsp;
          </span>
        </template>
        <template slot="deptId" slot-scope="scope">
          {{ scope.row.deptName }}
        </template>
        <template slot="lockFlag" slot-scope="scope">
          <el-tag>{{ scope.label }}</el-tag>
        </template>
        <template slot="menu" slot-scope="scope">
          <el-button
            v-if="permissions.sys_user_edit"
            type="text"
            icon="el-icon-edit"
            @click="handleUpdate(scope.row, scope.index)"
            >编辑
          </el-button>
          <el-button
            v-if="permissions.sys_user_del"
            type="text"
            icon="el-icon-delete"
            @click="deletes(scope.row, scope.index)"
            >删除
          </el-button>
        </template>
        <template slot="deptIdForm" slot-scope="scope">
          <avue-input-tree
            v-model="form.deptId"
            placeholder="请选择所属部门"
            :node-click="getNodeData"
            :dic="treeDeptData"
            :props="defaultProps"
          ></avue-input-tree>
        </template>
        <template slot="roleForm" slot-scope="scope">
          <avue-select
            v-model="role"
            multiple
            placeholder="请选择角色"
            :dic="rolesOptions"
            :props="roleProps"
          ></avue-select>
        </template>
      </avue-crud>

      <!--excel 模板导入 -->
      <excel-upload
        ref="excelUpload"
        title="用户信息导入"
        url="/admin/user/import"
        temp-name="用户信息.xlsx"
        temp-url="/admin/sys-file/local/user.xlsx"
        @refreshDataList="handleRefreshChange"
      ></excel-upload>
    </basic-container>
  </div>
</template>

<script>
import { addObj, delObj, fetchList, putObj } from "@/api/admin/user";
import { deptRoleList } from "@/api/admin/role";
import { fetchTree } from "@/api/admin/dept";
import { tableOption } from "@/const/crud/admin/user";
import { mapGetters } from "vuex";
import ExcelUpload from "@/components/upload/excel";

export default {
  name: "table_user",
  components: { ExcelUpload },
  data() {
    return {
      option: tableOption,
      treeDeptData: [],
      checkedKeys: [],
      roleProps: {
        label: "roleName",
        value: "roleId"
      },
      defaultProps: {
        label: "name",
        value: "id"
      },
      page: {
        total: 0, // 总页数
        currentPage: 1, // 当前页数
        pageSize: 20, // 每页显示多少条,
        isAsc: false //是否倒序
      },
      query: {},
      list: [],
      listLoading: true,
      role: [],
      form: {},
      rolesOptions: []
    };
  },
  computed: {
    ...mapGetters(["permissions"])
  },
  watch: {
    role() {
      this.form.role = this.role;
    }
  },
  methods: {
    getList(page, params) {
      this.listLoading = true;
      fetchList(
        Object.assign(
          {
            current: page.currentPage,
            size: page.pageSize
          },
          params
        )
      ).then(response => {
        this.list = response.data.data.records;
        this.page.total = response.data.data.total;
        this.listLoading = false;
      });
    },
    getNodeData(data) {
      deptRoleList().then(response => {
        this.rolesOptions = response.data.data;
      });
    },
    sizeChange(pageSize) {
      this.page.pageSize = pageSize;
    },
    currentChange(current) {
      this.page.currentPage = current;
    },
    handleFilter(param, done) {
      this.query = param;
      this.page.currentPage = 1;
      this.getList(this.page, param);
      done();
    },
    handleRefreshChange() {
      this.getList(this.page);
    },
    handleOpenBefore(show, type) {
      window.boxType = type;
      // 查询部门树
      fetchTree().then(response => {
        this.treeDeptData = response.data.data;
      });
      // 查询角色列表
      deptRoleList().then(response => {
        this.rolesOptions = response.data.data;
      });
      // 若是编辑、查看回显角色名称
      if (["edit", "views"].includes(type)) {
        this.role = [];
        for (let i = 0; i < this.form.roleList.length; i++) {
          this.role[i] = this.form.roleList[i].roleId;
        }
      } else if (type === "add") {
        // 若是添加角色列表设置为空
        this.role = [];
      }
      show();
    },
    handleUpdate(row, index) {
      this.$refs.crud.rowEdit(row, index);
      this.form.password = undefined;
    },
    create(row, done, loading) {
      addObj(this.form)
        .then(() => {
          this.getList(this.page);
          done();
          this.$notify.success("创建成功");
        })
        .catch(() => {
          loading();
        });
    },
    update(row, index, done, loading) {
      putObj(this.form)
        .then(() => {
          this.getList(this.page);
          done();
          this.$notify.success("修改成功");
        })
        .catch(() => {
          loading();
        });
    },
    deletes(row) {
      this.$confirm(
        "此操作将永久删除该用户(用户名:" + row.username + "), 是否继续?",
        "提示",
        {
          confirmButtonText: "确定",
          cancelButtonText: "取消",
          type: "warning"
        }
      ).then(() => {
        delObj(row.userId)
          .then(() => {
            this.getList(this.page);
            this.$notify.success("删除成功");
          })
          .catch(() => {
            this.$notify.error("删除失败");
          });
      });
    },
    exportExcel() {
      this.downBlobFile("/admin/user/export", this.query, "user.xlsx");
    }
  }
};
</script>
