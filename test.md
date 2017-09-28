# Element UI table配合Pagination使用

今天我们来讲解一下使用Element UI Table组件，将要实现以下功能。

- 使用Element UI Table组件渲染JSON数据
- 使用Element UI 的Pagination组件实现表格的分页
- 将当前的JSON数据导出CVS格式的文件

废话不多说，直接开始。

1.首先引入官网的样式文件和相应的JS文件

```html
<!-- 引入样式 -->
<link rel="stylesheet" href="https://unpkg.com/element-ui/lib/theme-default/index.css">
<!-- 引入导出CSV的文件-->
<script src="export.js"></script>
<!-- 先引入 Vue -->
<script src="https://unpkg.com/vue/dist/vue.js"></script>
<!-- 引入组件库 -->
<script src="https://unpkg.com/element-ui/lib/index.js"></script>
```

2.页面布局

```html
<div id="main">
  <template>
    <el-card class="box-card">
      <div style="margin-bottom: 15px;">
        <el-button type="primary"  @click="ExportFunc">导出</el-button>
      </div>
      <!-- 表格 -->
        <el-table v-loading="loading" :element-loading-text="loadingText" :empty-text="emptyText" :data="tableData" stripe border style="margin-bottom: 15px; width: 100%;">
            <el-table-column v-for="col in cols" :prop="col.prop" :label="col.label" min-width="100">
            </el-table-column>
        </el-table>
        <!-- 分页 -->
        <el-pagination @size-change="handleSizeChange" @current-change="handleCurrentChange" :current-page="currentPage" :page-size="PageSize" layout="total, sizes, prev, pager, next, jumper" :total="count">
        </el-pagination>
    </el-card>
  </template>
</div>
```
3.javascript 代码
```javascript
var Main = {
  data: function() {
    return {
      tableData: [], // 表格加载的数据
      AllData: [], // 全部数据
      cols: [],
      PageSize: 10,
      count:0,
      currentPage: 1,
      emptyText: '暂无数据',
      loading: true,
      loadingText: '拼命加载中'
    }
  },
  mounted: function() {
    this.initData();
  },
  methods: {
    /*
    * data 数据
    * PageSize 每页条数 10
    * PageNo 当前页数 1
    */
    getCurData: function(data, PageSize, PageNo) {
      var Pedata = [], idx;
      for(let i = 0; i < PageSize; i++) {
        idx = PageSize * PageNo - PageSize + i;
        if(idx < data.length) Pedata.push(data[idx]);
      }
      return Pedata;
    },
    initData: function() {
      // 数据请求 fetch是ES6新增的
      var seft = this;
      fetch('./json.json', {method: 'GET', cache: 'reload'}).then(function(res) {
        if(res['ok']) {
          seft.loading = false;
          seft.emptyText = '正在整理数据';
          res.json().then(function(data) {
            seft.count = data.length;
            seft.AllData = data;
            let arr = [];
            for(let key in data[0]) {
              if(data[0].hasOwnProperty(key)) {
                arr.push({prop: key, label: key})
              }
            }
            seft.cols = arr;
            seft.tableData = seft.getCurData(data, seft.PageSize, seft.currentPage);
          })
        }
      }).catch(function() {
        seft.emptyText = '请求失败';
        seft.loading = false;
      });
    },
    handleSizeChange: function(val) {
      var pageSize = val,
      pageNo = this.currentPage,
      data = this.AllData;
      this.PageSize = pageSize;
      this.tableData = this.getCurData(data, pageSize, pageNo);
    },
    handleCurrentChange: function(val) {
      var pageSize = this.PageSize,
      pageNo = val,
      data = this.AllData;
      this.currentPage = val;
      this.tableData = this.getCurData(data, pageSize, pageNo);

    },
    ExportFunc: function() {
      /*
      * 参数 data 是导出的数据
      * 参数 title 是导出的标题
      * showLabel 表示是否显示表头 默认显示
      */
      JSONToCSVConvertor({data:this.AllData, title: '这里导出的标题', showLabel: true});
    }
  }
};
var Ctor = Vue.extend(Main);
new Ctor().$mount('#main');
```
