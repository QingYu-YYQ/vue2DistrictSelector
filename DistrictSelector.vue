<template>
  <div class="district-selector">
    <van-popup v-model="show" round position="bottom" :style="{ height: '80vh' }" closeable>
      <div class="title">{{ title }}</div>
      <div class="selected-group-box">
        <span class="selected-group-title">已选地区：</span>
        <span class="selected-group">{{ selectedGroup.map((item) => item.label).join(' / ') }}</span>
      </div>
      <div class="search-box">
        <van-search v-model="searchValue" placeholder="省/市/区" clearable shape="round" background="#FFFFFF" />
        <div v-if="searchResultLists.length > 0" class="search-result">
          <div v-for="item in searchResultLists" :key="item.value" :title="item.label" @click="handleSelect(item)" class="search-item">
            <span v-html="highlightedText(item.map((item) => item.label).join(' / '))"></span>
          </div>
        </div>
      </div>
      <div class="tab-box">
        <div class="tab-bar">
          <div @click="handleTabClick(item)" v-for="item in tabLists" :key="item.value" class="tab-item">
            <span :style="{'color': item.value === null ? '#999999' : ''}">{{ item.label }}</span>
            <div :class="{'active': item.level === activeTab}" class="tab-bar-line"></div>
          </div>
        </div>
        <div class="tab-content">
          <div @click="handleClick(item)" v-for="item in tabResultLists" :key="item.value" class="content-item">
            <div class="content-left" :class="{'checked': item.checked}">
              <span v-if="item.showFirstChar" class="index">{{ item.firstChar }}</span>
              <span class="label">{{ item.label }}</span>
            </div>
            <van-icon v-if="item.checked" name="success" color="#409EFF" />
          </div>
        </div>
      </div>
      <div class="footer" @click="handleConfirm">确认</div>
    </van-popup>
  </div>
