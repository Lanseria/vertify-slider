<template>
  <div
    class="vertifyWrap"
    :style="{
      width: width + 'px',
      margin: '0 auto',
      display: visible ? '' : 'none',
    }"
    @mousemove="handleDragMove"
    @mouseup="handleDragEnd"
    @touchmove="handleDragMove"
    @touchend="handleDragEnd"
  >
    <div class="canvasArea">
      <canvas ref="canvasRef" :width="width" :height="height"></canvas>
      <canvas
        ref="blockRef"
        class="block"
        :width="width"
        :height="height"
        @mousedown="handleDragStart"
        @touchstart="handleDragStart"
      ></canvas>
    </div>
    <div
      :class="sliderClass"
      :style="{
        pointerEvents: isLoading ? 'none' : 'auto',
        width: width + 'px',
      }"
    >
      <div class="sliderMask" :style="{ width: sliderLeft + 'px' }">
        <div
          class="slider"
          :style="{ left: sliderLeft + 'px' }"
          @mousedown="handleDragStart"
          @touchstart="handleDragStart"
        >
          <div class="sliderIcon">&rarr;</div>
        </div>
      </div>
      <div class="sliderText">{{ textTip }}</div>
    </div>
    <div
      class="refreshIcon"
      @click="handleRefresh"
      :style="{ backgroundImage: `url(${refreshIcon})` }"
    ></div>
    <div
      class="loadingContainer"
      :style="{
        width: width + 'px',
        height: height + 'px',
        display: isLoading ? '' : 'none',
      }"
    >
      <div class="loadingIcon"></div>
      <span>加载中...</span>
    </div>
  </div>
