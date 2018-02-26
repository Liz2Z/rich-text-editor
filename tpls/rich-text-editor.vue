<template>
  <div class="rich-text-editor">

    <div class="editor-hd" >
      <!-- 一级功能区域 -->
      <div class="rich-t-tools-wrap">
        <!-- 字体大小 -->
        <div class="rich-t-tool" @click="openSecondLevel('fontsize')">{{activeOpt.fontsize}}</div>
        <!-- 粗体 -->
        <div class="rich-t-tool" @click="setFont('bold')" :class="{'active':activeOpt.bold}">
          <i class="iconfont icon-bold"></i>
        </div>
        <!-- 斜体 -->
        <div class="rich-t-tool" @click="setFont('italic')" :class="{'active':activeOpt.italic}">
          <i class="iconfont icon-italic"></i>
        </div>
        <!-- 对齐方式 -->
        <div class="rich-t-tool" @click="openSecondLevel('justify')" >
          <i class="iconfont" :class="activeOpt.justify"></i>
        </div>
        <!-- 颜色 -->
        <div class="rich-t-tool" @click="openSecondLevel('forecolor')" :style="'color:'+activeOpt.forecolor">Az</div>
        <!-- 插入图片 -->
        <div class="rich-t-tool" @click="toggleUpload(1)">
          <i class="iconfont icon-pic"></i>
        </div>
        <!--<div class="tool" @click="bgColor">背景</div>-->
      </div>
      <!-- 二级功能区 -->
      <second-level
        v-if="SLToolsType !== 0"
        :type = 'SLToolsType'
        @selected="selectedSecondLevel"
      />
    </div>

    <div class="editor-bd">
      <!-- 编辑区域 -->
      <div class="rich-t-edit-area needsclick"
           ref="editor"
           contenteditable="true"
           @click="checkStyle"
           @keydown="keyDown"
           @focus="activekbd = true"
           @blur="activekbd = false"
      ></div>
    </div>


    <!-- 提交 -->
    <div class="rich-t-submit-btn" :class="{active: activekbd}">
      <button class=" btn base-fill" type="button" @click="submit">提交</button>
    </div>

    <!-- 上传图片 -->
    <div class="upload-img-mask" v-if="showUpload">
      <form  class="upload-img">
        <canvas style="display: none;" id="js-canvas" ref="canvas"></canvas>
        <p class="upload-img-title">上传图片</p>
        <label for="upload-img-input">
          {{selectedImg ? selectedImg : '选择图片'}}
          <input type="file" ref="imgInput" id="upload-img-input" name="image" accept= 'image/*' @change="imgInput($event)">
        </label>
        <p class="upload-btn" @click="submitImg">开始上传</p>
        <p class="upload-close-btn" @click="toggleUpload(-1)">关闭</p>
      </form>
    </div>
  </div>
</template>

