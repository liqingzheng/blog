### 前言
将数据报表导出，是web数据报告展示常用的附带功能。通常这种功能都是用后端开发人员编写的。今天我们主要讲的是直接通过前端js将数据导出Excel的CSV格式的文件。
### 原理
首先在本地用Excel新建一个test.csv的文件 ===> 随便填写一些数据，保存并用Safari浏览打开该文件 ===> 打开浏览器的开发者工具，执行`JSON.stringify(document.body.innerText);`,我们得到结果如下图：
![图片描述][1]

从图中，可以看出：

 - CSV文件格式单元格之间是通过`,`隔开的
 - CSV文件格式里，换行是通过`\n`实现的

从上面两条结论，我们只有把相应的数据转换成`,`和`\n`就可以了。但其实真正的答案应该是把相应的数据转换成**`,`和`\r\n`**。
为什么会这样？且让我一一道来：
我们在编辑Excel文件时，当编辑完成当前单元格时，想要编辑下一行紧挨着的单元格，按一下`Enter`键就可以。而`Enter`键在js字符串中是用`\r`表示的。那是不是吧`\n`替换成`\r`就可以了呢？
其实不可以，因为涉及到操作系统的问题：

 - 在Windows系统中，标准模式采用的是`\r\n`匹配`Enter`键
 - 在mac系统中，用`\r`匹配`Enter`键
 - 在Linux系统中，用`\n`匹配`Enter`键

所以，最最最最终终终的结论是：
 - **将相应的数据转换成`,`和`\r\n`，即：`名称,熟练\r\n张三,2`**
 - **由于单元格之间使用`,`隔开，所以不支持单元格的合并行、合并列，其实这句话有点多余，CSV格式的文件本身就不支持单元格的合并列和行**
### 实现方式
在编写代码之前，我们先来看一下具体数据和样式。假如当前的JSON数据是这样的

```
[
    {name: '张三', amont: '323433.56', proportion: 33.4},
    {name: '李四', amont: '545234.43', proportion: 55.45}
]
```
数据报告展示样式如下：


| 姓名  | 金额     | 占比  |
| ---- |:------:| -----:|
| 张三 | 323433.56 | 33.40%|
| 李四 | 545234.43 | 55.45%|

那如何使得导出的数据与展示的保持一致呢？
答案是：

 - 把要展示的表头文字也进行处理
 - 遍历取对应的key值
 - 设置formatter回调处理的当前值的函数

由此我们得到如下代码：

```
var JSonToCSV = {
  /*
   * obj是一个对象，其中包含有：
   * ## data 是导出的具体数据
   * ## fileName 是导出时保存的文件名称 是string格式
   * ## showLabel 表示是否显示表头 默认显示 是布尔格式
   * ## columns 是表头对象，且title和key必须一一对应，包含有
        title:[], // 表头展示的文字
        key:[], // 获取数据的Key
        formatter: function() // 自定义设置当前数据的 传入(key, value)
   */
  setDataConver: function(obj) {
    var data = obj['data'],
        ShowLabel = typeof obj['showLabel'] === 'undefined' ? true : obj['showLabel'],
        fileName = (obj['fileName'] || 'UserExport') + '.csv',
        columns = obj['columns'] || {
            title: [],
            key: [],
            formatter: undefined
        };
    var ShowLabel = typeof ShowLabel === 'undefined' ? true : ShowLabel;
    var row = "", CSV = '', key;
    // 如果要现实表头文字
    if (ShowLabel) {
        // 如果有传入自定义的表头文字
        if (columns.title.length) {
            columns.title.map(function(n) {
                row += n + ',';
            });
        } else {
            // 如果没有，就直接取数据第一条的对象的属性
            for (key in data[0]) row += key + ',';
        }
        row = row.slice(0, -1); // 删除最后一个,号，即a,b, => a,b
        CSV += row + '\r\n'; // 添加换行符号
    }
    // 具体的数据处理
    data.map(function(n) {
        row = '';
        // 如果存在自定义key值
        if (columns.key.length) {
            columns.key.map(function(m) {
                row += '"' + (typeof columns.formatter === 'function' ? columns.formatter(m, n[m]) || n[m] : n[m]) + '",';
            });
        } else {
            for (key in n) {
                row += '"' + (typeof columns.formatter === 'function' ? columns.formatter(key, n[key]) || n[key] : n[key]) + '",';
            }
        }
        row.slice(0, row.length - 1); // 删除最后一个,
        CSV += row + '\r\n'; // 添加换行符号
    });
    if(!CSV) return;
    this.SaveAs(fileName, CSV);
  },
  SaveAs: function(fileName, csvData) {
    // console.log(fileName, csvData);
  }
};
```
然后我们分别测试了如下数据：