</template>
<script lang="ts">
import { defineComponent, PropType, ref, watchEffect } from "vue";
import { getRandomNumberByRange, square, sum } from "../../utils";
const Verify = defineComponent({
  name: "Verify",
  components: {},
  props: {
    width: {
      type: Number,
      default: 320,
    },
    height: {
      type: Number,
      default: 160,
    },
    l: {
      type: Number,
      default: 42,
    },
    r: {
      type: Number,
      default: 9,
    },
    imgUrl: {
      type: String,
      required: true,
    },
    text: {
      type: String,
      required: true,
    },
    refreshIcon: {
      type: String,
      default: "http://cdn.dooring.cn/dr/icon12.png",
    },
    visible: {
      type: Boolean,
      default: true,
    },
    onSuccess: {
      type: Function as PropType<Fn>,
    },
    onFail: {
      type: Function as PropType<Fn>,
    },
    onRefresh: {
      type: Function as PropType<Fn>,
    },
  },
  setup(props, { emit }) {
    const isLoading = ref(false);
    const sliderLeft = ref(0);
    const sliderClass = ref("sliderContainer");
    const textTip = ref(props.text);

    const canvasRef = ref<HTMLCanvasElement>();
    const blockRef = ref<HTMLCanvasElement>();
    const imgRef = ref<HTMLImageElement>();

    const isMouseDownRef = ref<boolean>(false);
    const trailRef = ref<number[]>([]);
    const originXRef = ref<number>(0);
    const originYRef = ref<number>(0);
    const xRef = ref<number>(0);
    const yRef = ref<number>(0);
    const PI = Math.PI;
    const L = props.l + props.r * 2 + 3; // 滑块实际边长

    const drawPath = (
      ctx: any,
      x: number,
      y: number,
      operation: "fill" | "clip"
    ) => {
      const l = props.l;
      const r = props.r;
      ctx.beginPath();
      ctx.moveTo(x, y);
      ctx.arc(x + l / 2, y - r + 2, r, 0.72 * PI, 2.26 * PI);
      ctx.lineTo(x + l, y);
      ctx.arc(x + l + r - 2, y + l / 2, r, 1.21 * PI, 2.78 * PI);
      ctx.lineTo(x + l, y + l);
      ctx.lineTo(x, y + l);
      ctx.arc(x + r - 2, y + l / 2, r + 0.4, 2.76 * PI, 1.24 * PI, true);
      ctx.lineTo(x, y);
      ctx.lineWidth = 2;
      ctx.fillStyle = "rgba(255, 255, 255, 0.7)";
      ctx.strokeStyle = "rgba(255, 255, 255, 0.7)";
      ctx.stroke();
      ctx.globalCompositeOperation = "destination-over";
      operation === "fill" ? ctx.fill() : ctx.clip();
    };

    const getRandomImgSrc = () => {
      return (
        props.imgUrl ||
        `https://picsum.photos/id/${getRandomNumberByRange(0, 1084)}/${
          props.width
        }/${props.height}`
      );
    };

    const createImg = (onload: VoidFunction) => {
      const img = new Image();
      img.crossOrigin = "Anonymous";
      img.onload = onload;
      img.onerror = () => {
        (img as any).setSrc(getRandomImgSrc()); // 图片加载失败的时候重新加载其他图片
      };

      (img as any).setSrc = (src: string) => {
        const isIE = window.navigator.userAgent.indexOf("Trident") > -1;
        if (isIE) {
          // IE浏览器无法通过img.crossOrigin跨域，使用ajax获取图片blob然后转为dataURL显示
          const xhr = new XMLHttpRequest();
          xhr.onloadend = function (e: any) {
            const file = new FileReader(); // FileReader仅支持IE10+
            file.readAsDataURL(e.target.response);
            file.onloadend = function (e) {
              img.src = e?.target?.result as string;
            };
          };
          xhr.open("GET", src);
          xhr.responseType = "blob";
          xhr.send();
        } else img.src = src;
      };

      (img as any).setSrc(getRandomImgSrc());
      return img;
    };

    const draw = (img: HTMLImageElement) => {
      const width = props.width;
      const height = props.height;
      const r = props.r;
      const canvasCtx = canvasRef.value!.getContext("2d");
      const blockCtx = blockRef.value!.getContext("2d");
      // 随机位置创建拼图形状
      xRef.value = getRandomNumberByRange(L + 10, width - (L + 10));
      yRef.value = getRandomNumberByRange(10 + r * 2, height - (L + 10));
      drawPath(canvasCtx, xRef.value, yRef.value, "fill");
      drawPath(blockCtx, xRef.value, yRef.value, "clip");

      // 画入图片
      canvasCtx!.drawImage(img, 0, 0, width, height);
      blockCtx!.drawImage(img, 0, 0, width, height);

      // 提取滑块并放到最左边
      const y1 = yRef.value - r * 2 - 1;
      const ImageData = blockCtx!.getImageData(xRef.value - 3, y1, L, L);
      blockRef.value!.width = L;
      blockCtx!.putImageData(ImageData, 0, y1);
    };

    const initImg = () => {
      const img = createImg(() => {
        isLoading.value = false;
        draw(img);
      });
      imgRef.value = img;
    };

    const reset = () => {
      const width = props.width;
      const height = props.height;
      const canvasCtx = canvasRef.value!.getContext("2d");
      const blockCtx = blockRef.value!.getContext("2d");
      // 重置样式
      sliderLeft.value = 0;
      sliderClass.value = "sliderContainer";
      blockRef.value!.width = width;
      blockRef.value!.style.left = 0 + "px";

      // 清空画布
      canvasCtx!.clearRect(0, 0, width, height);
      blockCtx!.clearRect(0, 0, width, height);

      // 重新加载图片
      isLoading.value = true;
      imgRef.value!.src = getRandomImgSrc();
    };
    const handleRefresh = () => {
      reset();
      typeof props.onRefresh === "function" && props.onRefresh();
    };

    const verify = () => {
      const arr = trailRef.value; // 拖动时y轴的移动距离
      const average = arr.reduce(sum) / arr.length;
      const deviations = arr.map((x) => x - average);
      const stddev = Math.sqrt(deviations.map(square).reduce(sum) / arr.length);
      const left = parseInt(blockRef.value!.style.left);
      return {
        spliced: Math.abs(left - xRef.value) < 10,
        verified: stddev !== 0, // 简单验证拖动轨迹，为零时表示Y轴上下没有波动，可能非人为操作
      };
    };

    const handleDragStart = function (e: any) {
      originXRef.value = e.clientX || e.touches[0].clientX;
      originYRef.value = e.clientY || e.touches[0].clientY;
      isMouseDownRef.value = true;
    };

    const handleDragMove = (e: any) => {
      const width = props.width;
      if (!isMouseDownRef.value) return false;
      e.preventDefault();
      const eventX = e.clientX || e.touches[0].clientX;
      const eventY = e.clientY || e.touches[0].clientY;
      const moveX = eventX - originXRef.value;
      const moveY = eventY - originYRef.value;
      if (moveX < 0 || moveX + 38 >= width) return false;
      sliderLeft.value = moveX;
      const blockLeft = ((width - 40 - 20) / (width - 40)) * moveX;
      blockRef.value!.style.left = blockLeft + "px";
      sliderClass.value = "sliderContainer sliderContainer_active";
      trailRef.value.push(moveY);
    };

    const handleDragEnd = (e: any) => {
      if (!isMouseDownRef.value) return false;
      isMouseDownRef.value = false;
      const eventX = e.clientX || e.changedTouches[0].clientX;
      if (eventX === originXRef.value) return false;
      sliderClass.value = "sliderContainer";
      const { spliced, verified } = verify();
      if (spliced) {
        if (verified) {
          sliderClass.value = "sliderContainer sliderContainer_success";
          typeof props.onSuccess === "function" && props.onSuccess();
        } else {
          sliderClass.value = "sliderContainer sliderContainer_fail";
          textTip.value = "请再试一次";
          reset();
        }
      } else {
        sliderClass.value = "sliderContainer sliderContainer_fail";
        typeof props.onFail === "function" && props.onFail();
        setTimeout(reset.bind(this), 1000);
      }
    };

    watchEffect(() => {
      if (props.visible) {
        imgRef.value ? reset() : initImg();
      }
    });

    return {
      sliderLeft,
      sliderClass,
      textTip,
      isLoading,

      canvasRef,
      blockRef,

      handleRefresh,
      handleDragStart,
      handleDragMove,
      handleDragEnd,
    };
  },
});
export default Verify;
export type VerifyRefs = InstanceType<typeof Verify>;
</script>
<style lang="less" scoped>
.vertifyWrap {
  position: relative;

  .block {
    position: absolute;
    left: 0;
    top: 0;
    cursor: pointer;
    cursor: grab;
  }

  .block:active {
    cursor: grabbing;
  }

  .sliderContainer {
    position: relative;
    text-align: center;
    width: 310px;
    height: 40px;
    line-height: 40px;
    margin-top: 15px;
    background: #f7f9fa;
    color: #45494c;
    border: 1px solid #e4e7eb;
  }

  .sliderContainer_active .slider {
    height: 38px;
    top: -1px;
    border: 1px solid #486cd6;
  }

  .sliderContainer_active .sliderMask {
    height: 38px;
    border-width: 1px;
  }

  .sliderContainer_success .slider {
    height: 38px;
    top: -1px;
    border: 1px solid #0db87f;
    background-color: #0ca14a !important;
  }

  .sliderContainer_success .sliderMask {
    height: 38px;
    border: 1px solid #0db87f;
    background-color: #d2f4ef;
  }

  .sliderContainer_success .sliderIcon {
    background-position: 0 -26px !important;
  }

  .sliderContainer_fail .slider {
    height: 38px;
    top: -1px;
    border: 1px solid #f57a7a;
    background-color: #f57a7a !important;
  }

  .sliderContainer_fail .sliderMask {
    height: 38px;
    border: 1px solid #f57a7a;
    background-color: #fce1e1;
  }

  .sliderContainer_fail .sliderIcon {
    top: 14px;
    background-position: 0 -82px !important;
  }

  .sliderContainer_active .sliderText,
  .sliderContainer_success .sliderText,
  .sliderContainer_fail .sliderText {
    display: none;
  }

  .sliderMask {
    position: absolute;
    left: 0;
    top: 0;
    height: 40px;
    border: 0 solid #486cd6;
    background: #d1e9fe;
  }

  .slider {
    position: absolute;
    top: 0;
    left: 0;
    width: 40px;
    height: 40px;
    background: #fff;
    box-shadow: 0 0 3px rgba(0, 0, 0, 0.3);
    transition: background 0.2s linear;
    cursor: pointer;
    cursor: grab;
  }

  .slider:active {
    cursor: grabbing;
  }

  .slider:hover {
    background: #486cd6;
  }

  .sliderIcon {
    font-size: 18px;
    color: #000;
  }

  .slider:hover .sliderIcon {
    color: #fff;
  }

  .refreshIcon {
    position: absolute;
    right: 5px;
    top: 5px;
    width: 30px;
    height: 30px;
    cursor: pointer;
    background-size: 32px;
  }

  .loadingContainer {
    position: absolute;
    left: 0;
    top: 0;
    width: 310px;
    height: 155px;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    font-size: 14px;
    color: #45494c;
    z-index: 2;
    background: #edf0f2;
  }

  .loadingIcon {
    width: 32px;
    height: 32px;
    margin-bottom: 10px;
    background: url(http://cdn.dooring.cn/dr/icon12.png);
    background-size: 32px;
    animation: loading-icon-rotate 0.8s linear infinite;
  }

  @keyframes loading-icon-rotate {
    from {
      transform: rotate(0);
    }
    to {
      transform: rotate(360deg);
    }
  }
}
</style>
