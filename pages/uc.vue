<template>
    <div>
      <h1>用户中心</h1>
      <i class="el-icon-loading"></i>
      <div ref="drag" id="drag">
        <input type="file" name="file" @change="handleFilerChange">
      </div>
      <div>
        <el-progress :stroke-width="20" :text-inside="true" :percentage="uploadProgress"></el-progress>
      </div>
      <div>
        <button @click="uploadFile">上传</button>
      </div>
      <div>
        <p>计算hash的进度</p>
        <el-progress :stroke-width="20" :text-inside="true" :percentage="hashProgress"></el-progress>
      </div>
      <div>
<!--        chunk.progress-->
<!--        progress< 0 报错 显示红色-->
<!--        == 100 成功-->
<!--        别的数字 方块高度显示-->
<!--        尽可能让方块看起来是正方形-->
<!--        比如10个方块 4* 4-->
        <div class="cube-container" :style="{width:cubeWidth + 'px'}">
          <div class="cube" v-for="chunk in chunks" :key="chunk.name">
            <div
              :class="{
                'uploading': chunk.progress< 100 && chunk.progress>0,
                'success': chunk.progress == 100,
                'error': chunk.progress < 0
              }"
              :style="{height:chunk.progress + '%'}"
            >
              <i class="el-icon-loading" style="color:#f56c6c" v-if="chunk.progress< 100 && chunk.progress>0"></i>
            </div>
          </div>
        </div>
      </div>
    </div>
</template>

