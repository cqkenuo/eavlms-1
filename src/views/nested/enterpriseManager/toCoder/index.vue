<template>
  <div class="app-container">
    <el-table
      v-loading="listLoading"
      :data="list"
      stripe
      element-loading-text="加载中"
      border
      max-height="390"
      height="390"
      fit
      highlight-current-row>
      <el-table-column align="center" label="索引" fixed width="60">
        <template slot-scope="scope">
          {{ scope.$index }}
        </template>
      </el-table-column>
      <el-table-column label="系统名" align="center" fixed prop="vulnSystemName"/>
      <el-table-column label="资产IP" align="center" fixed>
        <template  slot-scope="scope">
          {{ scope.row.assets.assetsIp }}
        </template>
      </el-table-column>

      <el-table-column label="资产URL" align="center">
        <template  slot-scope="scope">
          {{ scope.row.assets.assetsUrl }}
        </template>
      </el-table-column>

      <el-table-column label="维护部门" align="center">
        <template  slot-scope="scope">
          {{ scope.row.assets.maintenanceDepartment }}
        </template>
      </el-table-column>
      <el-table-column label="使用部门" align="center">
        <template  slot-scope="scope">
          {{ scope.row.assets.belongedDepartment }}
        </template>
      </el-table-column>
      <el-table-column label="资产性质" align="center">
        <template  slot-scope="scope">
          {{ scope.row.assets.attributes }}
        </template>
      </el-table-column>
      <el-table-column label="维护接口人" align="center">
        <template  slot-scope="scope">
          {{ scope.row.assets.mainTenanceContactPerson }}
        </template>
      </el-table-column>
      <el-table-column label="安全接口人" align="center">
        <template  slot-scope="scope">
          {{ scope.row.assets.safetyContactPerson }}
        </template>
      </el-table-column>
      <el-table-column label="安全部门" align="center">
        <template  slot-scope="scope">
          {{ scope.row.assets.safetyDepartment }}
        </template>
      </el-table-column>

      <el-table-column label="漏洞名称" align="center" prop="vulnsName"/>
      <el-table-column label="漏洞发现者" align="center" prop="vulnsCreatePerson"/>

      <el-table-column class-name="status-col" label="漏洞等级" align="center">
        <template slot-scope="scope">
          <el-tag :type="scope.row.vulnsRand | vulnsRand">{{ scope.row.vulnsRand }}</el-tag>
        </template>
      </el-table-column>
      <el-table-column align="center" label="发现时间" prop="vulnsCreateTime"/>

      <el-table-column label="操作" fixed="right" width="150" align="center">
        <template slot-scope="scope">
          <el-button size="mini" @click="coderHandleClick(scope.row)">下发</el-button> <!--给开发人员发整改邮件-->

          <el-popconfirm
            icon="el-icon-info"
            iconColor="red"
            @onConfirm="misreportHandleClick(scope.row)"
            title="确定为误报吗？">
            <el-button slot="reference"
                       size="mini" type="danger">误报
            </el-button>
          </el-popconfirm>
        </template>
      </el-table-column>
    </el-table>
    <!--    分页-->
    <el-row type="flex" justify="end" style="margin-top: 50px">
      <el-pagination
        @size-change="e=>{pageSize=e;this.getAll()}"
        @current-change="e=>{currentPage=e;this.getAll()}"
        :current-page.sync="currentPage"
        :page-sizes="[10, 20, 30, 50]"
        :page-size="pageSize"
        layout="sizes, prev, pager, next"
        :total="total">
      </el-pagination>
    </el-row>
    <!--    选择用户（自己封装组件）src/components/user-type-dialog/index.vue -->
    <type-user-dialog :user-dialog-visible="userDialogVisible" user-type="1" @userSubmit="userHandleSubmit"
                      @userCancel="userHandleCancel"></type-user-dialog>
  </div>
</template>

<script>
  import TypeUserDialog from '@/components/user-type-dialog'
  import { vulnsState, vulnsRand, vulnsStateText } from '@/utils/vulns-info'
  export default {
    filters: {
      vulnsRand
    },
    components: {
      TypeUserDialog
    },
    data() {
      return {
        list: [],
        listLoading: true,
        pageSize: 10,
        total: 0,
        currentPage: 1,
        userDialogVisible: false,
        userSelection: null,
        vulnsSelection: null
      }
    },
    created() {
      //生命周期函数 周期开始就自动调用
      this.getAll()
    },

    methods: {
      //获取所有
      getAll() {
        this.listLoading = true
        this.$http.get('/assets-vulns', {
          //状态为0待修复的
          vulnsState: 0,
          page: this.currentPage - 1,
          size: this.pageSize
        }).then(response => {
          this.list = response.content   //后端传递的page分页 content是分页的内容
          console.log('getAll',this.list)
          this.total = response.totalElements   //页码数
          this.listLoading = false
        })
      },
      //下发给开发人员
      userHandleSubmit(user) {
        if (user) {  //判断是否传入
          let formUser = JSON.parse(sessionStorage.getItem('user'))  //json.parse() 将当前用户的session中的字符串转换为对象然后赋给enterprise参数formuser；从对象到json字符串的方法是json.stringify
          this.$http.post('/to-coder', {     //封装后台的tocoder对象
            coder: {   //将user的id赋给tocoder对象的coder属性
              id: user.id
            },
            enterprise: {
              id: formUser.id
            },
            date: new Date(),
            vulns: {
              id: this.vulnsSelection.id
            }
          }).then(v => {
            this.userDialogVisible = false
            this.getAll()
          })
        } else {
          this.userDialogVisible = true
        }
      },

      userHandleCancel() {
        this.userSelection = null
        this.userDialogVisible = false
      },
      coderHandleClick(row) {
        console.log('vulnsSelection', row);
        this.vulnsSelection = {id: row.id}
        this.userDialogVisible = true
      },
      //误报
      misreportHandleClick(row) {
        console.log('misreportHandleClick', row)
        this.$http.post('/assets-vulns/updateState', {
          id: row.id,
          vulnsState: 4,
          'createUser.id':JSON.parse(sessionStorage.getItem('user')).id //给user子对象的id属性赋值
        }).then(v => {
          this.getAll()
        })
      },
    }
  }
</script>
