<template>
    <div class="content">
        <canvas id="canvas" class="cavans" ref="canvas" width="800" height="800"></canvas>
        <div class="rightHandle">
            <el-upload
                action="http://127.0.0.1"
                multiple
                :auto-upload="false"
                :limit="limit"
                :on-change="slectFile"
            >
                <el-button type="primary">{{ `选择${limit === 1 ? '目标' : '填充'}文件` }}</el-button>
            </el-upload>
            <span v-show="!isTarget" style="margin-top: 20px;">已选择填充图片：{{ count }}张</span>
            <el-button type="primary" @click="reload">重置</el-button>
            <el-button type="primary" @click="generateImg">生成图片</el-button>
            <el-button type="primary" @click="exportCanvas">导出图片</el-button>
        </div>
    </div>
</template>

<script setup>
import { fabric } from 'fabric'
import { ref, onMounted } from 'vue'
import { getAverageColor } from '../utils/averageColor'

// canvas
let canvas = null
let ctx = null
const blockList = [] //画布数据
let blockMainColors = [] // 每个格子的主色调
//初始化画布
const initCanvas = () => {
    canvas = new fabric.Canvas('canvas', {
        isDrawingMode: false, //自由绘画模式
        selectable: false,
        selection: false,
        hoverCursor: 'pointer',
        devicePixelRatio: true, //Retina 高清屏 屏幕支持
        stroke: 'lightgreen',
        strokeWidth: 4,
    })
    ctx = canvas.getContext('2d')
    canvas.freeDrawingBrush.color = 'blue'
    canvas.freeDrawingBrush.width = 5
}
//获取画布像素数据
const getCanvasData = () => {
    for (let Y = 0; Y < canvas.height / 4; Y++) {
        for (let X = 0; X < canvas.width / 4; X++) {
            //每4*4像素的一块区域一组
            let tempColorData = ctx.getImageData(X * 4, Y * 4, 4, 4).data
            //将获取到数据每4个一组,每组都是一个像素
            blockList[Y * 200 + X] = { position: [X, Y], color: [] }
            for (let i = 0; i < tempColorData.length; i += 4) {
                blockList[Y * 200 + X].color.push([
                    tempColorData[i],
                    tempColorData[i + 1],
                    tempColorData[i + 2],
                    tempColorData[i + 3],
                ])
            }
        }
    }
    blockMainColors = mostBlockColor(blockList)
}
//获取每个格子的主色调
const mostBlockColor = (blockList) => {
    let blockMainColors = []
    //所有颜色的平均值为主色调
    for (let i = 0; i < blockList.length; i++) {
        let r = 0,
            g = 0,
            b = 0,
            a = 0
        for (let j = 0; j < blockList[i].color[j].length; j++) {
            r += blockList[i].color[j][0]
            g += blockList[i].color[j][1]
            b += blockList[i].color[j][2]
            a += blockList[i].color[j][3]
        }

        // 求取平均值
        r /= blockList[i].color[0].length
        g /= blockList[i].color[0].length
        b /= blockList[i].color[0].length
        a /= blockList[i].color[0].length
        // 将最终的值取整
        r = Math.round(r)
        g = Math.round(g)
        b = Math.round(b)
        a = Math.round(a)
        blockMainColors.push({
            position: blockList[i].position,
            color: [r, g, b, a],
        })
    }
    return blockMainColors
}

onMounted(() => initCanvas())

// 获取文件
const limit = ref(1)
const isTarget = ref(true)
const imgList = []
const count = ref(0)

// 目标图片/素材图片选择回调
const slectFile = async file => {
    let tempUrl = window.URL.createObjectURL(file.raw)
    if(isTarget.value) {
        drawImage(tempUrl)
        limit.value = 10000
        isTarget.value = false
    } else {
        let image = await getAverageColor(tempUrl)
        imgList.push(image)
        count.value ++
    }
}
// 绘制目标图片
const drawImage = (url) => {
    fabric.Image.fromURL(url, (img) => {
        //设置缩放比例,长图的缩放比为this.canvas.width / img.width,宽图的缩放比为this.canvas.height / img.height
        let scale = img.height > img.width ? canvas.width / img.width : canvas.height / img.height
        img.set({
            left: canvas.width / 2, //距离左边的距离
            originX: 'center', //图片在原点的对齐方式
            originY: 'center',
            top: canvas.height / 2,
            scaleX: scale, //横向缩放
            scaleY: scale, //纵向缩放
            selectable: false, //可交互
        })
        img.on('added', (e) => {
            //这里有个问题,added后获取的是之前的画布像素数据,其他手动触发的事件,不会有这种问题
            //故用一个异步解决
            setTimeout(() => {
                getCanvasData()
            }, 500)
        })
        canvas.add(img) //将图片添加到画布
    })
}
//计算颜色差异
const colorDiff = (color1, color2) => {
    let d = 0
    for (let i = 0; i < color1.length; i++) {
        d += (color1[i] - color2[i]) ** 2
    }
    return Math.sqrt(d)
}


// 操作按钮
// 重置
const reload = () => {
    location.reload()
}

//生成图片
const generateImg = () => {
    let diffColorList = []
    //遍历所有方块
    for (let i = 0; i < blockMainColors.length; i++) {
        diffColorList[i] = { diffs: [] }
        //遍历所有图片
        for (let j = 0; j < imgList.length; j++) {
            diffColorList[i].diffs.push({
            url: imgList[j].url,
            diff: colorDiff(
                blockMainColors[i].color,
                imgList[j].color,
            ),
            color: imgList[j].color,
            })
        }
        //对比较过的图片进行排序,差异最小的放最前面
        diffColorList[i].diffs.sort((a, b) => {
            return a.diff - b.diff
        })
        //取第0个图片信息
        diffColorList[i].url = diffColorList[i].diffs[0].url
        diffColorList[i].position = blockMainColors[i].position
        diffColorList[i].Acolor = blockMainColors[i].color
        diffColorList[i].Bcolor = diffColorList[i].diffs[0].color
    }
    // 遍历每一个方块,对其渲染
    diffColorList.forEach((item) => {
        fabric.Image.fromURL(item.url, (img) => {
            let scale = img.height > img.width ? 4 / img.width : 4 / img.height
            img.set({
                left: item.position[0] * 4,
                top: item.position[1] * 4,
                originX: 'center',
                // originY: 'center',
                scaleX: scale,
                scaleY: scale,
            })
            canvas.add(img)
        })
    })
}

// 导出图片
const exportCanvas = () => {
    const dataURL = canvas.toDataURL({
        width: canvas.width,
        height: canvas.height,
        left: 0,
        top: 0,
        format: 'png',
    })
    const link = document.createElement('a')
    link.download = 'canvas.png'
    link.href = dataURL
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)
}

</script>

<style scoped>
.content {
    width: 1000px;
    display: flex;
    justify-content: space-around;
}
.cavans {
    border: 2px dotted #000;
}
.rightHandle {
    display: flex;
    flex-direction: column;
}
.rightHandle > .el-button {
    margin-top: 20px;
}
.rightHandle > .el-button+.el-button {
    margin-left: 0px;
}
:deep(.el-upload-list) {
    display: none;
}
</style>