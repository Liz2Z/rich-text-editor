<template>
  <div class="second-level">
    <div class=" rich-t-tools-wrap">
      <div class="rich-t-tool"
           v-for="item in list"
           @click="select(item.t, item.v)"
           :style="{background:type === 'forecolor' ? item.v : ''}"
           :class="{ active: selected.v === item.v,
         'color-tool': type === 'forecolor'
         }"
      >
        <i class="iconfont" :class="item.t" v-if="type === 'justify'"></i>
        <span v-else>{{item.t}}</span>
      </div>
    </div>
    <div class="confirm-btn" @click="confirmSelect">确认</div>
  </div>
</template>

<script>
  export default {
    name: 'rich-text-tools',
    props: ['type'],
    data () {
      return {
        selected : '',

        list: [],
        fontsize: [
          {t: '10', v: '1'},
          {t: '13', v: '2'},
          {t: '16', v: '3'},
          {t: '18', v: '4'},
          {t: '24', v: '5'},
          {t: '32', v: '6'},
          {t: '48', v: '7'}
        ],
        justify: [
          {t: 'icon-left', v: 'left'},
          {t: 'icon-center', v: 'center'},
          {t: 'icon-right', v: 'right'}
        ],
        forecolor: [
          {t: '', v: '#000'},
          {t: '', v: '#a6a6a6'},
          {t: '', v: '#808080'},
          {t: '', v: '#555'},
          {t: '', v: '#df402a'},
          {t: '', v: '#fae220'},
          {t: '', v: '#77c94b'},
          {t: '', v: '#528cd8'},
          {t: '', v: '#e66c00'},
          {t: '', v: '#2c859d'}
        ]
      }
    },
    methods: {
      select (t, v) {

        this.selected = {t: t, v: v}
      },
      confirmSelect () {
        this.$emit('selected', this.selected)
      },
      // 根据父组件传的类型不同显示不同数据
      caseType (v) {
        this.selected = ''
        switch (v) {
          case 'justify' :   // 对齐方式
            this.list = this.justify
            break
          default:
            this.list = this[v]
            break

        }
      }
    },
    //监测 工具类型的变化
    watch: { type (v) { this.caseType(v) } },
    mounted () { this.caseType(this.type) }
  }
</script>

<style scoped lang='less'>
  @import "../Less/base";

  .rich-t-tools-wrap{
    background: #fff;
    position: absolute;
    top:2px;left:2px;right:50px;
    bottom:2px;
  }
  .confirm-btn{
    position: absolute;
    top:0;bottom:0;
  }
  .confirm-btn{
    right:0;
    padding:0 10px;
    font-size:16px;
    line-height:50px;
    background: #fff;
    .shadow(0 0 10px 0 #999);
    &:active{
      background: #eee;
    }
  }
</style>
