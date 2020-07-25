<template>
  <div id="app">
    <el-container>
      <el-main>
        <div id="menu" :class="menuFixed == true ? 'isFixed' :''">
          <el-row :gutter="10"t>
            <el-col :lg="16">
              <el-button round size="small" icon="el-icon-arrow-left" @click="goBack">返回</el-button>
              <el-button-group>
                <el-button plain size="small" icon="el-icon-top" @click="move('top')">置顶</el-button>
                <el-button  plain size="small" icon="el-icon-arrow-up" @click="move('up')">上移</el-button>
                <el-button plain size="small" icon="el-icon-arrow-down" @click="move('down')">下移</el-button>
                <el-button plain size="small" icon="el-icon-bottom" @click="move('bottom')">置底</el-button>
              </el-button-group>
              <el-dropdown @command="sortCommand">
                <el-button plain size="small">
                  排序<i class="el-icon-arrow-down el-icon--right"></i>
                </el-button>
                <el-dropdown-menu slot="dropdown">
                  <el-tooltip class="item" effect="dark" content="按标题中的(左到右)数字排序" placement="right">
                    <el-dropdown-item command="number">数字<i class="el-icon-right"></i></el-dropdown-item>
                  </el-tooltip>
                  <el-tooltip class="item" effect="dark" content="按标题中的(右到左)数字排序" placement="right">
                    <el-dropdown-item command="reverseNumber">数字<i class="el-icon-back"></i></el-dropdown-item>
                  </el-tooltip>
                  <el-tooltip class="item" effect="dark" content="按标题的文本排序" placement="right">
                    <el-dropdown-item command="text">文本</el-dropdown-item>
                  </el-tooltip>
                  <el-tooltip class="item" effect="dark" content="按标题前缀文本排序" placement="right">
                    <el-dropdown-item command="prefix">前缀</el-dropdown-item>
                  </el-tooltip>
                  <el-tooltip class="item" effect="dark" content="按标题后缀文本排序" placement="right">
                    <el-dropdown-item command="subfix">后缀</el-dropdown-item>
                  </el-tooltip>
                  <el-tooltip class="item" effect="dark" content="按文件时间排序" placement="right">
                    <el-dropdown-item command="time">时间</el-dropdown-item>
                  </el-tooltip>
                  <el-tooltip class="item" effect="dark" content="按文件类型(纸质在前,电子在后)排序" placement="right">
                    <el-dropdown-item command="paperFirst">纸电</el-dropdown-item>
                  </el-tooltip>
                </el-dropdown-menu>
              </el-dropdown>

              <el-button plain size="small" icon="el-icon-finished" @click="clearSelected">清除选中</el-button>
              <el-button plain size="small" type="warning" icon="el-icon-refresh-left" @click="undo" :disabled="undoable">撤销</el-button>
              <el-button size="small" type="primary" icon="el-icon-circle-check" v-loading.fullscreen.lock="fullscreenLoading" @click="save">保存</el-button>
            </el-col>
            <el-col :lg="8">
              <el-input
                placeholder="查找文件"
                suffix-icon="el-icon-search"
                size="small"
                v-model="keywords"
                @blur="selectAll(keywords)"
                @keyup.enter.native="selectAll(keywords)"
                >
              </el-input>
            </el-col>
          </el-row>
        </div>

        <el-row :gutter="20">
          <el-col :lg="listSpan">
            <div id="list">
              <div v-for="(file,index) in files"
                v-bind:key="file.id"
                v-on:click="toggleCheck(index)"
                :index="index"
                :id="file.id"
                :class="{checked:file.checked}">
                <el-tooltip class="item" effect="dark" :content="fileTip(file)" placement="top-end">
                  <el-alert
                    type="info"

                    :closable="false">
                    <i :class="iconSelect(file.type)"></i> {{file.title}}
                  </el-alert>
                </el-tooltip>
              </div>
            </div>
          </el-col>
          <el-col :lg="6" v-show="debug">
            <pre>
{{files}}
            </pre>
          </el-col>
        </el-row>

      </el-main>
    </el-container>


    <el-dialog
      title="提交内容"
      :visible.sync="dialogVisible">
      <pre>
{{submitResult}}
      </pre>
      <span slot="footer" class="dialog-footer">
        <el-button type="primary" @click="dialogVisible = false">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