<script>

  import secondLevel from '@/tpls/rich-text-tools'
  export default {
    name: 'rich-text-editor',
    data () {
      return {
        activekbd: false,
        showUpload: false,    // 显示 上传图片工具
        selectedImg: '',       // 选中图片
        /****
         * secondLevelToolsType 第二级工具类型
         * 同时控制是否显示  0 不显示
         * */
        SLToolsType: 0,
        /****
         * 激活的样式设置，用于界面显示，提示用户
         * */
        activeOpt: {
          fontsize: '16',
          forecolor: '#000',
          justify: 'icon-left',
          bold: false,
          italic: false
        },
        /****
         * 样式收集器
         * */
        collectOpt: {}

      }
    },
    methods: {
      /****
       * 在 ios 上  只有在编辑器失去焦点, 再点击才会触发
       * 用于界面显示  提示用户当前范围激活了哪些样式
       * */
      checkStyle () {

        let d = document,
          activeOpt = {
            bold: d.queryCommandState('bold'),
            italic: d.queryCommandState('italic'),
            forecolor: d.queryCommandValue('forecolor')
          },
          fontsize =  d.queryCommandValue('fontsize'),
          justify = d.queryCommandState('justifyleft') ?
            'icon-left' :
            d.queryCommandState('justifyright') ?
              'icon-right' :
              d.queryCommandState('justifycenter') ?
                'icon-center':
                'icon-left'

        switch (fontsize) {
          case '1': fontsize = 10; break;
          case '2': fontsize = 13; break;
          case '3': fontsize = 16; break;
          case '4': fontsize = 18; break;
          case '5': fontsize = 24; break;
          case '6': fontsize = 32; break;
          case '7': fontsize = 38; break;
          default: fontsize = '字体'; break;
        }
        this.activeOpt = Object.assign({
          fontsize,
          justify,
        }, activeOpt)
      },
      // 打开二级菜单
      openSecondLevel (v) {
        this.SLToolsType = v
      },
      //  接收 二级菜单组件传回来的值
      selectedSecondLevel (data) {
        if (data) {
          let type = this.SLToolsType,
            obj = {}
          // 收集用户设置
          this.collectOption(type, type === 'justify' ? 'justify'+data.v : data.v)

          // 显示应用样式
          if (type === 'fontsize' || type === 'justify') {
            obj[type] = data.t
          } else {
            obj[type] = data.v
          }
          Object.assign(this.activeOpt, obj)
        }
        this.SLToolsType = 0
      },
      // 粗体 斜体
      setFont (v) {
        this.activeOpt[v] = !this.activeOpt[v]
        this.collectOption(v, null, false)
      },
      /*
      * 富文本框工作模式,
      * !!!!   必须先获得焦点, 才能设置样式  !!!!
      *
      * 情况一：   ==>> 当设置某个样式后，没有输入内容的情况下失去焦点，该样式不会生效 <<==
      *            如： 点击bold字体按钮，编辑器获得焦点，设置了bold字体，随后点击italic字体按钮，
      *            文本框失去焦点，bold字体不会生效
      * 解决办法： 创建一个操作收集器对象，把用户设置的样式先收集起来，然后统一设置
      *            监听收集集合变化，每次有新样式加入，或者 老样式改变 就重新设置
      *
      * 情况二：   ==> 背景为使用了情况一解决方案，在同一个位置多次设置同一样式样式，新样式 会覆盖原来的 <==
      *            前提：已经设置 bold字体
      *            操作：修改字体大小
      *            情况：该位置的所有设置都要重新设置一遍，导致该位置bold字体被改为普通字体
      * 解决办法： 设置了样式后，检测到字符输入，则 删除 样式收集器中的 bold 及 italic 项
      *
      * */
      /****
       * 收集用户操作
       * 用于统一设置富文本框
       * 最后一个 参数用于 判断 当设置同一个样式时,是覆盖原样式, 还是删除
       * 例如: 当点击  bold字体 , 再次点击bold 选项, 则删除此属性
       *       当点击  不同的字体大小  则  覆盖
       *
       * */
      collectOption (property, value  = null, ifCover = true) {
        let obj = this.collectOpt
        if (obj.hasOwnProperty(property) && !ifCover) {
          //  如果 已经有了, 且不是覆盖的 ,就删除
          delete obj[property]
        } else {
          obj[property] = value
        }
        this.collectOpt = Object.assign({}, obj)
        this.collapseSelection()
      },
      /****设置选区,让光标出现在最后位置*/
      collapseSelection () {
        /*
         *   @method:折叠选区范围 以 设置 光标位置
         *      |                 |
         *     锚点, 选区起点     焦点, 选区终点
         *     将选区范围从锚点折叠到焦点, 即锚点与焦点位置重合, 光标就会出现在终点位置
         *     当选区范围已折叠, sel.isCollapsed  为true
         *
         *     如果此范围失去焦点, 光标消失之后
         *     再进行折叠到相同位置处, 则没有效果,光标不会出现
         *     因为该处已经折叠了
         *     此时需要把选区折叠到其他位置, 再折叠回来
         * */
        let editor = this.$refs.editor,
          len = editor.childNodes.length,
          sel = window.getSelection()
        sel.collapse(document.body, 0)
        sel.collapse(editor, len)
        editor.scrollTop = editor.clientHeight
      },
      //  富文本编辑器命令
      execCommand (option, value = null, ifCSS = true) {
        // 默认用CSS 替代格式元素
        if (ifCSS) { document.execCommand('styleWithCSS', false, true) }
        document.execCommand(option, false, value)
      },
      keyDown () {
        let obj = this.collectOpt
        // 如果有justify必须提前设置, 不然在其之前的样式都会失效
        if (obj.hasOwnProperty('justify')) {
          this.execCommand(obj.justify)
          delete obj.justify
        }
        for (let item in obj) {
          if (obj.hasOwnProperty(item)) {
            this.execCommand(item, obj[item])
          }
        }
        this.collectOpt = {}
      },


      // 插入图片
      toggleUpload (v) {
        this.showUpload = v > 0
        this.selectedImg = ''
      },
      // 监听图片输入框 change
      imgInput (eve) {
        let file = eve.target.files[0]
        this.selectedImg = file.name
        let URL = window.URL || window.webkitURL,
          dataURL = URL.createObjectURL(file)
        let img = new Image()
        img.crossOrigin = 'anonymous'
        img.onload = (function (receiver, callback) {
          return function (e) {
            let img = e.target
            img.onload = ''
            URL.revokeObjectURL(img.src)  // 解除URL对象

            let canvas = document.getElementById('js-canvas'),
              ctx = canvas.getContext('2d')
            canvas.width = img.width
            canvas.height = img.height
            ctx.drawImage(img, 0, 0)
            receiver.focus()   // 富文本编辑器必须聚焦才可设置
            let imgWrap = document.createElement('p'),
              newImg = document.createElement('img')
            newImg.src = canvas.toDataURL()
            newImg.style = 'width:100%'
            imgWrap.appendChild(newImg)
            receiver.appendChild(imgWrap)
            callback()
          }
        })(this.$refs.editor, () =>{ this.showUpload = false; this.collapseSelection()})
        img.src = dataURL

      },
      // 提交图片
      submitImg () {
        this.$axios.post('/img',{data: {img: this.$refs.imgInput.value}})
        .then( res => {

        })
        .catch( err => {
          console.log(err)
        })
      },
      // 提交 
      submit () {
        let html = this.$refs.editor.innerHTML
        if (html) {
          let temp = document.createElement('div')
          temp.innerText = html
          html = temp.innerHTML
          this.$emit('submit', html)
        }
      }
    },
    components: {
      'second-level': secondLevel
    }
  }
