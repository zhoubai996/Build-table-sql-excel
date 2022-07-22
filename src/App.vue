<script>
import { defineComponent, ref, reactive } from 'vue'
import * as XLSX from 'xlsx' // vue3可用此引入
import { format } from 'sql-formatter'
export default defineComponent({
  name: 'upload',
  setup() {
    // 提示信息
    const warning = () => {
      ElMessage({
        message: 'excel中无内容，请上传有内容的文件！',
        type: 'warning',
      })
    }
    const uploadChange = async (e) => {
      console.log(e)
      const files = e.raw
      if (files.length <= 0) {
        return false
      } else if (!/\.(xls|xlsx)$/.test(files.name.toLowerCase())) {
        console.log('上传格式不正确，请上传xls或者xlsx格式')
        return false
      }
      // 读取表格
      const fileReader = new FileReader()
      fileReader.onload = (ev) => {
        const workbook = XLSX.read(ev.target.result, {
          type: 'binary',
        })
        const wsname = workbook.SheetNames[0]
        const ws = XLSX.utils.sheet_to_json(workbook.Sheets[wsname])
        // console.log(ws); // 转换成json的数据
        if (ws.length === 0) {
          warning()
        } else {
          tableData.value = dealExcel(ws) // ...对数据进行自己需要的操作
          console.log(tableData.value)
        }
      }
      fileReader.readAsBinaryString(files)
    }
    const dealExcel = (ws) => {
      let keymap = {
        // 我们要转换的开头
        字段名: 'fieldName',
        类型: 'type',
        长度: 'length',
        描述: 'describe',
      }
      const test = []
      ws.forEach((sourceObj) => {
        const test2 = {}
        for (let key in sourceObj) {
          if (keymap[key]) {
            test2[keymap[key]] = sourceObj[key]
          }
        }
        test.push(test2)
      })
      return test
    }
    // 拼接生产sql语句
    const generateSql = () => {
      if (tableName.value && tableNameCN.value) {
        let sqlHeader = `CREATE TABLE ${tableName.value} (`
        let sqlContent = ''
        tableData.value.forEach((item) => {
          sqlContent += `'${item.fieldName}' ${item.type}(${item.length}) COLLATE utf8_unicode_ci NOT NULL COMMENT '${item.describe}',`
        })
        let sqlFooter = `) ENGINE=MyISAM DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci COMMENT='${tableNameCN.value}';`
        sql.value = format(sqlHeader + sqlContent + sqlFooter)
        console.log(format(sqlHeader + sqlContent + sqlFooter))
      } else {
        ElMessage({
          message: '请将内容填写完整！',
          type: 'warning',
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
    return {
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
      logo
    }
  },
})
</script>

<template>
  <div class="common-layout">
    <el-container>
      <el-header>
        <el-menu :default-active="activeIndex" class="el-menu-demo" mode="horizontal" :ellipsis="false" @select="handleSelect">
          <el-menu-item index="0"><el-image style="width: 280px; height: 80px" :src="logo" /></el-menu-item>
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
          <el-table :data="tableData" style="width: 100%">
            <el-table-column prop="index" label="序号" type="index" width="180" />
            <el-table-column prop="fieldName" label="字段名" />
            <el-table-column prop="type" label="类型" />
            <el-table-column prop="length" label="长度" />
            <el-table-column prop="describe" label="描述" />
          </el-table>
        </el-card>
        <el-card class="box-card">
          <template #header>
            <div class="card-header">
              <span>自定义规则</span>
            </div>
          </template>
          <el-row>
            <el-col :span="24"
              ><el-input v-model="tableName" placeholder="请输入表名" clearable>
                <template #prepend>表名:</template>
              </el-input></el-col
            >
            <el-col :span="24" style="margin-top:20px"
              ><el-input v-model="tableNameCN" placeholder="请输入表中文名" clearable>
                <template #prepend>表中文名:</template>
              </el-input></el-col
            >
            <el-col :span="24" style="margin-top:20px"
              ><el-input v-model="ruleName" placeholder="请输入规则" clearable>
                <template #prepend>规则:</template>
              </el-input></el-col
            >
            <el-col :style="{ textAlign: 'center', marginTop: '20px' }" :span="24"><el-button type="primary" @click="generateSql">生成语句</el-button></el-col>
          </el-row>
        </el-card>
        <el-card class="box-card">
          <template #header>
            <div class="card-header">
              <span>SQL语句展示</span>
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

<style scoped></style>