```
JSonToCSV.setDataConver({
  data: [
    {name: '张三', amont: '323433.56', proportion: 33.4},
    {name: '李四', amont: '545234.43', proportion: 55.45}
  ],
  fileName: 'test',
  columns: {
    title: ['姓名', '金额', '占比'],
    key: ['name', 'amont', 'proportion'],
    formatter: function(n, v) {
      if(n === 'amont' && !isNaN(Number(v))) {
        v = v + '';
        v = v.split('.');
        v[0] = v[0].replace(/(\d)(?=(?:\d{3})+$)/g, '$1,'); // 千分位的设置
         return v.join('.');
      }
      if(n === 'proportion') return v + '%';
    }
  }
});
```
到此，数据转换完毕
### 下载方式
由于浏览器之间的差异，尤其是IE，所以不同的浏览器下载的方式也不一样，如Chrome和Firefox都支持`a`标签设置download属性和href值，然后调用`a`的`click`方法即可下载，IE既不支持`a`download属性也不允许调用`a`的`click`方法。代码如下：

```
var a = document.querySelector('a');
a.click(); // 在这里 IE是拒绝执行的，会提示权限问题
```
那么对于支持a的download属性的，直接设置download属性值和href值，具体代码如下：
####Chrome、Firefox等浏览器的的下载方式
```
SaveAs: function(fileName, csvData) {
    var bw = this.browser();
    if(!bw['edge'] ||  !bw['ie']) {
      var alink = document.createElement("a");
      alink.id = "linkDwnldLink";
      alink.href = this.getDownloadUrl(csvData);
      document.body.appendChild(alink);
      var linkDom = document.getElementById('linkDwnldLink');
      linkDom.setAttribute('download', fileName);
      linkDom.click();
      document.body.removeChild(linkDom);
    }
  },
  getDownloadUrl: function(csvData) {
    var _utf = "\uFEFF"; // 为了使Excel以utf-8的编码模式，同时也是解决中文乱码的问题
    return 'data:attachment/csv;charset=utf-8,' + _utf + encodeURIComponent(csvData);
  },
  browser: function() {
    var Sys = {};
    var ua = navigator.userAgent.toLowerCase();
    var s;
    (s = ua.indexOf('edge') !== - 1 ? Sys.edge = 'edge' : ua.match(/rv:([\d.]+)\) like gecko/)) ? Sys.ie = s[1]:
        (s = ua.match(/msie ([\d.]+)/)) ? Sys.ie = s[1] :
        (s = ua.match(/firefox\/([\d.]+)/)) ? Sys.firefox = s[1] :
        (s = ua.match(/chrome\/([\d.]+)/)) ? Sys.chrome = s[1] :
        (s = ua.match(/opera.([\d.]+)/)) ? Sys.opera = s[1] :
        (s = ua.match(/version\/([\d.]+).*safari/)) ? Sys.safari = s[1] : 0;
    return Sys;
  }
```
虽然看起来是可以了，但还是有问题。什么问题呢？
就是当数据量大的时候，比如几千条甚至几万条，在数据转换的时候，href的数值自然也就长了。若是超过浏览器自身限制的最大长度，会导致下载失败。具体每个浏览器之前URL最大长度限制如下（HTTP协议并没有限制URL的长度）：

| 浏览器  | 最大长度（字符数）     | 备注  |
| ---- |:---------:| -----:|
| IE | 2083 | 如果超过这个数字，提交按钮没有任何反应 |
| Firefox | 65,536 | - |
| Chrome | 8,182 | - |
| Safari | 80,000 | - |
| Opera | 190,000 | - |


所以我们这里借助 Blob（[Blob传送门][2]）来将转换好的数据进行处理，代码如下：