const DEBUG = false; //调试模式使用测试数据,不提交保存
const GET_ITEMS_URL = location.href; //初始化数据接口
const SET_ITEMS_URL = location.href; //设置数据排序接口
let BACK_URL = ''; //返回链接
if (BACK_URL == '') {
  BACK_URL = document.referrer;
}

import { Sortable, MultiDrag } from 'sortablejs';
Sortable.mount(new MultiDrag());

import axios from 'axios';
import Qs from 'qs';
const Axios = axios.create({
  baseURI: '/',
  headers: {
    'X-Requested-With': 'XMLHttpRequest',
    'Content-Type':'application/x-www-form-urlencoded;charset=UTF-8',
  },
  transformRequest: [function (data) {
    const d = Qs.stringify(data)
    return d;
  }]
});

export default {
  data(){
    return {
      debug:DEBUG,
      submitResult:[], //提交结果
      dialogVisible: false, //提交结果提示窗显示状态
      fullscreenLoading:true, //加载遮罩
      keywords:"", //关键词
      history:[], //历史记录
      historyIndex:0, //历史记录索引
      checked:[], //选中条目
      unchecked:[], //未选中条目
      checkChange:false, //选中状态
      topClick:false, //置顶点击
      bottomClick:false, //置底点击
      menuFixed:false, //菜单固定标识
      lastClickIndex:-1, //最后点击索引
      shiftPress:false, //shift键是否按住
      filesIdMap:{},//文件id索引
      files:[
        /*测试数据*/
        {
          id:1,
          title:'111.edc',
          type:1,
          time:"2018-01-02 08:30:00",
          checked:0
        },
        {
          id:4,
          title:'222.edc',
          type:2,
          time:"2018-06-02 08:30:00",
          checked:0
        },
        {
          id:6,
          title:'333.edc',
          type:1,
          time:"2018-01-02 07:00:00",
          checked:0
        },
        {
          id:9,
          title:'444.edc',
          type:2,
          time:"2018-01-01 08:30:00",
          checked:1
        },
      ]
    }
  },
  mounted () {
    //初始化
    if(!DEBUG){
      //获取数据
      this.init();
    } else {
      this.initData();
    }

    var _this = this;
    var $ul = document.getElementById('list');
    new Sortable($ul, {
      multiDrag: true,
      selectedClass: 'selected',
      fallbackTolerance: 3,
      animation: 150,
      forceFallback: true,
      onSelect: function(evt){
        // console.log(evt.item);
        _this.checkItem(evt.item);
        evt.item;
      },
      onDeselect: function(evt) {
        _this.uncheckItem(evt.item);
        evt.item;
      },
      onUpdate: function(evt){
        //ui更新后数组排序需要同步更新
        evt;
        let result = [];
        let items = $ul.childNodes;
        for (let i = 0; i < items.length; i++) {
          const index = items[i].getAttribute('index');
          // console.log(index);
          result.push(_this.files[index]);
        }
        _this.updateFilesSort(result);
      }
    });

    // 这个是获取键盘按住事件
    window.addEventListener('keydown', code => {
      if (code.keyCode === 16 && code.shiftKey) {
        // 判断是否按住shift键
        _this.shiftPress = true;
      } else if (code.keyCode === 65 && code.ctrlKey) {
        // 全选操作CTRL+A
        window.event.preventDefault() //关闭浏览器快捷键
        _this.selectAll('all');
      }
    });

    // 这个是获取键盘松开事件
    window.addEventListener('keyup', code => {
      if(code.keyCode === 16){
        // 判断是否松开shift键
        _this.shiftPress = false;
      }
    });

    //监听滚动条
    window.addEventListener('scroll', this.handleScroll);

  },
  methods: {
    //返回
    goBack:function(){
      window.location.href = BACK_URL;
    },
    //监听滚动条
    handleScroll: function() {
      var scrollTop = window.pageYOffset || document.documentElement.scrollTop || document.body.scrollTop;
      var offsetTop = document.querySelector('#menu').offsetTop;
      if (scrollTop > offsetTop) {
        this.menuFixed = true;
      } else {
        this.menuFixed = false;
      }
    },
    //分配icon图标
    iconSelect:function(fileType){
      if(fileType == 1){ //电子原件
        return 'el-icon-tickets';
      }else{ //纸质
        return 'el-icon-document';
      }
    },
    //文件提示
    fileTip: function(file){
      let tip = '';
      let fileTypeLabel = {
        '0':'未知类型',
        '1':'电子原件',
        '2':'纸质原件',
        '3':'复印件',
      }
      let fileType = fileTypeLabel[file.type.toString()] || '未知类型';
      tip = '文件类型:'+ fileType +' 时间:'+file.time;
      return tip;
    },
    //生成文件id映射数据
    createFilesIdMap: function(files){
      this.filesIdMap = {};
      for (var i = files.length - 1; i >= 0; i--) {
        this.filesIdMap[files[i].id] = files[i];
      }
    },
    //根据关键词选中,默认全选,关键词空格分隔
    selectAll: function(keywords){
      let keys = [];
      if(keywords == ''){
        return;
      }
      if(keywords != 'all'){
        keys = keywords.split(' ');
      }

      let checkedCount = 0;
      this.files.forEach(function(file){
        let checked = 1;
        for (let i = 0; i < keys.length; i++) {
          const key = keys[i];
          if(file.title.indexOf(key) == -1){
            checked = 0;
            break;
          }
        }
        file.checked = checked;
        if(checked>0) checkedCount++;
      });

      if(checkedCount <= 0){
        this.$message({
          showClose:true,
          duration:1000,
          message:'未找到相应目标'
        });
      } else {
        this.checkChange = true;
      }
    },
    //选中动作
    toggleCheck: function(index){
      console.log(this.lastClickIndex);
      if(this.shiftPress && this.lastClickIndex >= 0){
        let minIndex = Math.min(index,this.lastClickIndex);
        let maxIndex = Math.max(index,this.lastClickIndex);
        this.clearSelected(false);
        for (var i = this.files.length - 1; i >= 0; i--) {
          if(i >= minIndex && i <= maxIndex){
            this.files[i].checked = 1;
          }
        }
      } else {
        this.files[index].checked = Math.abs(this.files[index].checked - 1);
        if(this.files[index].checked){
          this.lastClickIndex = index;
        } else {
          this.lastClickIndex = -1;
        }
      }
      this.checkChange = true;
    },
    //选中元素
    checkItem: function(e){
      // let index = e.getAttribute('index');
      // this.files[index].checked = 1;
      // this.checkChange = true;
    },
    //取消选中元素
    uncheckItem: function(e){
      // let index = e.getAttribute('index');
      // this.files[index].checked = 0;
      // this.checkChange = true;
    },
    //清除选中
    clearSelected:function(clearLastIndex){
      if(clearLastIndex == undefined){
        clearLastIndex = true;
      }
      this.keywords = "";
      this.files.forEach(function(file){
        file.checked = 0;
      });
      this.checkChange = true;
      if(clearLastIndex){
        this.lastClickIndex = -1;
      }
    },
    //更新排序
    updateFilesSort: function(result){
      this.files = result;
      this.historyIndex++;
      this.history[this.historyIndex] = this.arrayColumn(result,'id');
      this.topClick = false;
      this.bottomClick = false;
    },
    //撤销
    undo: function(){
      this.historyIndex = Math.max(0,this.historyIndex-1);
      this.files = this.historyFiles(this.history[this.historyIndex]);
      this.checkChange = true;
    },
    //历史数据根据记录里的id重新排序
    historyFiles: function(idList){
      let _this = this;
      return idList.map(function(v,i){
        return _this.filesIdMap[v];
      });
    },
    //数组上移
    moveUpSort:function(list){
      let change = false;
      for (let i = 0; i < list.length; i++) {
        if(i == 0) continue;
        const item = list[i];
        if(item.checked == 0) continue;
        const prevItem = list[i-1];
        if(prevItem.checked == 1) continue;

        //交换位置
        list[i-1] = item;
        list[i] = prevItem;
        change = true;
      }
      if(change){
        return list;
      }
      return change;
    },
    //移动操作
    move:function(type){
      let result;
      let newResult;
      switch (type) {
        case 'up': //上移
          result = this.deepCopy(this.files);
          newResult = this.moveUpSort(result);
          break;
        case 'down': //下移
          result = this.deepCopy(this.files);
          newResult = this.moveUpSort(result.reverse());
          if(newResult) newResult = newResult.reverse();
          break;
        case 'top': //置顶
          if(!this.checkChange && this.topClick) return;
          this.getCheckedItems(true);
          if(this.checked.length <= 0) return;
          newResult = this.deepCopy(this.checked)
          newResult.push.apply(newResult,this.unchecked);
          this.updateFilesSort(newResult);
          this.topClick = true;
          return;
          break;
        case 'bottom': //置底
          if(!this.checkChange && this.bottomClick) return;
          this.getCheckedItems(true);
          if(this.checked.length <= 0) return;
          newResult = this.deepCopy(this.unchecked)
          newResult.push.apply(newResult,this.checked);
          this.updateFilesSort(newResult);
          this.bottomClick = true;
          return;
          break;

        default:
          return this.$message({
            showClose:true,
            type:'error',
            message:"不支持的移动类型:"+type
          });
          break;
      }

      if(!newResult) {
        this.$message({
          showClose: true,
          duration: 1000,
          message: '中国移动,移不动了'
        });
        return;
      }
      this.updateFilesSort(newResult);
    },
    //纸质文件->电子文件
    sortPaperElectron:function(paperFirst){
      let paperItems = [];
      let electronItems = [];
      let result = [];
      paperFirst = paperFirst || true;
      for (let i = 0; i < this.files.length; i++) {
        const file = this.files[i];
        if(file.type == 1){
          electronItems.push(file);
        } else{
          paperItems.push(file);
        }
      }

      if(paperFirst){
        paperItems.push.apply(paperItems,electronItems);
        result = paperItems;
      } else {
        electronItems.push.apply(electronItems,paperItems);
        result = electronItems;
      }

      this.updateFilesSort(result);
    },
    //中文数字转阿拉伯数字
    cnNumberReplace:function(title){
      let cnNumChar = {
        '零':0,
        '一':1,
        '二':2,
        '三':3,
        '四':4,
        '五':5,
        '六':6,
        '七':7,
        '八':8,
        '九':9
      };
      for(let cn in cnNumChar){
        title = title.replace(cn,cnNumChar[cn]);
      }
      return title;
    },
    //排序命令
    sortCommand:function(type){
      switch (type) {
        case 'prefix':
          this.$prompt('请输入前缀长度', '提示', {
            confirmButtonText: '确定',
            cancelButtonText: '取消',
            inputPattern: /\d+/,
            inputValue:8,
            inputErrorMessage: '只能输入大于0的数字'
          }).then(({ value }) => {
            this.sortItems(type,1,value);
          }).catch(function(){
            console.log('取消排序');
          });
          break;
        case 'subfix':
          this.$prompt('请输入后缀长度', '提示', {
            confirmButtonText: '确定',
            cancelButtonText: '取消',
            inputPattern: /\d+/,
            inputValue:8,
            inputErrorMessage: '只能输入大于0的数字'
          }).then(({ value }) => {
            this.sortItems(type,1,value);
          }).catch(function(){
            console.log('取消排序');
          });
          break;
        case 'paperFirst':
          this.sortPaperElectron();
          break;
        default:
          return this.sortItems(type);
          break;
      }
    },
    //排序操作
    sortItems:function(type,sort,fixLength){
      console.log(type,sort,fixLength);
      this.getCheckedItems(true);
      if(this.checked.length <= 0) return;
      let filesDict = {};
      let sortKeys = [];
      let key = '';
      let result = [];
      let newResult = [];
      sort = sort || 1;
      fixLength = fixLength || 0;
      console.log('fixLength:',fixLength);
      for (let i = 0; i < this.checked.length; i++) {
        const file = this.checked[i];
        switch (type) {
          case 'number':
            key = file.title;
            key = this.cnNumberReplace(key);
            key = key.replace(/\D/g,'');
            key = parseInt(key);
            break;
          case 'reverseNumber':
            key = file.title;
            key = this.cnNumberReplace(key);
            key = this.reverseNumber(key);
            key = parseInt(key);
            break;
          case 'text':
            key = file.title;
            break;
          case 'time':
            key = file.time;
            break;
          case 'prefix':
            key = file.title;
            key = key.slice(0,fixLength);
            console.log('prefix:',key);
            key = key.replace(/\D/g,0);
            if(key == '') key = 0;
            key = parseInt(key);
            break;
          case 'subfix':
            key = file.title;
            key = key.substr(key.length-fixLength);
            console.log('prefix:',key);
            key = key.replace(/\D/g,0);
            if(key == '') key = 0;
            key = parseInt(key);
            break;

          default:
            return this.$message({
              showClose:true,
              type:'error',
              message:"不支持的移动类型:"+type
            });
            break;
        }

        sortKeys.push(key);
        if(!(key in filesDict)){
          filesDict[key] = [];
        }
        filesDict[key].push(file);
      }

      // console.log('before sortKeys:',sortKeys);
      // console.log('filesDict:',filesDict);
      sortKeys = this.unique(sortKeys); //索引值去重
      if(type == 'text'){
        sortKeys.sort();
      } else {
        sortKeys.sort(function(a,b){
          return a-b;
        });
      }

      if(sort == -1) sortKeys.reverse()

      // console.log('after sortKeys:',sortKeys);
      for (let i = 0; i < sortKeys.length; i++) {
        const key = sortKeys[i];
        filesDict[key].forEach(function(file){
          result.push(file);
        });
      }
      // console.log('result:',result)

      let appendFlag = true;
      for (let i = 0; i < this.files.length; i++) {
        const file = this.files[i];
        if(file.checked == 1 && appendFlag){
          for (let j = 0; j < result.length; j++) {
            const checkedFile = result[j];
            newResult.push(checkedFile);
          }
          appendFlag = false;
        }
        if(file.checked == 0) newResult.push(file);
      }

      this.updateFilesSort(newResult);
    },
    //获取数组字段
    arrayColumn: function(array, columnName) {
      if (array !== null) {
        if (Array.isArray(array)) {
          return array.map(function (value, index) {
            return value[columnName];
          });
        } else if (typeof array === 'object') {
          var newarray = []
          for (var key in array) {
            newarray.push(array[key][columnName]);
          }
          return newarray;
        }
      }
    },
    //数组去重
    unique: function(arr){
      let res = [];
      for (let i = 0; i < arr.length; i++) {
        if(res.indexOf(arr[i]) === -1){
          res.push(arr[i]);
        }
      }
      return res;
    },
    //文本中数字逆序
    reverseNumber: function(key){
      let reg = /\d+/g;
      let res = [];
      let arr = reg.exec(key)
      while(arr){
          res.push(arr[0]);
          arr = reg.exec(key);
      }
      let result = res.reverse().join('');
      return result;
    },
    //获取初始数据
    init: function(){
      var _this = this;
      Axios.get(GET_ITEMS_URL).then(function(res){
        if(res.data.code != 0){
          _this.$alert(res.data.msg);
          return;
        }
        _this.files = [];
        for (let i = 0; i < res.data.data.length; i++) {
          const file = res.data.data[i];
          file.checked = 0;
          _this.files.push(file);
        }

        _this.initData();
      }).catch(function(err){
        _this.$alert('初始化失败','错误',{
          confirmButtonText: '确定',
          callback: function(){
            _this.goBack();
          }
        });
      });
    },
    //数据初始化
    initData: function(){
      this.fullscreenLoading = false;
      this.history[this.historyIndex] = this.arrayColumn(this.files,'id');
      this.createFilesIdMap(this.files);
    },
    //保存结果
    save: function(){
      let _this = this;
      let sortResult = [];
      for (let i = 0; i < this.files.length; i++) {
        const file = this.files[i];
        sortResult.push({
          'id':file.id,
          'sno':i+1
        });
      }
      this.fullscreenLoading = true;
      if(DEBUG){
        this.submitResult = sortResult;
        this.dialogVisible = true;
        this.fullscreenLoading = false;
        return;
      }
      Axios.post(SET_ITEMS_URL,{
        data:sortResult
      }).then(function(res){
        if(res.data.code != 0){
            _this.$alert(res.data.msg,'错误',{
              confirmButtonText: '确定',
              callback: function(action){
                console.log(err);
              }
            });
            return;
        }
        _this.$confirm(res.data.msg, '成功', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'success'
        }).then(function(){
          _this.goBack();
        }).catch(function(){
          console.log('取消跳转,停留在当前页');
        });
        _this.fullscreenLoading = false;
      }).catch(function(err){
        _this.fullscreenLoading = false;
        _this.$alert('提交失败', '错误', {
          confirmButtonText: '确定',
          callback: function(action){
            console.log(err);
          }
        });
      });
    },
    //验证是否选中
    getCheckedItems: function(tip){
      // if(!this.checkChange) return;
      this.checked = [];
      this.unchecked = [];
      for (let i = 0; i < this.files.length; i++) {
        const file = this.files[i];
        if(file.checked){
          this.checked.push(file);
        } else {
          this.unchecked.push(file);
        }
      }
      this.checkChange = false;
      if(tip && this.checked.length == 0){
        this.$message({
          showClose:true,
          duration:2000,
          message:"请先选中条目"
        });
      }
    },
    //数组深拷贝
    deepCopy:function(arr){
      return arr.map((e)=>{
        if(typeof e==='object'){
          return Object.assign({},e);
        }else{
          return e;
        }
      });
    }
  },
  computed:{
    undoable: function(){
      return this.historyIndex > 0 ? false : true;
    },
    listSpan: function(){
      if(DEBUG) return 18;
      return 24;
    }

  }
}
</script>

<style>
  .el-alert {
    margin:10px;
  }
  .checked .el-alert {
    background-color: #d9ecff;
    color: #409EFF;
    border: 1px solid #409EFF;
  }
  .checked .el-alert .el-alert__description {
    font-weight: bold;
    color: #409EFF;
  }

  .sortable-ghost {
    background-color: #fde2e2
  }

  .el-alert .el-alert__description {
    color:#606266;
    font-size: 14px;
    font-weight:bold;
    -moz-user-select: none;
    -webkit-user-select: none;
    -ms-user-select: none;
    user-select: none;
  }

  pre {
    display: block;
    padding: 9.5px;
    margin: 0 0 10px;
    font-size: 13px;
    line-height: 1.42857143;
    color: #333;
    word-break: break-all;
    word-wrap: break-word;
    background-color: #f5f5f5;
    border: 1px solid #ccc;
    border-radius: 4px;
  }

  .isFixed{
    position:fixed;
    background-color:#Fff;
    top:0;
    z-index:999;
    width:98%;
    padding:10px;
  }
</style>