<script>
  import SparkMD5 from 'spark-md5'
  const CHUNK_SIZE = 0.1*1024*1024
    export default {
      name: "uc",
      data(){
        return {
          file: null,
          // uploadProgress: 0,
          chunks: [],
          hashProgress: 0,
          hash: ''
        }
      },
      computed:{
        cubeWidth(){
          return Math.ceil(Math.sqrt(this.chunks.length)) * 16
        },
        uploadProgress(){
          if(!this.file || this.chunks.length){
            return 0
          }
          const loaded = this.chunks.map(item => item.chunk.size * item.progress)
                              .reduce((acc, cur) => acc + cur, 0)
          return parseInt(((loaded * 100) / this.file.size).toFixed(2))
        }
      },
      async mounted() {
        const ret = await this.$http.get('/user/info')
        console.log(ret)
        this.bindEvents()
      },
      methods: {
        bindEvents () {
          const drag = this.$refs.drag
          drag.addEventListener('dragover', e => {
            drag.style.borderColor = 'red'
            e.preventDefault()
          })
          drag.addEventListener('dragleave', e => {
            drag.style.borderColor = '#eee'
            e.preventDefault()
          })
          drag.addEventListener('drop', e => {
            console.log(e.dataTransfer)
            const fileList = e.dataTransfer.files
            this.file = fileList[0]
            e.preventDefault()
          })
        },
        async blobToString(bolb){
          return new Promise(resolve => {
            const reader = new FileReader()
            reader.onload = function(){
              const ret = reader.result.split('')
                            .map(v => v.charCodeAt())
                            .map(v => v.toString(16).toUpperCase())
                            .map(v => v.padStart(2, '0'))
                            .join(' ')
              resolve(ret)
            }
            reader.readAsBinaryString(bolb)
          })
        },
        async isGif(file){
          // GIF89a  和 GIF87a
          // 前面6个16进制 ’47 49 46 38  39 61’  ‘47 49 46 38 37 61‘
          // 16进制的转换
          const ret = await this.blobToString(file.slice(0, 6))
          console.log(ret)
          const isGif = (ret == '47 49 46 38 39 61') || (ret == '47 49 46 38 37 61')
          return isGif
        },
        async isPng(file){
          const ret = await this.blobToString(file.slice(0, 8))
          const ispng = (ret == '89 50 4E 47 0D 0A 1A 0A')
          return ispng
        },
        async isJpg(file){
          const len = file.size
          const start = await this.blobToString(file.slice(0, 2))
          const tail = await this.blobToString(file.slice(-2, len))
          const isjpg = (start == 'FF D8') && (tail == 'FF D9')
          return isjpg
        },
        async isImage(file){
          // 通过文件流来判定 是一个异步的过程
          // 先判定是不是gif
          return await this.isGif(file) || await this.isPng(file) || await this.isJpg(file)
        },
        createFileChunk(file, size = CHUNK_SIZE){
          const chunks = []
          let cur = 0
          while (cur < this.file.size){
            chunks.push({
              index: cur,
              file: this.file.slice(cur, cur+size)
            })
            cur += size
          }
          return chunks
        },
        async calculateHashWorker(){
          return new Promise(resolve => {
            this.worker = new Worker('/hash.js')
            this.worker.postMessage({chunks: this.chunks})
            this.worker.onmessage = e => {
              const { progress, hash } = e.data
              this.hashProgress = Number(progress.toFixed(2))
              if(hash){
                resolve(hash)
              }
            }
          })
        },
        async calculateHashIdle(){
          return new Promise(resolve => {
            const spark = new SparkMD5.ArrayBuffer()
            let count = 0
            const appendToSpark = async file => {
              return new Promise(resolve => {
                const reader = new FileReader()
                reader.readAsArrayBuffer(file)
                reader.onload = e => {
                  spark.append(e.target.result)
                  resolve()
                }
              })
            }
            const workLoop = async deadline => {
              while(count<this.chunks.length && deadline.timeRemaining() > 1){
                //  空闲时间 且有任务
                await appendToSpark(this.chunks[count].file)
                count ++
                if(count < this.chunks.length){
                  this.hashProgress = Number(
                    ((100*count)/this.chunks.length).toFixed(2)
                  )
                }else{
                  this.hashProgress = 100
                  resolve(spark.end())
                }
              }
              window.requestIdleCallback(workLoop)
            }
            window.requestIdleCallback(workLoop)
          })
        },
        async calculateHashSample() {
          // 布隆过滤器 半段一个数据是否存在
          // 1个G的文件，抽样后5M以内
          // hash一样 文件不一定一样
          // hash 不一样 文件一定不一样
          return new Promise(resolve => {
            const spark = new SparkMD5.ArrayBuffer()
            const reader = new FileReader()

            const file = this.file
            const size = file.size

            const offset = 2*1024*1024
            // 第一个2M,最后一个区块数据全要
            let chunks = [file.slice(0, offset)]
            let cur = offset
            while(cur < size){
              if(cur+offset >= size){
                // 最后一个区块
                chunks.push(file.slice(cur, cur+offset))
              }else{
                // 中间的区块
                const mid = cur+offset/2
                const end = cur+offset
                chunks.push(file.slice(cur, cur+2))
                chunks.push(file.slice(mid, mid+2))
                chunks.push(file.slice(end-2, end))
              }
              cur += offset
            }
            // 中间的，取前中后各2个字节
            reader.readAsArrayBuffer(new Blob(chunks))
            reader.onload = e => {
              spark.append(e.target.result)
              this.hashProgress = 100
              resolve(spark.end())
            }
          })
        },
        async uploadFile(){
          // if(!await this.isImage(this.file)){
          //   console.log('文件格式不对')
          //   return
          // }else{
          //   console.log('文件格式正确')
          // }

          if(!this.file){
            return
          }
          // 制作切片
          const chunks = this.createFileChunk(this.file)
          // console.log(chunks)
          // const hash = await this.calculateHashWorker()
          // console.log('文件hash', hash)
          // const hash1 = await this.calculateHashIdle()
          // console.log('文件hash1', hash1)
          // 抽样hash 不算全量
          // 布隆过滤器 损失一小部分的精度 换取效率
          const hash2 = await this.calculateHashSample()
          console.log('文件hash2', hash2)
          this.hash = hash2

          // 断点续传 问一下后端，文件是否上传过，如果没有，是否存在切片
          const { data: { uploaded, uploadedList } } = await this.$http.post('/checkfile', {
            hash: this.hash,
            ext: this.file.name.split('.').pop()
          })
          console.log(uploaded, uploadedList)
          if(uploaded){
            return this.$message.success('秒传成功！')
          }
          // return
          this.chunks = chunks.map((chunk, index) => {
            // 切片的名字 hash+index
            const name = hash2 + '-' + index
            return {
              hash: hash2,
              name,
              index,
              chunk: chunk.file,
              progress: uploadedList.indexOf(name) >= 1 ? 100 : 0
            }
          })
          // console.log('this.chunks', this.chunks)

          await this.uploadChunks(uploadedList)

          // const form = new FormData()
          // form.append('name', 'file')
          // form.append('file', this.file)
          // const ret = this.$http.post('/uploadfile', form, {
          //   onUploadProgress: progress => {
          //    this.uploadProgress = Number((progress.loaded / progress.total) * 100).toFixed(2)
          //   }
          // })
          // console.log(ret)
        },
        async uploadChunks(uploadedList=[]){
          const requests = this.chunks
            .filter(chunk => uploadedList.indexOf(chunk.name) == -1)
            .map((chunk, index) => {
            const form = new FormData()
            form.append('chunk', chunk.chunk)
            form.append('hash', chunk.hash)
            form.append('name', chunk.name)
            form.append('index', chunk.index)

            return {form, index: chunk.index, error: 0}
          })
            // .map(({form, index}) => {
            // this.$http.post('/uploadfile', form,{
            //   onUploadProgress: progress => {
            //     // 不是整体的精度条，二十每个区块有自己的进度条，整体的进度条需要计算
            //     this.chunks[index].progress = Number((progress.loaded / progress.total) * 100).toFixed(2)
            //   }
            // })
          // })
          //并发量控制
          // 尝试申请tcp链接过多，也会造成卡顿 异步的并发数控制，
          // await Promise.all(requests)
          await this.sendRequest(requests)
          await this.mergeRequest()
        },
        // tcp慢启动，先上传一个初始区块，比如10kb 根据上传成功时间，决定下一个区块是20kb还是50kb还是5k
        // 在下一个一样的逻辑 可能编程 100k 200k
        // 上传可能报错 报错之后 进度条变红 开始重试  一个切片重试失败三次 整体全部终止
        async sendRequest(chunks, limit = 4){
          // limit 是并发数
          // 核心的思路  一个数组，长度是limit [task1, task2, task3]
          return new Promise(((resolve, reject) => {
            const len = chunks.left
            let count = 0
            let isStop = false

            const start = async () => {
              if(isStop){
                return
              }
              const task = chunks.shift()
              if(task){
                const {form, index} = task
                try{
                  await this.$http.post('/uploadfile', form, {
                    onUploadProgress: progress => {
                      this.chunks[index].progress = Number((progress.loaded / progress.total) * 100).toFixed(2)
                    }
                  })
                  if(count == len -1){
                    // 最后一个人人物
                    resolve()
                  }else{
                    count++
                    // 启动下一个任务
                    start()
                  }
                }catch(e){
                  this.chunks[index].progress = -1
                  if(task.error < 3){
                    task.error ++
                    chunks.unshift(task)
                    start()
                  }else{
                    // 错误三次
                    isStop = true
                    reject()
                  }
                }
              }
            }


            while(limit > 0){
              // 启动limit个任务
              setTimeout(() => {
                start()
              }, Math.random() * 2000)
              limit -= 1
            }
          }))
        },
        async mergeRequest() {
          this.$http.post('/mergefile', {
            ext: this.file.name.split('.').pop(),
            size: CHUNK_SIZE,
            hash: this.hash
          })
        },
        handleFilerChange(e) {

          const [file] = e.target.files
          if(!file) return
          this.file = file
        }
      }
    }
</script>

<style lang="stylus">
  #drag
    height 100px
    line-height 100px
    border 2px dashed #eee
    text-align center
    vertical-align middle
  .cube-container
    .cube
      width 14px
      height 14px
      border 1px solid black
      line-height 12px
      background-color #eee
      float left
      .success
        background green
        height 12px
      .uploading
        background blue
        height 12px
      .error
        background red
        height 12px
</style>