```
getDownloadUrl: function(csvData) {
    var _utf = "\uFEFF"; // 为了使Excel以utf-8的编码模式，同时也是解决中文乱码的问题
    if (window.Blob && window.URL && window.URL.createObjectURL) {
        var csvData = new Blob([_utf + csvData], {
            type: 'text/csv'
        });
        return URL.createObjectURL(csvData);
    }
    // return 'data:attachment/csv;charset=utf-8,' + _utf + encodeURIComponent(csvData);
  }
```
我们在查看href值为：`blob:http://127.0.0.1:3000/9715ca8a-bb9a-4b0c-8546-9bd13e8f0b69`。
这样不管几万条还是几十万条数据都可以下载的
这里涉及到的知识点：[encodeURIComponent][3]、[URL.createObjectURL][4]
到这里，Chrome、Firefox等浏览器解决了。
#### IE10~Edge浏览的下载方式
IE10到Edge等浏览器调用`windows.navigator.msSaveBlob`实现保存文件，[msSaveBlob][5]是IE10~Edge的私有方法。
所以`SaveAs`代码改写如下：

```
SaveAs: function(fileName, csvData) {
    var bw = this.browser();
    if(!bw['edge'] || !bw['ie']) {
      var alink = document.createElement("a");
      alink.id = "linkDwnldLink";
      alink.href = this.getDownloadUrl(csvData);
      document.body.appendChild(alink);
      var linkDom = document.getElementById('linkDwnldLink');
      linkDom.setAttribute('download', fileName);
      linkDom.click();
      document.body.removeChild(linkDom);
    }
    else if(bw['ie'] >= 10 || bw['edge'] == 'edge') {
      var _utf = "\uFEFF";
      var _csvData = new Blob([_utf + csvData], {
          type: 'text/csv'
      });
      navigator.msSaveBlob(_csvData, fileName);
    }
  }
```
####IE9下载方式
IE9使用[execCommand][6]方法来保存csv文件，`SaveAs`改写如下：

```
SaveAs: function(fileName, csvData) {
    var bw = this.browser();
    if(!bw['edge'] || !bw['ie']) {
      var alink = document.createElement("a");
      alink.id = "linkDwnldLink";
      alink.href = this.getDownloadUrl(csvData);
      document.body.appendChild(alink);
      var linkDom = document.getElementById('linkDwnldLink');
      linkDom.setAttribute('download', fileName);
      linkDom.click();
      document.body.removeChild(linkDom);
    }
    else if(bw['ie'] >= 10 || bw['edge'] == 'edge') {
      var _utf = "\uFEFF";
      var _csvData = new Blob([_utf + csvData], {
          type: 'text/csv'
      });
      navigator.msSaveBlob(_csvData, fileName);
    }
    else {
      var oWin = window.top.open("about:blank", "_blank");
      oWin.document.write('sep=,\r\n' + csvData);
      oWin.document.close();
      oWin.document.execCommand('SaveAs', true, fileName);
      oWin.close();
    }
  }
```
所以最终代码整体如下：

