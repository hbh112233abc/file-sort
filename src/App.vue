<template>
  <div id="app">
    <el-container>
      <el-main>
        <el-row :gutter="10">
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
                <el-dropdown-item command="number">数字<i class="el-icon-right"></i></el-dropdown-item>
                <el-dropdown-item command="reverseNumber">数字<i class="el-icon-back"></i></el-dropdown-item>
                <el-dropdown-item command="text">文本</el-dropdown-item>
                <el-dropdown-item command="time">时间</el-dropdown-item>
                <el-dropdown-item command="paperFirst">纸电</el-dropdown-item>
                <el-dropdown-item command="prefix">前缀</el-dropdown-item>
                <el-dropdown-item command="subfix">后缀</el-dropdown-item>
              </el-dropdown-menu>
            </el-dropdown>

            <el-button plain size="small" icon="el-icon-finished" @click="clearSeleced">清除选中</el-button>
            <el-button plain size="small" type="warning" icon="el-icon-refresh-left" @click="undo" :disabled="undoable">撤销 {{historyIndex}}</el-button>
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

        <el-row :gutter="20">
          <el-col :lg="listSpan">
            <div id="list">
              <div v-for="(file,index) in files" v-bind:key="file.id" :index="index" :id="file.id" class="" :class="{checked:file.checked}">
                <el-alert
                  type="info"
                  :closable="false">
                  <i :class="iconSelect(file.type)"></i> {{file.title}}
                </el-alert>
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
const DEBUG = true; //调试模式使用测试数据,不提交保存
const GET_ITEMS_URL = '__SELF__'; //初始化数据接口
const SET_ITEMS_URL = '__SELF__'; //设置数据排序接口

import { Sortable, MultiDrag } from 'sortablejs';
Sortable.mount(new MultiDrag());

import axios from 'axios';
const Axios = axios.create({
  baseURI: '/',
  headers: {'X-Requested-With': 'XMLHttpRequest'},
  transformRequest: [function (data) {
    const d = Qs.stringify(data)
    return d;
  }]
});

export default {
  data(){
    return {
      selected:[],
      debug:DEBUG,
      submitResult:[], //提交结果
      dialogVisible: false, //提交结果提示窗显示状态
      fullscreenLoading:false, //加载遮罩
      keywords:"", //关键词
      history:[], //历史记录
      historyIndex:0, //历史记录索引
      checked:[], //选中条目
      unchecked:[], //未选中条目
      checkChange:false, //选中状态
      topClick:false, //置顶点击
      bottomClick:false, //置底点击
      files:[
        /* 测试数据 */
        {
          id:1,
          title:'111.edc',
          type:1,
          time:"2018-01-02 08:30:00",
          checked:0
        },
        {
          id:2,
          title:'222.edc',
          type:2,
          time:"2018-06-02 08:30:00",
          checked:0
        },
        {
          id:3,
          title:'333.edc',
          type:1,
          time:"2018-01-02 07:00:00",
          checked:0
        },
        {
          id:4,
          title:'444.edc',
          type:2,
          time:"2018-01-01 08:30:00",
          checked:0
        },
      ]
    }
  },
  mounted () {
    //初始化
    var _this = this;

    if(!DEBUG){
      //获取数据
      this.init();
    } else {
      _this.fullscreenLoading = false;
      _this.history.push(_this.files); //初始版本
    }

    var $ul = document.getElementById('list');
    new Sortable($ul, {
      multiDrag: true,
      selectedClass: 'selected',
      fallbackTolerance: 3,
      animation: 150,
      forceFallback: true,
      onSelect: function(evt){
        evt.item;
        _this.checkItem(evt.item);
        _this.selected = evt.newIndicies;
        console.log('----select----');
        console.log(this);
        console.log(this.multiDrag);
        console.log(this.multiDragElements);
        console.log(this.multiDrag.multiDragElements);
        // console.log(evt.item);
        // console.log('items:',evt.items);
        // console.log('clone:',evt.clone);
        // console.log('oldIndicies:',evt.oldIndicies);
        // console.log('newIndicies:',evt.newIndicies);
        console.log('----------------');
      },
      onDeselect: function(/**Event*/evt) {
        evt.item;
        console.log('----deselect----');
        console.log(this.multiDrag);
        // console.log(evt.item);
        // console.log('items:',evt.items);
        // console.log('clone:',evt.clone);
        // console.log('oldIndicies:',evt.oldIndicies);
        // console.log('newIndicies:',evt.newIndicies);
        console.log('----------------');
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

    //监听ctrl+A
    document.onkeydown = function(e) {
      let key = window.event.keyCode;
      if (key== 65 && event.ctrlKey) {
        window.event.preventDefault() //关闭浏览器快捷键
          _this.selectAll();
        }
    }

  },
  methods: {
    //返回
    goBack:function(){
      window.history.go(-1);
    },
    //分配icon图标
    iconSelect:function(fileType){
      if(fileType == 1){ //电子原件
        return 'el-icon-tickets';
      }else{ //纸质
        return 'el-icon-document';
      }
    },
    //根据关键词选中,默认全选,关键词空格分隔
    selectAll: function(keywords){
      let keys = [];
      if(keywords){
        keys = keywords.split(' ');
      }
      console.log(keys);
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
      console.log(checkedCount);
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
    //选中元素
    checkItem: function(e){
      let index = e.getAttribute('index');
      this.files[index].checked = 1;
      this.checkChange = true;
    },
    //清除选中
    clearSeleced:function(){
      this.keywords = "";
      this.files.forEach(function(file){
        file.checked = 0;
      });
      this.checkChange = true;
    },
    //更新排序
    updateFilesSort: function(result){
      this.files = result;
      this.history.push(result);
      this.historyIndex++;
      this.topClick = false;
      this.bottomClick = false;
    },
    //撤销
    undo: function(){
      this.historyIndex = Math.max(0,this.historyIndex-1);
      this.files = this.history[this.historyIndex];
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
        console.log(item.title);
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
          paperItems.push(file);
        } else{
          electronItems.push(file);
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
          }).catch(() => {
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
          }).catch(() => {
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
        if(res.data.errcode != 0){
          _this.$alert(res.data.msg);
          return;
        }
        _this.files = [];
        for (let i = 0; i < res.data.data.length; i++) {
          const file = res.data.data[i];
          file.checked = 0;
          _this.files.push(file);
        }
        _this.fullscreenLoading = false;
      }).catch(function(err){
        _this.$alert('初始化失败');
      });
    },
    //保存结果
    save: function(){
      let sortResult = [];
      for (let i = 0; i < this.files.length; i++) {
        const file = this.files[i];
        sortResult.push({
          'id':file.id,
          'sno':i
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

  .el-alert__description {
    -moz-user-select: none;
    -webkit-user-select: none;
    -ms-user-select: none;
    user-select: none;
  }

  /* .selected .el-alert {
    background-color: #d9ecff;
    color: #409EFF;
    border: 1px solid #409EFF;
  }
  .selected .el-alert .el-alert__description {
    font-weight: bold;
    color: #409EFF;
  }

  .sortable-ghost {
    background-color: #fde2e2
  } */

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
</style>
