# 01-HTML-4-表单标签

## 一 表单标签

### 1.1 表单标签简介

表单用来收集信息，是前台与后台进行交互的最常见的地方。构成包含：

- 表单域：`form` 标签即代表包裹了一个表单域
- 表单控件：`input` 等

如下所示：

```html
<!--  action:用来处理表单数据 method:表单提交得方式 -->
<form action="" method="">
  <label>账户：</label>
  <input type="text" name="username" placeholder="请输入账户" />
  <label>密码：</label>
  <input type="password" name="password" placeholder="请输入密码" />
</form>
```

### 1.2 表单优化写法

为了提升用户体验，点击 label 标签的文字应该让焦点也到达 label 标签后续的表单控件，for 属性就是这个用途：

```html
<!-- for 中填写对应要获取焦点的 id 即可 -->
<form action="" method="">
  <label for="account">账户：</label>
  <input type="text" id="account" name="username" />
  <label for="pass">密码：</label>
  <input type="password" id="pass" name="password" />
</form>
```

当然也可以使用 label 包裹 表单控件的方式，此时无需利用 for 与 id 属性即可实现自动获取焦点：

```html
<form action="" method="">
  <label>
    账户：
    <input type="text" name="username" />
  </label>
  <label>
    密码：
    <input type="password" name="password" />
  </label>
</form>
```

### 1.3 表单控件

只有位于表单域中的数据才能被提交，这些数据存放于 input 标签（表单控件）中。

input 标签的 type 可以指定控件的类型为文本、密码、单选框、邮箱等等多种类型，而且不同的 type，浏览器默认会有不同的校验，比如当类型有邮箱时，输入的数据不是邮箱，就无法提交。

注意：`type="submit"` 是用于提交表单的按钮。

### 1.4 表单控件属性

表单控件的常见属性有：

```txt
maxlength="6"             限制输入字符长度
readonly="readonly"       将输入框设置为只读状态（不能编辑）
disabled="disabled"       输入框未激活状态
name="username"           输入框的名称
value="大前端"             为当前控件设置默认值，将输入框的内容传给处理文件
placeholder="请输入密码"    提示信息属性
require                   强制填写
autocomplete="on"         开启历史输入数据提示
```

## 二 常用表单控件

```html
<!-- 文本框-->
<input type="text" />

<!-- 密码框：密码输入框的属性与文本输入框一致-->
<input type="password" />

<!-- 单选框：表单控件一般都有 name 属性，单选框如果没有 name，就不能实现单选，checked="checked"，表示默认选中-->
<input type="radio" name="a" />
男
<input type="radio" name="a" />
女

<!-- 多选框 checked 代表默认选中-->
<input type="checkbox" checked="checked" />
喝酒

<!--普通按钮，长与 JavaScript 配合使用-->
<input type="button" vulue="登录" />

<!--提交按钮，用来完成内容提交-->
<input type="submit" />

<!-- 重置按钮：该按钮将页面中的表单控件中的值恢复到默认值 -->
<input type="reset" />

<!-- 上传文件：此时需要设置  form 的 enctype="multipart/form-data"-->
<!-- 上传多个可以使用 name="image[]" 上传限制：accpet="image/png,image/jpeg -->
<input type="file" name="image" />

<!-- 数据提示 -->
<inpyt type="search" name="searchList" list="lesson" />
<datalist id="lsesson">
  <option value="html">html</option>
  <option value="css">css</option>
  <option value="js">js</option>
</datalist>

<!-- 下拉列表：属性 multiple="multiple"可以实现多选-->
<select>
  <option>河北</option>
  <option selected="selected">河南</option>
</select>
<!-- 下拉还可以有更深的子级嵌套： -->
<select>
  <optgroup label="河南">
    <option>南阳</option>
    <option>洛阳</option>
  </optgroup>
</select>

<!-- 分组控件 -->
<fieldset>
  <legend>用户注册信息</legend>
</fieldset>

<!-- 多行文本域 cols  控制输入字符的长度，rows  控制输入的行数-->
<textarea></textarea>
```

## 三 H5 中表单的改变

### 3.1 新增表单类型

H5 的表单增加了很多类型：

```txt
email       输入 email 格式
tel         手机号码，移动设备上得到焦点后会弹出键盘
url         只能输入 url 格式
number      只能输入数字，此时有属性 min  max  step 等
search      搜索框，输入内容时候会出现清除按钮
range       范围滑动条
color       拾色器
time        时间，可控制时间，且有增减按钮
date        日期，可控制日期，且有日期控件！
month       月份，同上
week        星期，同上
datetime    时间日期
```

部分类型是针对移动设备生效的，且具有一定的兼容性，在实际应用当中可选择性的使用，比如：

```js
<form>
    <label>
        邮箱：<input type="email" name="email" class="email">
    </label>
    <input type="submit" value="提交">
</form>
```

上述表单类型设置为 email 后，可以验证邮箱书写的合法性。

### 3.2 新增表单属性

```txt
placeholder     占位符
autofocus       获取焦点
multiple        文件上传多选或多个邮箱地址
autocomplete    自动完成，用于表单元素，也可用于表单自身 (on/off)
form            指定表单项属于哪个 form，处理复杂表单时会需要
novalidate      关闭验证，可用于<form>标签
required        必填项
pattern         正则表达式 验证表单

autocapitalize  iOS 独有属性，设置为 off 关闭首字母大写
autocorrect     iOS 独有属性，设置为 off 关闭输入自动修正
```

### 3.3 新增表单事件

```txt
oninput         用户输入内容时触发，可用于移动端输入字数统计
oninvalid       验证不通过时触发
```

### 3.4 表单自动联想

```html
<input type="text" list="data" />
<datalist id="data">
  <option>男</option>
  <option>女</option>
</datalist>
```