</template>
<script>
import { getAddress } from '@/request/api';
import pinyin from 'js-pinyin';
import { Toast } from 'vant';
export default {
  name: 'DistrictSelector',
  props: {
    title: {
      type: String,
      default: '所在地区',
    },
    districts: {
      type: Array,
      default: () => [],
    }
  },
  watch: {
    districts: {
      handler(newVal, oldVal) {
        if (newVal.length > 0) {
          this.preprocess(newVal);
        } else {
          // 初始化
          this.handleReset();
        }
      },
      deep: true,
    },
    searchValue (newVal, oldVal) {
      if (newVal.trim() === '') {
        this.searchResultLists = [];
      } else if (newVal !== oldVal) {
        this.handleSearch();
      }
    }
  },
  data() {
    return {
      show: false,
      searchValue: '',

      baseLists: [], // 省市区列表嵌套
      activeTab: 0, // 选中的tab索引
      tabLists: [
        {
          value: null,
          label: '请选择',
          level: 0,
        },
      ], // 省市区tab列表
      tabResultLists: [], // tab切换列表 checked
      searchResultLists: [
        // [
        //   {
        //     value: '232423423523489875',
        //     label: '云南',
        //   },
        //   {
        //     value: '829681197151031296',
        //     label: '大理',
        //   },
        //   {
        //     value: '829681197268471808',
        //     label: '祥云',
        //   },
        // ],
        // [
        //   {
        //     value: '232352436564564576',
        //     label: '江苏',
        //   },
        //   {
        //     value: '839681197151031296',
        //     label: '苏州',
        //   },
        // ],
      ], // 搜索结果列表  悬浮
      selectedGroup: [
        // {
        //   label: '河北省',
        //   value: '82968119715103434',
        // },
        // {
        //   label: '保定市',
        //   value: '829681197151031296',
        // },
        // {
        //   label: '朝阳区',
        //   value: '829681197331386368',
        // },
      ], // 选中的省市区组
      timer: null, // 搜索防抖定时器
    }
  },
  mounted() {
    this.getAddressList();
  },
  methods: {
    handleReset() {
      this.tabResultLists = this.baseLists;

      this.searchValue = this.$options.data().searchValue;
      this.activeTab = this.$options.data().activeTab;
      this.tabLists = this.$options.data().tabLists;
      this.searchResultLists = this.$options.data().searchResultLists;
      this.selectedGroup = this.$options.data().selectedGroup;
      this.timer = this.$options.data().timer;
    },
    preprocess (newVal) {
      let selectedList = [];
      this.tabLists = this.$options.data().tabLists;
      newVal.forEach((item, index) => {
        if (item.value) {
          const parentItem = index > 0 ? newVal[index - 1] : null;
          selectedList.push({
            value: item.value,
            label: item.label,
            level: index,
            parentValue: parentItem ? parentItem.value : null,
          });
        }
      });

      // 递归baseLists, 找到与selectedList中每个item.value相同的项，返回新的数组
      const newSelectedList = selectedList.map(item => {
        const children = this.recursiveSearch(this.baseLists, item.value);
        return children;
      });
      
      this.selectedGroup = newSelectedList.map(item => ({
        label: item.label,
        value: item.value,
        level: item.level,
        parentValue: item.parentValue,
      }));

      newSelectedList.forEach((item, index) => {
        this.tabLists[index] = {
          value: item.value,
          label: item.label,
          level: item.level,
          parentValue: item.parentValue,
        };
      });

      this.tabLists = JSON.parse(JSON.stringify(this.tabLists));
      
      const lastItem = newSelectedList[newSelectedList.length - 1];
      if (lastItem.children && lastItem.children.length > 0) {
        this.tabLists.push({ label: '请选择', value: null, level: this.tabLists.length });
        this.tabResultLists = (lastItem.children || []).map(item => ({
          ...item,
          checked: this.selectedGroup.findIndex(group => group.value === item.value) !== -1 || false,
        })) || [];
      } else {
        if (newSelectedList.length > 1) {
          this.tabResultLists = (newSelectedList[newSelectedList.length - 2].children || []).map(item => ({
            ...item,
            checked: this.selectedGroup.findIndex(group => group.value === item.value) !== -1 || false,
          })) || [];
        } else {
          this.tabResultLists = [];
        }
      }

      this.activeTab = this.tabLists.length - 1;

      this.searchValue = '';
      this.searchResultLists = [];
    },
    handleTabClick (row) {
      if (row.value === null) return;
      this.activeTab = row.level;
      // 递归baseLists，找到row.parentValue所在列并返回
      if (!row.parentValue && row.level === 0) {
        this.tabResultLists = (this.baseLists || []).map(item => ({
          ...item,
          checked: this.selectedGroup.findIndex(group => group.value === item.value) !== -1 || false,
        })) || [];
      } else if (!row.parentValue) {
        this.tabResultLists = [];
      } else {
        const item = this.recursiveSearch(this.baseLists, row.parentValue);
        this.tabResultLists = (item.children || []).map(item => ({
          ...item,
          checked: this.selectedGroup.findIndex(group => group.value === item.value) !== -1 || false,
        })) || [];
      }
    },

    // 递归函数，用于在嵌套数组中搜索匹配项
    recursiveSearch (arr, searchValue) {
      for (const item of arr) {
        if (item.value === searchValue) {
          return item;
        }
        if (item.children && item.children.length > 0) {
          const result = this.recursiveSearch(item.children, searchValue);
          if (result) {
            return result;
          }
        }
      }
      return null;
    },

    handleClick (row) {
      this.tabResultLists.forEach(item => {
        item.checked = false;
        if (item.value === row.value) {
          item.checked = true;
          this.tabLists[item.level] = {
            value: item.value,
            label: item.label,
            level: item.level,
            parentValue: item.parentValue,
          };
          // 删除this.tabLists中索引大于item.level的项
          this.tabLists = this.tabLists.slice(0, item.level + 1);
          // 加入当前项
          this.selectedGroup[item.level] = {
            label: item.label,
            value: item.value,
            level: item.level,
            parentValue: item.parentValue,
          };
          // 删除this.selectedGroup中索引大于item.level的项
          this.selectedGroup = this.selectedGroup.slice(0, item.level + 1);
        }
      });

      this.tabLists = JSON.parse(JSON.stringify(this.tabLists));
      this.selectedGroup = JSON.parse(JSON.stringify(this.selectedGroup));

      if (row.children && row.children.length > 0) {
        this.tabLists.push({ label: '请选择', value: null, level: this.tabLists.length });
        this.activeTab = this.tabLists.length - 1;
        this.tabResultLists = (row.children || []).map(item => ({
          ...item,
          checked: this.selectedGroup.findIndex(group => group.value === item.value) !== -1 || false,
        })) || [];
      }
    },
    handleSearch () {
      if (this.searchValue.length < 2) return;
      // 防抖
      if (this.timer) clearTimeout(this.timer);
      this.timer = setTimeout(() => {
        // 循环baseLists拿到筛选结果
        const result = [];
        this.baseLists.forEach(item => {
          if (item.label.includes(this.searchValue)) {
            result.push([item]);
          }
          // 第一级children
          if (item.children && item.children.length > 0) {
            item.children.forEach(child => {
              if (child.label.includes(this.searchValue)) {
                result.push([item, child]);
              }
              // 第二级children
              if (child.children && child.children.length > 0) {
                child.children.forEach((grandchild) => {
                  if (grandchild.label.includes(this.searchValue)) {
                    result.push([item, child, grandchild]);
                  }
                });
              }
            });
          }
        });
        this.searchResultLists = result;
      }, 500);
    },
    highlightedText(text) {
      const highlighted = text.replace(new RegExp(this.searchValue, 'g'), '<span class="highlight">$&</span>');
      return highlighted;
    },
    handleSelect (selectedList) {
      this.selectedGroup = selectedList.map(item => ({
        label: item.label,
        value: item.value,
        level: item.level,
        parentValue: item.parentValue,
      }));

      this.tabLists = this.$options.data().tabLists;
      selectedList.forEach((item, index) => {
        this.tabLists[index] = {
          value: item.value,
          label: item.label,
          level: item.level,
          parentValue: item.parentValue,
        };
      });

      this.tabLists = JSON.parse(JSON.stringify(this.tabLists));

      const lastItem = selectedList[selectedList.length - 1];
      if (lastItem.children && lastItem.children.length > 0) {
        this.tabLists.push({ label: '请选择', value: null, level: this.tabLists.length });
        this.tabResultLists = (lastItem.children || []).map(item => ({
          ...item,
          checked: this.selectedGroup.findIndex(group => group.value === item.value) !== -1 || false,
        })) || [];
      } else {
        this.tabResultLists = (selectedList[selectedList.length - 2].children || []).map(item => ({
          ...item,
          checked: this.selectedGroup.findIndex(group => group.value === item.value) !== -1 || false,
        })) || [];
      }

      this.activeTab = this.tabLists.length - 1;

      this.searchValue = '';
      this.searchResultLists = [];
    },
    handleConfirm () {
      if (this.selectedGroup.length === 0) {
        Toast(`请选择${this.title}`);
        return;
      }
      // this.selectedGroup 包含 label, value, level, parentValue
      this.$emit('confirm', this.selectedGroup);
      this.show = false;
    },
    getAddressList() {
      getAddress().then((res) => {
        if (res.code == 200) {
          const list = this.addPropToTree(res.data);
          const sortedList = this.sortTreeByPinyinName(list);
          // console.log('处理后：', sortedList);
          this.baseLists = JSON.parse(JSON.stringify(sortedList));
          this.tabResultLists = this.baseLists;
        }
      });
    },
    // 递归为树形结构添加parentValue、pinyinName、firstChar属性
    addPropToTree(tree, parentValue = null, level = 0) {
      if (!Array.isArray(tree) || tree.length === 0) {
        return tree;
      }

      return tree.map(node => {
        // 创建新节点，避免修改原对象
        let pinyinName = pinyin.getCamelChars(node.label);
        // label第一个字的声母
        let firstChar = pinyin.getCamelChars(node.label[0]);

        if (node.label === '重庆市') {
          pinyinName = 'CQS';
          firstChar = 'C';
        }
        const newNode = {
          ...node,
          pinyinName,
          firstChar,
          parentValue: parentValue,
          level: level,
        };
        
        // 递归处理子节点，将当前节点的 value 作为子节点的 parentValue
        // 增加层级 level
        if (node.children && Array.isArray(node.children) && node.children.length > 0) {
          newNode.children = this.addPropToTree(node.children, node.value, newNode.level + 1);
        }
        
        return newNode;
      });
    },
    // 递归,对树形结构根据pinyinName进行排序
    sortTreeByPinyinName(nodes) {
      if (!nodes || !Array.isArray(nodes)) {
        return nodes;
      }

      // 对当前层级的节点按照 pinyinName 进行排序
      nodes.sort((a, b) => {
        const pinyinA = a.pinyinName || '';
        const pinyinB = b.pinyinName || '';
        return pinyinA.localeCompare(pinyinB);
      });

      // 找出 nodes 中首次出现的不同声母的节点，并在该节点添加showFirstChar属性，用于展示声母索引
      let firstChars = [];
      nodes.forEach(node => {
        if (!firstChars.includes(node.firstChar)) {
          firstChars.push(node.firstChar);
          node.showFirstChar = true;
        }
      })

      // 递归处理每个节点的 children
      nodes.forEach(node => {
        if (node.children && Array.isArray(node.children) && node.children.length > 0) {
          this.sortTreeByPinyinName(node.children);
        }
      });

      return nodes;
    },
  },
}
</script>
<style>
.highlight {
  color: #409EFF;
}
</style>
<style lang="less" scoped>
@colosrs: #409EFF;
.district-selector {
  color: #333333;
  width: 100%;
  .title {
    font-size: 16px;
    font-weight: bold;
    height: 60px;
    line-height: 60px;
    text-align: center;
    border-bottom: 1px solid #f0f0f0;
  }
  .selected-group-box {
    font-weight: bold;
    padding: 10px 15px;
    border-bottom: 1px solid #f0f0f0;
    display: flex;
    justify-content: space-between;
    flex-wrap: nowrap;
    align-items: center;
    .selected-group-title {
      font-weight: bold;
      flex-shrink: 0;
      font-size: 14px;
    }
    .selected-group {
      font-size: 14px;
      color: @colosrs;
      width: calc(100vw - 30px);
      font-weight: normal;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }
  }
  .search-box {
    padding-bottom: 0;
    position: relative;
    .search-result {
      position: absolute;
      top: 40px;
      left: 0;
      width: 100%;
      background: #fff;
      box-sizing: border-box;
      padding: 5px 15px;
      box-shadow: 0 5px 5px -2px rgba(0, 0, 0, 0.1);
      z-index: 100;
      .search-item {
        font-size: 14px;
        padding: 9px 0;
        border-bottom: 1px solid #f0f0f0;
      }
    }
  }
  .tab-box {
    .tab-bar {
      display: flex;
      justify-content: flex-start;
      flex-wrap: nowrap;
      align-items: center;
      height: 30px;
      line-height: 30px;
      font-size: 14px;
      color: #333333;
      padding: 0 10px;
      .tab-item {
        padding: 0 10px;
        cursor: pointer;
        max-width: 100px;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
        .tab-bar-line {
          height: 3px;
          border-radius: 2px;
          &.active {
            background: @colosrs;
          }
        }
      }
    }
    .tab-content {
      padding: 10px;
      max-height: calc(80vh - 190px);
      padding-bottom: 50px;
      box-sizing: border-box;
      overflow-y: auto;
      .content-item {
        display: flex;
        justify-content: space-between;
        flex-wrap: nowrap;
        align-items: center;
        padding: 10px 0;
        .content-left {
          position: relative;
          .index {
            position: absolute;
            top: 2px;
            left: 0;
            font-size: 14px;
          }
          .label {
            margin-left: 20px;
            font-size: 14px;
          }
        }
        .checked {
          .label {
            color: @colosrs;
          }
        }
      }
    }
  }

  .footer {
    position: fixed;
    bottom: 0;
    left: 0;
    width: 100vw;
    height: 50px;
    line-height: 50px;
    box-sizing: border-box;
    text-align: center;
    background: #fff;
    border-top: 1px solid #f0f0f0;
    color: @colosrs;
    font-weight: bold;
  }
}
</style>
