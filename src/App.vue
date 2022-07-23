<template>
  <div class="common-layout">
    <el-container>
      <el-header>
        <el-menu :default-active="activeIndex" class="el-menu-demo" mode="horizontal" :ellipsis="false"
          @select="handleSelect">
          <el-menu-item index="0">
            <el-image style="width: 280px; height: 80px" :src="logo" />
          </el-menu-item>
          <div class="flex-grow" />
        </el-menu>
      </el-header>
      <el-main>
        <el-card class="box-card">
          <template #header>
            <div class="card-header">
              <span>SQL建表语句快速生成器</span>
              <el-button type="primary" style="margin-left: 16px" @click="drawer = true">说明文档</el-button>
            </div>
          </template>
          <el-upload class="upload-demo" action="" drag :auto-upload="false" :on-change="uploadChange" :limit="1">
            <i class="el-icon-upload"></i>
            <div class="el-upload__text">将文件拖到此处，或<em>点击上传</em></div>
          </el-upload>
        </el-card>
        <el-card class="box-card">
          <template #header>
            <div class="card-header">
              <span>文件内容展示</span>
            </div>
          </template>
          <el-table :data="tableData" style="width: 100%" border>
            <el-table-column prop="index" label="序号" type="index" width="150" />
            <el-table-column prop="fieldName" label="字段名" />
            <el-table-column prop="type" label="类型" />
            <el-table-column prop="length" label="长度" />
            <el-table-column prop="describe" label="描述" />
            <el-table-column fixed="right" label="操作" width="400">
              <template #default="scope">
                <el-button size="small" @click="setNotNull(scope.$index, scope.row)">{{ setNotNullName }}</el-button>
                <el-button size="small" @click="setSequence(scope.$index, scope.row)">{{ setSequenceName }}</el-button>
                <el-button size="small" @click="setDeValue(scope.$index, scope.row)">默认值</el-button>
                <el-button size="small" type="danger" @click="setKey(scope.$index, scope.row)">{{ setKeyName }}
                </el-button>
              </template>
            </el-table-column>
          </el-table>
          <el-dialog v-model="dialogVisible" title="Tips" width="30%" :before-close="handleClose">
            <span>请输入默认值</span>
            <el-input v-model="defaultValue" placeholder="Please input" />
            <template #footer>
              <span class="dialog-footer">
                <el-button @click="dialogVisible = false">取消</el-button>
                <el-button type="primary" @click="getDeValue">确定</el-button>
              </span>
            </template>
          </el-dialog>
        </el-card>
        <el-card class="box-card">
          <template #header>
            <div class="card-header">
              <span>自定义规则</span>
            </div>
          </template>
          <el-row>
            <el-col :span="24">
              <el-input v-model="tableName" placeholder="请输入表名" clearable>
                <template #prepend>表名:</template>
              </el-input>
            </el-col>
            <el-col :span="24" style="margin-top:20px">
              <el-input v-model="tableNameCN" placeholder="请输入表中文名" clearable>
                <template #prepend>表中文名:</template>
              </el-input>
            </el-col>
            <el-col :span="24" style="margin-top:20px">
              <el-input v-model="ruleName" placeholder="请输入规则" clearable>
                <template #prepend>规则:</template>
              </el-input>
            </el-col>
            <el-col :style="{ textAlign: 'center', marginTop: '20px' }" :span="24">
              <el-button type="primary" @click="generateSql">生成语句</el-button>
            </el-col>
          </el-row>
        </el-card>
        <el-card class="box-card">
          <template #header>
            <div class="card-header">
              <span>SQL语句展示</span>
              <el-button type="primary" style="margin-left: 16px" @click="copy">复制SQL语句</el-button>
            </div>
          </template>
          <el-input v-model="sql" :rows="50" type="textarea" disabled />
        </el-card>
      </el-main>
      <el-drawer v-model="drawer" title="使用说明" :direction="direction">
        <span>1.文档格式</span>
        <el-image :src="src" />
      </el-drawer>
    </el-container>
  </div>