```
var JSonToCSV = {
  /*
   * obj是一个对象，其中包含有：
   * ## data 是导出的具体数据
   * ## fileName 是导出时保存的文件名称 是string格式
   * ## showLabel 表示是否显示表头 默认显示 是布尔格式
   * ## columns 是表头对象，且title和key必须一一对应，包含有
        title:[], // 表头展示的文字
        key:[], // 获取数据的Key
        formatter: function() // 自定义设置当前数据的 传入(key, value)
   */
  setDataConver: function(obj) {
    var bw = this.browser();
    if(bw['ie'] < 9) return; // IE9以下的
    var data = obj['data'],
        ShowLabel = typeof obj['showLabel'] === 'undefined' ? true : obj['showLabel'],
        fileName = (obj['fileName'] || 'UserExport') + '.csv',
        columns = obj['columns'] || {
            title: [],
            key: [],
            formatter: undefined
        };
    var ShowLabel = typeof ShowLabel === 'undefined' ? true : ShowLabel;
    var row = "", CSV = '', key;
    // 如果要现实表头文字
    if (ShowLabel) {
        // 如果有传入自定义的表头文字
        if (columns.title.length) {
            columns.title.map(function(n) {
                row += n + ',';
            });
        } else {
            // 如果没有，就直接取数据第一条的对象的属性
            for (key in data[0]) row += key + ',';
        }
        row = row.slice(0, -1); // 删除最后一个,号，即a,b, => a,b
        CSV += row + '\r\n'; // 添加换行符号
    }
    // 具体的数据处理
    data.map(function(n) {
        row = '';
        // 如果存在自定义key值
        if (columns.key.length) {
            columns.key.map(function(m) {
                row += '"' + (typeof columns.formatter === 'function' ? columns.formatter(m, n[m]) || n[m] : n[m]) + '",';
            });
        } else {
            for (key in n) {
                row += '"' + (typeof columns.formatter === 'function' ? columns.formatter(key, n[key]) || n[key] : n[key]) + '",';
            }
        }
        row.slice(0, row.length - 1); // 删除最后一个,
        CSV += row + '\r\n'; // 添加换行符号
    });
    if(!CSV) return;
    this.SaveAs(fileName, CSV);
  },
  SaveAs: function(fileName, csvData) {
    var bw = this.browser();
    if(!bw['edge'] || !bw['ie']) {
      var alink = document.createElement("a");
      alink.id = "linkDwnldLink";
      alink.href = this.getDownloadUrl(csvData);
      document.body.appendChild(alink);
      var linkDom = document.getElementById('linkDwnldLink');
      linkDom.setAttribute('download', fileName);
      linkDom.click();
      document.body.removeChild(linkDom);
    }
    else if(bw['ie'] >= 10 || bw['edge'] == 'edge') {
      var _utf = "\uFEFF";
      var _csvData = new Blob([_utf + csvData], {
          type: 'text/csv'
      });
      navigator.msSaveBlob(_csvData, fileName);
    }
    else {
      var oWin = window.top.open("about:blank", "_blank");
      oWin.document.write('sep=,\r\n' + csvData);
      oWin.document.close();
      oWin.document.execCommand('SaveAs', true, fileName);
      oWin.close();
    }
  },
  getDownloadUrl: function(csvData) {
    var _utf = "\uFEFF"; // 为了使Excel以utf-8的编码模式，同时也是解决中文乱码的问题
    if (window.Blob && window.URL && window.URL.createObjectURL) {
        var csvData = new Blob([_utf + csvData], {
            type: 'text/csv'
        });
        return URL.createObjectURL(csvData);
    }
    // return 'data:attachment/csv;charset=utf-8,' + _utf + encodeURIComponent(csvData);
  },
  browser: function() {
    var Sys = {};
    var ua = navigator.userAgent.toLowerCase();
    var s;
    (s = ua.indexOf('edge') !== - 1 ? Sys.edge = 'edge' : ua.match(/rv:([\d.]+)\) like gecko/)) ? Sys.ie = s[1]:
        (s = ua.match(/msie ([\d.]+)/)) ? Sys.ie = s[1] :
        (s = ua.match(/firefox\/([\d.]+)/)) ? Sys.firefox = s[1] :
        (s = ua.match(/chrome\/([\d.]+)/)) ? Sys.chrome = s[1] :
        (s = ua.match(/opera.([\d.]+)/)) ? Sys.opera = s[1] :
        (s = ua.match(/version\/([\d.]+).*safari/)) ? Sys.safari = s[1] : 0;
    return Sys;
  }
};
// 测试
JSonToCSV.setDataConver({
  data: [
    {name: '张三', amont: '323433.56', proportion: 33.4},
    {name: '李四', amont: '545234.43', proportion: 55.45}
  ],
  fileName: 'test',
  columns: {
    title: ['姓名', '金额', '占比'],
    key: ['name', 'amont', 'proportion'],
    formatter: function(n, v) {
      if(n === 'amont' && !isNaN(Number(v))) {
        v = v + '';
        v = v.split('.');
        v[0] = v[0].replace(/(\d)(?=(?:\d{3})+$)/g, '$1,');
         return v.join('.');
      }
      if(n === 'proportion') return v + '%';
    }
  }
});
```
也可以访问我的[GitHub][7]下载最新的js文件


  [1]: test11.png
  [2]: https://developer.mozilla.org/zh-CN/docs/Web/API/Blob
  [3]: https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/encodeURIComponent
  [4]: https://developer.mozilla.org/zh-CN/docs/Web/API/URL/createObjectURL
  [5]: https://msdn.microsoft.com/zh-CN/library/hh779016%28v=vs.85%29.aspx#
  [6]: https://developer.mozilla.org/zh-CN/docs/Web/API/Document/execCommand
  [7]: https://github.com/liqingzheng/pc/blob/master/JsonExportToCSV.js
