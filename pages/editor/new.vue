<template>
  <div>
    <div contenteditable="true">自定义编辑器</div>
<!--    document.ex二次Command（'')-->
    <div class="write-btn">
      <el-button @click="submit" type="primary">提交</el-button>
    </div>
    <el-row>
      <el-col :span="12">
        <textarea ref="editor" class="md-editor" @input="update" @value="content"></textarea>
      </el-col>
      <el-col :span="12">
        <div class="markdown-body" v-html="compiledContent"></div>
      </el-col>
    </el-row>
  </div>
</template>

<script>
    import marked from 'marked'
    import hljs from 'highlight.js'
    import javascript from 'highlight.js/lib/languages/javascript'
    import 'highlight.js/styles/monokai-sublime.css'
    export default {
      name: "new",
      data(){
        return {
          content: `# zzz
          * 上班
          * 吃饭
          * 睡觉`
        }
      },
      mounted() {
        this.timer = null
        this.bindEvent()
        // marked的配置
        marked.setOptions({
          render: new marked.Renderer(),
          highlight(code){
            return hljs.highlightAuto(code).value
          }
        })
      },
      methods:{
        bindEvent(){
          this.$ref.editor.addEventListener('paste', async e => {
            const files = e.clipboardData.files
            console.log(files)
            // 直接上传
          })
          this.$ref.editor.addEventListener('drop', async e => {
            const files = e.dataTransfer.files
            console.log(files)
            // 直接上传
            e.preventDefault()
          })
        },
        // lodash/debounce  也可以用
        update(e){
          clearTimeout(this.timer)
          this.timer = setTimeout(()=>{
            this.content = e.target.value
          }, 350)
        },
        async submit(){
          let ret = await this.$http.post('/article/create', {
            content: this.content, // selected: false
            compiledContent: this.compiledContent  // 显示只读取这个
          })
        }
      },
      computed: {
        compiledContent(){
          return marked(this.content, {})
        }
      }
    }
</script>

<style scoped>
  .md-editor{
    width: 100%;
    height: 100vh;
    outline: none;
  }
  .write-btn{
    position: fixed;
    right:10px;
    top:10px;
    z-index: 100;
  }
</style>
