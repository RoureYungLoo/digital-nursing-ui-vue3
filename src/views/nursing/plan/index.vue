<template>
  <div class="app-container">
    <el-form
        :model="queryParams"
        ref="queryRef"
        :inline="true"
        v-show="showSearch"
        label-width="68px"
    >
      <el-form-item label="名称" prop="planName" style="width: 250px">
        <el-input
            v-model="queryParams.planName"
            placeholder="请输入名称"
            clearable
            @keyup.enter="handleQuery"
        />
      </el-form-item>
      <el-form-item label="状态" prop="status" style="width: 250px">
        <el-select
            v-model="queryParams.status"
            placeholder="请选择状态"
            clearable
        >
          <el-option
              v-for="dict in nursing_plan_status"
              :key="dict.value"
              :label="dict.label"
              :value="parseInt(dict.value)"
          />
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="Search" @click="handleQuery"
        >搜索
        </el-button
        >
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
        >新增
        </el-button
        >
      </el-col>
    </el-row>

    <!--  数据表格  -->
    <el-table
        v-loading="loading"
        :data="planList"
        @selection-change="handleSelectionChange"
    >
      <el-table-column label="序号" type="index" width="50"/>
      <el-table-column label="名称" align="center" prop="planName"/>
      <el-table-column label="状态" align="center" prop="status">
        <template #default="scope">
          <el-tag :type="scope.row.status === 1 ? 'success' : 'danger'">
            {{ scope.row.status === 1 ? '启用' : '禁用' }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column
          label="创建时间"
          align="center"
          prop="createTime"
          width="180"
      >
        <template #default="scope">
          <span>{{
              parseTime(scope.row.createTime, '{y}-{m}-{d} {h}:{i}:{s}')
            }}</span>
        </template>
      </el-table-column>
      <el-table-column
          label="操作"
          align="center"
          fixed="right"
          width="280"
          class-name="small-padding fixed-width"
      >
        <template #default="scope">
          <el-button
              link
              type="primary"
              icon="Edit"
              :class="scope.row.count ? 'disabled' : ''"
              @click="handleUpdate(scope.row)"
          >修改
          </el-button
          >
          <el-button
              link
              type="primary"
              icon="Delete"
              :class="scope.row.count ? 'disabled' : ''"
              @click="handleDelete(scope.row)"
          >删除
          </el-button
          >
          <el-button
              link
              type="primary"
              icon="View"
              @click="handleLook(scope.row)"
          >查看
          </el-button
          >
          <el-button
              link
              type="primary"
              :icon="scope.row.status === 0 ? 'Unlock' : 'lock'"
              @click="handleEnable(scope.row)"
          >{{ scope.row.status === 1 ? '禁用' : '启用' }}
          </el-button
          >
        </template>
      </el-table-column>
    </el-table>

    <pagination
        v-show="total > 0"
        :total="total"
        v-model:page="queryParams.pageNum"
        v-model:limit="queryParams.pageSize"
        @pagination="getNursingPalnList"
    />

    <!-- 新增护理计划   -->
    <el-dialog
        :title="title"
        v-model="dialogVisible"
        width="840"
        @close="cancel()"
    >
      <el-form
          ref="planRef"
          :model="formData"
          :rules="rules"
          label-width="120px"
      >
        <el-row>
          <el-col :span="24">
            <el-form-item label="护理计划名称：" prop="planName">
              <el-input
                  v-model="formData.planName"
                  :disabled="isLook"
              ></el-input>
            </el-form-item>
            <el-form-item label="状态：" prop="status">
              <el-radio-group v-model="formData.status" :disabled="isLook">
                <el-radio
                    v-for="dict in nursing_plan_status"
                    :value="Number(dict.value)"
                    :label="dict.label"
                    :key="dict.value"
                />
              </el-radio-group>
            </el-form-item>
            <el-form-item label="排序：" prop="sortNo">
              <el-input-number placeholder="排序号"
                               :disabled="isLook"
                               v-model="formData.sortNo"
                               :min="0"
                               large-number
                               :max="999999"
                               :decimal-places="0"
                               @blur="textBlurNo"
                               @change="textBlurNo"
              ></el-input-number>
            </el-form-item>
            <el-form-item label="护理项目：">
              <el-col :span="24">
                <el-row :gutter="10">
                  <el-col :span="5" style="text-align: center">护理项目名称</el-col>
                  <el-col :span="5" style="text-align: center">期望服务时间</el-col>
                  <el-col :span="5" style="text-align: center">执行间隔</el-col>
                  <el-col :span="5" style="text-align: center">执行频次</el-col>
                  <el-col :span="4" style="text-align: center">操作</el-col>
                </el-row>
                <template v-for="(nip,index) in formData.itemPlanVoList">
                  <el-row :gutter="10" style="margin-bottom: 5px">
                    <el-col :span="5" style="text-align: center">
                      <el-form-item>
                        <el-select v-model="nip.itemId">
                          <el-option v-for="{label,value} in nursingItemOptions" :key="value" :label="label"
                                     :value="value"/>
                        </el-select>
                      </el-form-item>
                    </el-col>
                    <el-col :span="5" style="text-align: center">
                      <el-form-item>
                        <el-time-picker v-model="nip.executeTime" value-format="HH:mm:ss" placeholder="服务时间"/>
                      </el-form-item>
                    </el-col>
                    <el-col :span="5" style="text-align: center">
                      <el-form-item>
                        <el-select v-model="nip.executeCycle">
                          <el-option v-for="{label,value} in executeCycleOptions" :key="value" :label="label"
                                     :value="value"/>
                        </el-select>
                      </el-form-item>
                    </el-col>
                    <el-col :span="5" style="text-align: center">
                      <el-form-item>
                        <el-input-number v-model="nip.executeFrequency" placeholder="请选择"/>
                      </el-form-item>
                    </el-col>
                    <el-col :span="4" style="text-align: center">
                      <el-button link @click="handleRowDel(index)" v-if="formData.itemPlanVoList.length>1">
                        <Minus style="color:orange;width: 1em; height: 1em; margin-right: 5px"/>
                      </el-button>
                      <el-button link @click="handleRowAdd">
                        <Plus style="color:red;width: 1em; height: 1em; margin-right: 5px"/>
                      </el-button>
                    </el-col>
                  </el-row>
                </template>

              </el-col>
            </el-form-item>
          </el-col>
        </el-row>
      </el-form>
      <div class="dialog-footer" v-if="!isLook">
        <el-button type="primary" @click="submitForm">确定</el-button>
        <el-button @click="cancel()">取消</el-button>
      </div>
    </el-dialog>
    <!-- //////////////////////////////// -->
  </div>
</template>

<script setup name="Plan">
import {
  listNursingPlan,
  getNursingPlan,
  delNursingPlan,
  addNursingPlan,
  updateNursingPlan,
  planStatus,
} from '@/api/nursing/plan'
import {getAllItems} from '@/api/nursing/item.js'
import {onMounted} from 'vue'
import {Minus, Plus} from "@element-plus/icons-vue"

const {proxy} = getCurrentInstance()
const {nursing_plan_status} = proxy.useDict('nursing_plan_status')
const queryRef = ref() // 表单
const planList = ref([])
const open = ref(false)
const loading = ref(true)
const showSearch = ref(true)
const ids = ref([])
const single = ref(true)
const multiple = ref(true)
const total = ref(0)
const title = ref('')
const dialogVisible = ref(false)
const nursingItemOptions = ref([])
const isLook = ref(false)
const formData = reactive({
  id: null,
  planName: null,
  status: null,
  sortNo: null,
  itemPlanVoList: [
    {
      itemId: null,
      executeTime: null,
      executeCycle: null,
      executeFrequency: null,
    },
  ]
})
// const planRef = ref({})

const queryParams = reactive({
  pageNum: 1,
  pageSize: 10,
  planName: null,
  status: null,
})

const rules = ref({
  planName: [
    {
      required: true,
      message: '护理计划名称为空，请输入护理计划名称',
      trigger: 'blur',
    },
  ],
  status: [
    {
      required: true,
      message: '状态为空，请选择状态',
      trigger: 'change',
    },
  ],
  sortNo: [
    {
      required: true,
      message: '排序为空，请选择排序',
      trigger: 'change',
    },
  ],
})

const executeCycleOptions = ref([
  {label: '每天', value: 0},
  {label: '每周', value: 1},
  {label: '每月', value: 2},
])

onMounted(() => {
  getAllItemList()
})

//查询所有护理项目
const getAllItemList = () => {
  getAllItems().then((res) => {
    nursingItemOptions.value = res.data
  })
}
const handleRowAdd = () => {
  formData.itemPlanVoList.push({
    itemId: null,
    executeTime: null,
    executeCycle: null,
    executeFrequency: null,
  })
}
//删除行数据
const handleRowDel = (index) => {
  if (formData.itemPlanVoList.length === 1) return
  formData.itemPlanVoList.splice(index, 1)
}

/** 查询护理计划列表 */
function getNursingPalnList() {
  loading.value = true
  listNursingPlan(queryParams).then((response) => {
    planList.value = response.rows
    total.value = response.total
    loading.value = false
  })
}

// 取消按钮
function cancel() {
  dialogVisible.value = false
  isLook.value = false
  reset()
}

// 表单重置
function reset(data) {
  if (data) {
    formData.id = data.id
    formData.planName = data.planName
    formData.status = data.status
    formData.sortNo = data.sortNo
    formData.itemPlanVoList = data.itemPlanVoList
  } else {
    formData.id = null
    formData.planName = null
    formData.status = null
    formData.sortNo = null
    formData.itemPlanVoList = [{
      itemId: null,
      executeTime: null,
      executeCycle: null,
      executeFrequency: null,
    }]
  }
}

/** 搜索按钮操作 */
function handleQuery() {
  queryParams.pageNum = 1
  getNursingPalnList()
}

/** 重置按钮操作 */
function resetQuery() {
  queryParams.pageNum = 1
  queryParams.pageSize = 10
  queryParams.planName = null
  queryParams.status = null
  dialogVisible.value = false
  handleQuery()
}

// 多选框选中数据
function handleSelectionChange(selection) {
  ids.value = selection.map((item) => item.id)
  single.value = selection.length != 1
  multiple.value = !selection.length
}

/** 新增按钮操作 */
function handleAdd() {
  dialogVisible.value = true
  formData.status = 0
  title.value = '添加护理计划'
}

/** 修改按钮操作 */
function handleUpdate(row) {
  const _id = row.id || ids.value
  getDetails(_id)
}

// 获取详情
const getDetails = (id) => {
  getNursingPlan(id).then((response) => {
    // formData = response.data
    reset(response.data)
    dialogVisible.value = true
    title.value = '修改护理计划'
    console.log("---> getDetails formData", formData)
  })
}

/** 提交按钮 */
function submitForm() {
  //判断选择的护理项目是否有重复的
  console.log(formData.itemPlanVoList)
  if (hasDuplicateIds(formData.itemPlanVoList)) {
    // 提示重复
    proxy.$modal.msgError('请勿选择重复的护理项目')
    return
  }

  proxy.$refs['planRef'].validate((valid) => {
    if (valid) {
      console.log("====> submitForm", formData)
      if (formData.id != null) {
        updateNursingPlan(formData).then((response) => {
          proxy.$modal.msgSuccess('修改成功')
          cancel()
          getNursingPalnList()
        })
      } else {
        addNursingPlan(formData).then((response) => {
          proxy.$modal.msgSuccess('新增成功')
          cancel()
          getNursingPalnList()
        })
      }
    }
  })
}

//帮我使用js代码，编写一段逻辑，判断数组中的对象的id是否有相同的(问心一言提供)
function hasDuplicateIds(objectsArray) {
  // 如果没有提供数组或数组为空，则直接返回false（因为没有元素可比较）
  if (!Array.isArray(objectsArray) || objectsArray.length === 0) {
    return false
  }

  // 创建一个新的Set来存储id
  const idSet = new Set()

  // 遍历数组中的每个对象
  for (const obj of objectsArray) {
    // 假设每个对象都有一个id属性
    if (obj.itemId !== undefined) {
      // 尝试将id添加到Set中
      idSet.add(obj.itemId)
    }
  }

  // 比较Set的大小和数组的大小
  // 如果Set的大小小于数组的大小，说明有重复的id
  return idSet.size !== objectsArray.length
}

// 使用 async/await 语法优化异步操作
const handleEnable = async (row) => {
  try {
    // 获取状态
    const status = row.status
    const info = status === 0 ? '启用' : '禁用'

    // 使用模板字符串
    const confirmMessage = `是否确认${info} ${row.planName}？`

    // 确认操作
    if (await proxy.$modal.confirm(confirmMessage)) {
      // 更新参数
      const param = {
        id: row.id,
        status: status === 0 ? 1 : 0,
      }

      // 执行更新操作
      await planStatus(param)
      // 刷新列表
      getNursingPalnList()
      // 成功消息
      proxy.$modal.msgSuccess(`${info}成功`)
    }
  } catch (error) {
    // 异常处理：这里可以根据实际需求进行调整，例如打印错误日志或显示用户友好的错误信息
    console.error('操作失败，请重试或联系管理员。')
  }
}

/** 删除按钮操作 */
function handleDelete(row) {
  const _ids = row.id || ids.value
  proxy.$modal
      .confirm('是否确认删除护理计划编号为"' + _ids + '"的数据项？')
      .then(function () {
        return delNursingPlan(_ids)
      })
      .then(() => {
        getNursingPalnList()
        proxy.$modal.msgSuccess('删除成功')
      })
      .catch(() => {
      })
}

//监听排序
const textBlurNo = () => {
  const data = formData.sortNo
  if (data <= 1) {
    formData.sortNo = 1
  }
  // 最大值设置
  if (data > 99999999) {
    formData.sortNo = 99999999
  }
}
// 查看
const handleLook = (row) => {
  isLook.value = true
  dialogVisible.value = true
  getDetails(row.id)
}
getNursingPalnList()
</script>
<style scoped></style>