</template>
<script>
import { defineComponent, ref, reactive } from 'vue'
import * as XLSX from 'xlsx' // vue3可用此引入
import { format } from 'sql-formatter'
import useClipboard from 'vue-clipboard3'
export default defineComponent({
  name: 'upload',
  setup()
  {
    const { toClipboard } = useClipboard()
    // 提示信息
    const warning = () =>
    {
      ElMessage({
        message: 'excel中无内容，请上传有内容的文件！',
        type: 'warning',
      })
    }
    const uploadChange = async (e) =>
    {
      console.log(e)
      const files = e.raw
      if (files.length <= 0)
      {
        return false
      } else if (!/\.(xls|xlsx)$/.test(files.name.toLowerCase()))
      {
        ElMessage({
          message: '上传格式不正确，请上传xls或者xlsx格式',
          type: 'error',
        })
        return false
      }
      // 读取表格
      const fileReader = new FileReader()
      fileReader.onload = (ev) =>
      {
        const workbook = XLSX.read(ev.target.result, {
          type: 'binary',
        })
        const wsname = workbook.SheetNames[0]
        const ws = XLSX.utils.sheet_to_json(workbook.Sheets[wsname])
        // console.log(ws); // 转换成json的数据
        if (ws.length === 0)
        {
          warning()
        } else
        {
          tableData.value = dealExcel(ws) // ...对数据进行自己需要的操作
          console.log(tableData.value)
        }
      }
      fileReader.readAsBinaryString(files)
    }
    const dealExcel = (ws) =>
    {
      let keymap = {
        // 我们要转换的开头
        字段名: 'fieldName',
        类型: 'type',
        长度: 'length',
        描述: 'describe',
      }
      const test = []
      ws.forEach((sourceObj) =>
      {
        const test2 = {}
        for (let key in sourceObj)
        {
          if (keymap[key])
          {
            test2[keymap[key]] = sourceObj[key]
          }
        }
        test.push(test2)
      })
      return test
    }
    // 拼接生产sql语句
    const generateSql = () =>
    {
      if (tableData.value.length === 0)
      {
        ElMessage({
          message: '请上传带有内容的excel表格文件！',
          type: 'warning',
        })
        return false
      }
      if (tableName.value && tableNameCN.value)
      {
        let sqlHeader = `CREATE TABLE '${tableName.value}' (`
        let sqlContent = ''
        // 定义主键语句
        let sqlPrimaryKey = ''
        // 是否存在主键
        let isKey = false
        tableData.value.forEach((item, index) =>
        {
          if (item.primaryKey)
          {
            sqlPrimaryKey = `PRIMARY KEY ('${item.fieldName}')`
            isKey = true
          }
          if (index + 1 < tableData.value.length)
          {
            sqlContent += `'${item.fieldName}' ${item.type}(${item.length})${item.notNull ? ' NOT NULL ' : ' '}${item.sequence ? ' AUTO_INCREMENT ' : ' '}COMMENT '${item.describe}',`
          } else if (index + 1 === tableData.value.length)
          {
            if (isKey)
            {
              sqlContent += `'${item.fieldName}' ${item.type}(${item.length})${item.notNull ? ' NOT NULL ' : ' '}${item.sequence ? ' AUTO_INCREMENT ' : ' '}COMMENT '${item.describe}',` + sqlPrimaryKey
            } else
            {
              sqlContent += `'${item.fieldName}' ${item.type}(${item.length})${item.notNull ? ' NOT NULL ' : ' '}${item.sequence ? ' AUTO_INCREMENT ' : ' '}COMMENT '${item.describe}'`
            }
          }

        })
        let sqlFooter = `) COMMENT='${tableNameCN.value}';`
        sql.value = format(sqlHeader + sqlContent + sqlFooter)
        console.log(format(sqlHeader + sqlContent + sqlFooter))
      } else
      {
        ElMessage({
          message: '请将内容填写完整！',
          type: 'warning',
        })
      }
    }
    let state01 = ref(false)
    // 点击设置非空
    const setNotNull = (index, row) =>
    {
      state01.value = !state01.value
      tableData.value[index]['notNull'] = state01.value
      state01.value ? setNotNullName.value = '取消非空' : setNotNullName.value = '设置非空'
    }
    const state02 = ref(false)
    // 设置主键
    const setKey = (index, row) =>
    {
      state02.value = !state02.value
      tableData.value[index]['primaryKey'] = state02.value
      state02.value ? setKeyName.value = '取消主键' : setKeyName.value = '设为主键'
    }
    let state03 = ref(false)
    // 设置为序列
    const setSequence = (index, row) =>
    {
      state03.value = !state03.value
      tableData.value[index]['sequence'] = state03.value
      state03.value ? setSequenceName.value = '取消序列' : setSequenceName.value = '设置序列'
    }
    // 设置默认值
    const setDeValue = (index, row) =>
    {
      // tableData.value[index]['sequence'] = true
      dialogVisible.value = true
    }
    const getDeValue = () =>
    {
      dialogVisible.value = false
    }
    // 实现复制功能
    const copy = async () =>
    {
      if (sql.value === '')
      {
        ElMessage({
          message: '别着急，还未生成sql语句呢！',
          type: 'success',
        })
        return false
      }
      try
      {
        // 复制
        await toClipboard(sql.value)
        ElMessage({
          message: '复制成功！',
          type: 'success',
        })
      } catch (e)
      {
        ElMessage({
          message: `复制失败:${e}`,
          type: 'error',
        })
      }
    }
    // 定义表格中要展示的数据
    const tableData = ref([])
    // 控制文档窗口
    const drawer = ref(false)
    // 控制文档窗口的位置 rtl / ltr / ttb / btt
    const direction = ref('rtl')
    // 定义变量接收表名
    const tableName = ref('')
    // 定义变量接收规则
    const ruleName = ref('')
    // 定义表中文名
    const tableNameCN = ref('')
    // 拼接好的sql语句
    const sql = ref('')
    // 定义图片路径
    const src = ref('https://img.codestu.cn/2022/07/23/49e58e1f8a60e.png')
    const logo = ref('https://img.codestu.cn/2022/07/23/a5d2fcd3db672.png')
    // 设置非空名称
    const setNotNullName = ref('设置非空')
    // 设置序列名称
    const setSequenceName = ref('设置序列')
    // 设置设为主键名称
    const setKeyName = ref('设为主键')
    // 定义默认值
    const defaultValue = ref('')
    // 定义对话框开关状态
    const dialogVisible = ref(false)
    return {
      setNotNullName,
      setSequenceName,
      setKeyName,
      uploadChange,
      tableData,
      drawer,
      direction,
      tableName,
      tableNameCN,
      ruleName,
      generateSql,
      sql,
      src,
      logo,
      setNotNull,
      setKey,
      setSequence,
      setDeValue,
      copy,
      defaultValue,
      dialogVisible,
      getDeValue
    }
  },
})
</script>



<style scoped>
</style>
