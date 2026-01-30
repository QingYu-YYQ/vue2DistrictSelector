# vue2DistrictSelector
基于Vue2.0的移动端省/市/区选择器

# 一、实现效果
<img width="469" height="947" alt="image" src="https://github.com/user-attachments/assets/efe18421-cca9-4974-85d6-6d0ec989bb6a" />

<img width="469" height="951" alt="image" src="https://github.com/user-attachments/assets/3d32a15f-aed6-4b81-81ee-3764c740ab62" />

### 1、支持分级选择地址（最多三级）
### 2、支持输入至少2个字模糊搜索
### 3、支持切换省/市/区重新选择
### 4、支持默认已选地区
### 5、isFinalNode 判断是否最里层，限制须选择所有层级


# 二、调用方式
```
<DistrictSelector
  ref="districtSelectorRef"
  :districts="districts"
  @confirm="districtSelectorConfirm"
/>
```
### 默认值，打开地区选择器
```
handleChooseArea (d) {
  this.districts = d; // 默认地区
  this.$refs.districtSelectorRef.show = true;
}

```
### 处理返回
```
districtSelectorConfirm (selected) {
  let d = JSON.parse(JSON.stringify(selected));
  for (let i = 0; i < 3; i++) {
    if (!d[i]) {
      d[i] = {
        label: '',
        value: '',
      };
    }
  }

  const place = d.map((item) => item.label).filter((item) => item !== '').join('-');
  
  this.addList = d;
  this.place = place; // 页面展示，如：江苏省-苏州市-姑苏区
},
```

### 参数说明
districts: [], // 默认的省市区
结构：[{value: '省id', label: '省名称'},{value: '市id', label: '市名称'},{value: '区id', label: '区名称'}] // 可为[], 最长3位

districtSelectorConfirm  选择器返回结果，包含字段：label(名称), value(id), level(层级0~3), parentValue(上一级id), isFinalNode(是否最里层)



# 三、组件内部接口返回的地区数据结构示例
<img width="1047" height="1008" alt="image" src="https://github.com/user-attachments/assets/b99c1828-88a7-423c-b335-13fc9250423e" />

