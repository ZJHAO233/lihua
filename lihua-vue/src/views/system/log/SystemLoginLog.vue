<template>
  <div>
    <a-flex :gap="16" vertical>
      <!--     检索条件-->
      <a-card :style="{border: 'none'}" :body-style="{'padding-bottom': '0'}">
        <a-form :colon="false">
          <a-row :gutter="16">
            <a-col>
              <a-form-item label="用户名">
                <a-input v-model:value="logQuery.username" placeholder="请输入用户名" allow-clear/>
              </a-form-item>
            </a-col>
            <a-col>
              <a-form-item label="操作人员">
                <a-input v-model:value="logQuery.createName" placeholder="请输入操作人员" allow-clear/>
              </a-form-item>
            </a-col>
            <a-col>
              <a-form-item label="登录状态">
                <a-select v-model:value="logQuery.executeStatus" placeholder="请选择" style="width: 120px" allow-clear>
                  <a-select-option :value="item.value" v-for="item in sys_log_status">{{item.label}}</a-select-option>
                </a-select>
              </a-form-item>
            </a-col>
            <a-col>
              <a-form-item label="登录时间">
                <a-range-picker allowClear v-model:value="logQuery.createTimeList" />
              </a-form-item>
            </a-col>
            <a-col>
              <a-form-item>
                <a-space size="small">
                  <a-button type="primary" @click="queryPage" :loading="tableLoad">
                    <template #icon>
                      <SearchOutlined />
                    </template>
                    查 询
                  </a-button>
                  <a-button :loading="tableLoad" @click="reloadPage">
                    <template #icon>
                      <RedoOutlined />
                    </template>
                    重 置
                  </a-button>
                </a-space>
              </a-form-item>
            </a-col>
          </a-row>
        </a-form>
      </a-card>
      <!--     列表-->
      <a-table
          :pagination="false"
          :columns="logColumn"
          :data-source="logList"
          :loading="tableLoad"
          :row-selection="logRowSelectionType"
          row-class-name="hover-cursor-pointer"
          :custom-row="handleRowClick"
          row-key="id"
          :scroll="{x: 1500}"
      >
        <template #title>
          <a-flex :gap="8">
            <a-popconfirm title="删除后不可恢复，是否删除？"
                          :open="openDeletePopconfirm"
                          ok-text="确 定"
                          cancel-text="取 消"
                          @confirm="handleDelete"
                          @cancel="closePopconfirm"
                          @open-change="(open: boolean) => !open ? closePopconfirm(): ''"
            >
              <a-button danger @click="openPopconfirm">
                <template #icon>
                  <DeleteOutlined />
                </template>
                删 除
                <span v-if="selectedIds && selectedIds.length > 0" style="margin-left: 4px"> {{selectedIds.length}} 项</span>
              </a-button>
            </a-popconfirm>

            <a-button ghost type="primary" @click="handleExportExcel">
              <template #icon>
                <ExportOutlined />
              </template>
              导出
            </a-button>

            <a-popconfirm title="清空后不可恢复，是否清空？"
                          :open="openClearPopconfirm"
                          @cancel="handleCloseClearPopconfirm"
                          @open-change="(open: boolean) => !open ? handleCloseClearPopconfirm(): ''"
            >
              <template #okButton>
                <a-button size="small" type="primary" :disabled="countdown" @click="handleClear">
                  <a-statistic-countdown
                      v-if="countdown"
                      format="s"
                      :value="Date.now() + 1000 * 6"
                      :value-style="{'font-size': '14px'}"
                      @finish="countdownFinish"
                  />
                  <a-typography-text v-else style="color: #fff">确 认</a-typography-text>
                </a-button>
              </template>
              <a-button danger type="primary" @click="handleOpenClearPopconfirm">
                <template #icon>
                  <DeleteFilled />
                </template>
                清 空
              </a-button>
            </a-popconfirm>
          </a-flex>
        </template>
        <template #bodyCell="{column,record,text}">
          <template v-if="column.key === 'description'">
            <a-tooltip placement="topLeft" ellipsis>
              <template #title>
                {{text}}
              </template>
              <a-typography-link @click="(event:MouseEvent) => selectById(event, record.id)">
                {{text}}
              </a-typography-link>
            </a-tooltip>
          </template>
          <template v-if="column.key === 'executeStatus' && text">
            <dict-tag :dict-data-option="sys_log_status" :dict-data-value="text"></dict-tag>
          </template>
          <template v-if="column.key === 'createTime'">
            {{dayjs(text).format('YYYY-MM-DD HH:mm:ss')}}
          </template>
          <template v-if="column.key === 'executeTime'">
            {{text ? text + ' 毫秒' : ''}}
          </template>
        </template>
        <template #footer>
          <a-flex justify="flex-end">
            <a-pagination v-model:current="logQuery.pageNum"
                          v-model:page-size="logQuery.pageSize"
                          show-size-changer
                          :total="logTotal"
                          :show-total="(total:number) => `共 ${total} 条`"
                          @change="initPage"/>
          </a-flex>
        </template>
      </a-table>
    </a-flex>
    <!--   日志详情模态框-->
    <a-modal cancelText="关 闭" v-model:open="openModal" width="1000px" :footer="null">
      <a-descriptions title="日志详情" bordered :label-style="{width: '110px'}">
        <a-descriptions-item label="业务描述" :span="1">
          <a-badge status="success" v-if="logInfo.executeStatus === '0'"/>
          <a-badge status="error" v-else/>
          {{logInfo.description}}
        </a-descriptions-item>
        <a-descriptions-item label="业务类型" :span="1">{{logInfo.typeMsg}}</a-descriptions-item>
        <a-descriptions-item label="用户名" :span="1">{{logInfo.username}}</a-descriptions-item>
        <a-descriptions-item label="类名" :span="2">{{logInfo.className}}</a-descriptions-item>
        <a-descriptions-item label="方法名" :span="1">{{logInfo.methodName}}</a-descriptions-item>
        <a-descriptions-item label="参数" :span="3">{{logInfo.params}}</a-descriptions-item>
        <a-descriptions-item label="操作人" :span="1">{{logInfo.createName}}</a-descriptions-item>
        <a-descriptions-item label="操作时间" :span="1">{{dayjs(logInfo.createTime).format("YYYY-MM-DD HH:mm:ss")}}</a-descriptions-item>
        <a-descriptions-item label="执行时长" :span="1">{{logInfo.executeTime ? logInfo.executeTime + ' 毫秒' : ''}}</a-descriptions-item>
        <a-descriptions-item label="请求地址" :span="2">{{logInfo.url}}</a-descriptions-item>
        <a-descriptions-item label="用户 ip" :span="1">{{logInfo.ipAddress}}</a-descriptions-item>
        <a-descriptions-item label="缓存key" :span="3">
          {{logInfo.cacheKey}}
        </a-descriptions-item>
        <a-descriptions-item label="操作环境" :span="3">
          {{logInfo.userAgent}}
        </a-descriptions-item>
        <a-descriptions-item label="异常信息" :span="3" v-if="logInfo.executeStatus === '1'">{{logInfo.errorMsg}}</a-descriptions-item>
        <a-descriptions-item label="异常堆栈" :span="3" v-if="logInfo.executeStatus === '1'">{{logInfo.errorStack}}</a-descriptions-item>
      </a-descriptions>
    </a-modal>
  </div>
