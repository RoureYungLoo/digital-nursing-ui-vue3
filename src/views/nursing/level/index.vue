<template>
  <div class="app-container">
    <el-form :model="queryParams" ref="queryRef" :inline="true" v-show="showSearch" label-width="68px">
      <el-form-item label="等级名称">
        <el-input
            v-model="queryParams.name"
            placeholder="请输入护理等级名称"
            clearable
            @keyup.enter="handleQuery"/>
      </el-form-item>
      <el-form-item label="等级状态">
        <el-input
            v-model="queryParams.status"
            placeholder="请选择护理等级状态"
            clearable
            @keyup.enter="handleQuery"/>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="Search" @click="handleQuery">搜索</el-button>
        <el-button icon="Refresh" @click="resetQuery">重置</el-button>
      </el-form-item>
    </el-form>

    <el-row :gutter="10" class="mb8">
      <el-col :span="1.5">
        <el-button
            type="primary"
            plain
            icon="Plus"
            @click="handleAdd"
            v-hasPermi="['nursing:level:add']"
        >新增
        </el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button
            type="success"
            plain
            icon="Edit"
            :disabled="single"
            @click="handleUpdate"
            v-hasPermi="['nursing:level:edit']"
        >修改
        </el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button
            type="danger"
            plain
            icon="Delete"
            :disabled="multiple"
            @click="handleDelete"
            v-hasPermi="['nursing:level:remove']"
        >删除
        </el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button
            type="warning"
            plain
            icon="Download"
            @click="handleExport"
            v-hasPermi="['nursing:level:export']"
        >导出
        </el-button>
      </el-col>
      <right-toolbar v-model:showSearch="showSearch" @queryTable="getList"></right-toolbar>
    </el-row>

    <el-table v-loading="loading" :data="levelList" @selection-change="handleSelectionChange">
      <el-table-column type="selection" width="55" align="center"/>
      <el-table-column label="主键ID" align="center" prop="id"/>
      <el-table-column label="等级名称" align="center" prop="name"/>
      <el-table-column label="护理计划ID" align="center" prop="lplanId"/>
      <el-table-column label="护理费用" align="center" prop="fee"/>
      <el-table-column label="状态" align="center" prop="status"/>
      <el-table-column label="等级说明" align="center" prop="description"/>
      <el-table-column label="创建时间" align="center" prop="createTime" width="180">
        <template #default="scope">
          <span>{{ parseTime(scope.row.createTime, '{y}-{m}-{d}') }}</span>
        </template>
      </el-table-column>
      <el-table-column label="操作" align="center" class-name="small-padding fixed-width">
        <template #default="scope">
          <el-button link type="primary" icon="Edit" @click="handleUpdate(scope.row)"
                     v-hasPermi="['nursing:level:edit']">修改
          </el-button>
          <el-button link type="primary" icon="Delete" @click="handleDelete(scope.row)"
                     v-hasPermi="['nursing:level:remove']">删除
          </el-button>
        </template>
      </el-table-column>
    </el-table>

    <pagination
        v-show="total>0"
        :total="total"
        v-model:page="queryParams.pageNum"
        v-model:limit="queryParams.pageSize"
        @pagination="getList"
    />

    <!-- 添加或修改护理等级对话框 -->
    <el-dialog :title="title" v-model="open" width="500px" append-to-body>
      <el-form ref="levelRef" :model="form" :rules="rules" label-width="80px">
        <el-form-item label="等级名称" prop="name">
          <el-input v-model="form.name" placeholder="请输入等级名称"/>
        </el-form-item>
        <el-form-item label="护理计划" prop="name">
          <el-select v-model="form.lplanId" placeholder="请选择护理计划">
            <el-option label="护理计划1" :value="1"/>
            <el-option label="护理计划2" :value="2"/>
            <el-option label="护理计划3" :value="3"/>
          </el-select>
        </el-form-item>
        <el-form-item label="护理费用" prop="fee">
          <el-input v-model.number="form.fee" placeholder="请设置护理费用"/>
        </el-form-item>
        <el-form-item label="状态" prop="fee">
          <el-radio-group v-model="form.status" placeholder="请输入护理费用">
            <el-radio label="启用" :value="1"/>
            <el-radio label="禁用" :value="0"/>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="等级说明" prop="description">
          <el-input type="textarea" v-model="form.description" placeholder="请输入等级说明"/>
        </el-form-item>
      </el-form>
      <template #footer>
        <div class="dialog-footer">
          <el-button type="primary" @click="submitForm">确 定</el-button>
          <el-button @click="cancel">取 消</el-button>
        </div>
      </template>
    </el-dialog>
  </div>