</script>

<style lang='less'>
  @import "../Less/base";

  .upload-img-mask{
    position: absolute;
    z-index: 30;
    top:0;left:0;right:0;bottom:0;
    background: rgba(0, 0, 0, 0.7);
  }
  .upload-img{
    position: fixed;
    top:50%;left:50%;
    -webkit-transform: translate(-50%, -50%);
    transform: translate(-50%, -50%);
    background:#fff;
    .radius(2px);
    width:300px;
    font-size:14px;
  }
  .upload-close-btn{
    position: absolute;
    top:-17px;right:-17px;
    width:34px;
    .radius(50%);
  }
  #upload-img-input{display: none;}
  .upload-img-title{
    padding:0 6px;
    height:34px;
    line-height:34px;
    border-bottom:1px solid #eee;
  }
  [for='upload-img-input']{
    display:block;
    margin:25px;
    background: #33cc99;
    font-size:13px;
    .radius(4px);
    &:active{
      background: #2fbb8c;
    }
  }
  .upload-close-btn,
  [for='upload-img-input'],
  .upload-btn{
    height:34px;
    line-height:34px;
    color:#fff;
    text-align:center;
  }
  .upload-close-btn,
  .upload-btn{
    background: #6699ff;
    &:active{
      background: darken(#6699ff, 5%);
    }
  }


  .editor-hd{
    overflow: hidden;
    -webkit-user-select: none;
    user-select: none;
    position: relative;
    background: #fff;
    padding:3px;
  }
  .rich-t-tools-wrap{
    overflow: scroll;
    -webkit-overflow-scrolling: touch;
    white-space: nowrap;
    font-size:0;
    padding:3px;
    background: rgba(35, 134, 251, 0.35);
    .radius(8px)
  }
  .rich-t-tool{
    display: inline-block;
    vertical-align: top;

    margin-right: 6px;
    padding:10px;
    height:36px;
    min-width:36px;

    font-size:16px;
    line-height:1;

    background:#EEE;
    color:#444;
    .radius(4px);

    -webkit-transition: all ease .2s;
    transition: all ease .2s;
    &:active{ background:darken(#EEE, 7%) }
    &.active{
      background:#888;
      color:#fff;
    }

    -webkit-transform: scale3d(1,1,1);
    transform: scale3d(1,1,1);
    &.color-tool.active{
      -webkit-transform: scale3d(1.3,1.3,1.3);
      transform: scale3d(1.3,1.3,1.3);
    }
  }

  .editor-bd{
    padding:5px;
    overflow:scroll;
    -webkit-overflow-scrolling: touch;
    max-height:500px;
  }
  .rich-t-edit-area{
    overflow: hidden;
    word-break: break-all;
    outline: none;
    .radius(4px);
    min-height:150px;
    padding: 10px;
    border:1px solid #eee;
    &:focus{ border:1px solid darken(#2386fb,6%); }
  }

  .rich-t-submit-btn{
    padding:10px 5px;
    background: #fff;
    &.active{opacity: 0;}
  }
</style>