</template>

<script setup lang="ts">
import {initDict} from "@/utils/Dict"
import {ref} from "vue";
import {
  findLoginPage,
  deleteLoginByIds,
  clearLoginLog,
  findLoginById,
  excelLoginExport
} from "@/api/system/log/Log.ts";
import {message} from "ant-design-vue";
import DictTag from "@/components/dict-tag/index.vue";
import type {SysLog, SysLogDTO} from "@/api/system/log/type/SysLog.ts";
import type {ColumnsType} from "ant-design-vue/es/table/interface";
import dayjs from "dayjs";
import {handleFunDownload} from "@/utils/FileDownload.ts";

const {sys_log_status} = initDict("sys_log_status")


const initSearch = () => {
  // 选中的数据id集合
  const selectedIds = ref<Array<string>>([])
  // 列表勾选对象
  const logRowSelectionType = {
    columnWidth: '55px',
    type: 'checkbox',
    // 支持跨页勾选
    preserveSelectedRowKeys: true,
    // 指定选中id的数据集合，操作完后可手动清空
    selectedRowKeys: selectedIds,
    onChange: (ids: Array<string>) => {
      selectedIds.value = ids
    }
  }
  const handleRowClick = (record:SysLog) => {
    return {
      onClick: () => {
        if (record.id) {
          const selected = selectedIds.value
          if (selected.includes(record.id)) {
            selected.splice(selected.indexOf(record.id),1)
          } else {
            selected.push(record.id)
          }
        }
      }
    }
  }

  const logColumn: ColumnsType = [
    {
      title: '日志描述',
      key: 'description',
      dataIndex: 'description',
      align: 'center'
    },
    {
      title: '用户名',
      key: 'username',
      dataIndex: 'username',
      align: 'center'
    },
    {
      title: '操作人员',
      key: 'createName',
      dataIndex: 'createName',
      align: 'center'
    },
    {
      title: 'ip 地址',
      key: 'ipAddress',
      dataIndex: 'ipAddress',
      align: 'center'
    },
    {
      title: '登陆结果',
      key: 'executeStatus',
      dataIndex: 'executeStatus',
      align: 'center'
    },
    {
      title: '失败原因',
      key: 'errorMsg',
      dataIndex: 'errorMsg',
      align: 'center',
      ellipsis: true
    },
    {
      title: '登陆时间',
      key: 'createTime',
      dataIndex: 'createTime',
      align: 'center'
    },
    {
      title: '操作耗时',
      key: 'executeTime',
      dataIndex: 'executeTime',
      align: 'center'
    }
  ]

  const reloadPage = async () => {
    logQuery.value = {
      pageNum: 1,
      pageSize: 10
    }
    await initPage()
  }

  const logQuery = ref<SysLogDTO>({
    pageNum: 1,
    pageSize: 10
  })

  const logTotal = ref<number>()
  const logList = ref<Array<SysLog>>([])
  const tableLoad = ref<boolean>(false)

  const queryPage = async () => {
    logQuery.value.pageNum = 1
    await initPage()
  }

  const initPage = async () => {
    tableLoad.value = true
    const resp = await findLoginPage(logQuery.value)
    if (resp.code === 200) {
      logTotal.value = resp.data.total
      logList.value = resp.data.records
      tableLoad.value = false
    } else {
      message.error(resp.msg)
    }
  }
  queryPage()
  return {
    logColumn,
    tableLoad,
    logList,
    logQuery,
    logTotal,
    logRowSelectionType,
    selectedIds,
    handleRowClick,
    queryPage,
    initPage,
    reloadPage
  }
}