</template>

<script setup name="Level">
import {listLevel, getLevel, delLevel, addLevel, updateLevel} from "@/api/nursing/level"

const {proxy} = getCurrentInstance()

const levelList = ref([])
const open = ref(false)
const loading = ref(true)
const showSearch = ref(true)
const ids = ref([])
const single = ref(true)
const multiple = ref(true)
const total = ref(0)
const title = ref("")

const data = reactive({
  form: {},
  queryParams: {
    pageNum: 1,
    pageSize: 10,
    name: null,
    status: null,
  },
  rules: {
    name: [
      {required: true, message: "等级名称不能为空", trigger: "blur"}
    ],
    lplanId: [
      {required: true, message: "护理计划ID不能为空", trigger: "change"}
    ],
    fee: [
      {required: true, message: "护理费用不能为空", trigger: "blur"}
    ],
    status: [
      {required: true, message: "状态不能为空", trigger: "change"}
    ],
    description: [
      {required: true, message: "等级说明不能为空", trigger: "blur"}
    ],
  }
})

const {queryParams, form, rules} = toRefs(data)

/** 查询护理等级列表 */
function getList() {
  loading.value = true
  listLevel(queryParams.value).then(response => {
    levelList.value = response.rows
    total.value = response.total
    loading.value = false
  })
}

// 取消按钮
function cancel() {
  open.value = false
  reset()
}

// 表单重置
function reset() {
  form.value = {
    id: null,
    name: null,
    planId: null,
    fee: null,
    status: null,
    description: null,
    createTime: null,
    createBy: null,
    updateBy: null,
    remark: null,
    updateTime: null
  }
  proxy.resetForm("levelRef")
}

/** 搜索按钮操作 */
function handleQuery() {
  queryParams.value.pageNum = 1
  getList()
}

/** 重置按钮操作 */
function resetQuery() {
  proxy.resetForm("queryRef")
  handleQuery()
}

// 多选框选中数据
function handleSelectionChange(selection) {
  ids.value = selection.map(item => item.id)
  single.value = selection.length != 1
  multiple.value = !selection.length
}

/** 新增按钮操作 */
function handleAdd() {
  reset()
  open.value = true
  title.value = "添加护理等级"
}

/** 修改按钮操作 */
function handleUpdate(row) {
  reset()
  const _id = row.id || ids.value
  getLevel(_id).then(response => {
    form.value = response.data
    open.value = true
    title.value = "修改护理等级"
  })
}

/** 提交按钮 */
function submitForm() {
  proxy.$refs["levelRef"].validate(valid => {
    if (valid) {
      if (form.value.id != null) {
        updateLevel(form.value).then(response => {
          proxy.$modal.msgSuccess("修改成功")
          open.value = false
          getList()
        })
      } else {
        addLevel(form.value).then(response => {
          proxy.$modal.msgSuccess("新增成功")
          open.value = false
          getList()
        })
      }
    }
  })
}

/** 删除按钮操作 */
function handleDelete(row) {
  const _ids = row.id || ids.value
  proxy.$modal.confirm('是否确认删除护理等级编号为"' + _ids + '"的数据项？').then(function () {
    return delLevel(_ids)
  }).then(() => {
    getList()
    proxy.$modal.msgSuccess("删除成功")
  }).catch(() => {
  })
}

/** 导出按钮操作 */
function handleExport() {
  proxy.download('nursing/level/export', {
    ...queryParams.value
  }, `level_${new Date().getTime()}.xlsx`)
}

getList()
</script>