const {logColumn, tableLoad, logList, logQuery, logTotal, logRowSelectionType, selectedIds, handleRowClick, queryPage, initPage, reloadPage} = initSearch()


// 初始化获取日志详情
const initLogInfo = () => {
  // 模态框
  const openModal = ref<boolean>(false)
  // 日志详情
  const logInfo = ref<SysLog>({})
  // 根据id查询日志详情
  const selectById = async (event:MouseEvent, id: string) => {
    event.stopPropagation()
    const resp = await findLoginById(id)

    if (resp.code === 200) {
      logInfo.value = resp.data
      // 关闭删除/清空提示框
      handleCloseClearPopconfirm()
      closePopconfirm()
      openModal.value = true
    } else {
      message.error(resp.msg)
    }
  }

  return {
    openModal,
    logInfo,
    selectById
  }
}

const {openModal, logInfo, selectById} = initLogInfo()

// 删除日志
const initDelete = () => {
  // 显示删除提示
  const openDeletePopconfirm = ref<boolean>(false);
  // 打开删除提示框
  const openPopconfirm = () => {
    if (selectedIds.value && selectedIds.value.length > 0) {
      openDeletePopconfirm.value = true
    } else {
      message.warning("请勾选数据")
    }
  }
  // 关闭删除提示框
  const closePopconfirm = () => {
    openDeletePopconfirm.value = false
  }
  // 处理删除逻辑
  const handleDelete = async () => {
    const deleteIds = [...selectedIds.value]

    if (deleteIds.length > 0) {
      const resp = await deleteLoginByIds(deleteIds)
      if (resp.code === 200) {
        message.success(resp.msg);
        selectedIds.value = []
        await initPage()
      } else {
        message.error(resp.msg)
      }
    } else {
      message.warning("请勾选数据")
    }
    closePopconfirm()
  }

  return {
    openDeletePopconfirm,
    closePopconfirm,
    handleDelete,
    openPopconfirm
  }
}

const {openDeletePopconfirm,closePopconfirm,handleDelete,openPopconfirm} = initDelete()

// 初始化清空日志
const initClear = () => {
  // 打开清空确认对话框
  const openClearPopconfirm = ref<boolean>(false)
  // 清空按钮倒计时
  const countdown = ref<boolean>(true)
  // 倒计时结束
  const countdownFinish = () => {
    countdown.value = false
  }
  // 处理打开提示框
  const handleOpenClearPopconfirm = () => {
    countdown.value = true
    openClearPopconfirm.value = true
  }
  // 处理关闭提示框
  const handleCloseClearPopconfirm = () => {
    openClearPopconfirm.value = false
  }
  // 处理清空日志
  const handleClear = async () => {
    const resp = await clearLoginLog()
    if (resp.code === 200) {
      message.success(resp.msg);
      selectedIds.value = []
      await initPage()
    } else {
      message.error(resp.msg);
    }
    handleCloseClearPopconfirm()
  }

  return {
    openClearPopconfirm,
    countdown,
    handleOpenClearPopconfirm,
    handleCloseClearPopconfirm,
    handleClear,
    countdownFinish
  }
}

const {openClearPopconfirm, countdown, handleOpenClearPopconfirm, handleCloseClearPopconfirm, handleClear, countdownFinish} = initClear()

// 处理excel 导出
const handleExportExcel = () => {
  handleFunDownload(excelLoginExport(logQuery.value))
}
</script>


<style scoped>

</style>