<template>
  <div>
    <div ref="container">
      <canvas
        style="width: 100%; height: 100vh"
        :width="width"
        :height="height"
        ref="canvas"
      >
      </canvas>
    </div>
  </div>
</template>

<script>
import Konva from "konva";

export default {
  props: {
    backgroundImage: {
      type: String,
      required: true,
    },
    drawingImage: {
      type: String,
      required: false,
    },
    mode: {
      type: String,
      required: true,
    },
    color: {
      type: String,
      required: true,
    },
    lineWidth: {
      type: Number,
      required: true,
    },
  },

  data() {
    return {
      width: 0,
      height: 0,
      stage: null,
      canvas: null,
      context: null,
      drawingLayer: null,
      drawingScope: null,
      lastPointerPosition: {},
      localPos: {
        x: 0,
        y: 0,
      },
      pos: null,
      isPaint: false,
      imageObj: null,
      backgroundLayer: null,
      backgroundImageScope: null,
      undoDataStack: [],
      redoDataStack: [],
      scale: 1,
      scaleWidth: 1,
      initialDistance: null,
      initialPointerPositions: null,
      created: false,
    };
  },

  watch: {
    backgroundImageTemp: {
      immediate: true,
      handler(newVal) {
        this.$nextTick(() => {
          this.setting(newVal);
        });
      },
    },
  },

  methods: {
    async setting() {
      await this.resizeBackgroundImage();
      await this.drowSetting();
    },

    async resizeBackgroundImage() {
      const maxContainerWidth = this.$refs.container.offsetWidth;

      try {
        const blob = await fetch(this.backgroundImage).then((response) =>
          response.blob()
        );
        const image = await createImageBitmap(blob);

        this.imageAspectRatio = image.width / image.height;
        let newWidth = maxContainerWidth;
        let newHeight = maxContainerWidth / this.imageAspectRatio;

        if (newWidth > maxContainerWidth) {
          newWidth = maxContainerWidth;
          newHeight = newWidth / this.imageAspectRatio;
        }

        this.width = newWidth;
        this.height = newHeight;
      } catch (error) {
        console.error(error);
      }
    },

    async drowSetting() {
      var container = this.$refs.container;
      this.stage = new Konva.Stage({
        container,
        width: this.width,
        height: this.height,
      });
      this.drawingLayer = new Konva.Layer();
      this.stage.add(this.drawingLayer);
      this.canvas = this.$refs.canvas;
      this.drawingScope = new Konva.Image({
        image: this.canvas,
        x: 0,
        y: 0,
        stroke: "black",
      });

      this.drawingLayer.add(this.drawingScope);
      this.stage.draw();
      this.context = this.canvas.getContext("2d");
      this.context.lineJoin = "round";
      this.context.strokeStyle = this.color;
      this.context.lineWidth = this.lineWidth;
      this.context.mode = this.mode;

      // イベント追加
      this.stage.addEventListener("mousedown", this.mousedown);
      this.stage.addEventListener("mouseup", this.mouseup);
      this.stage.addEventListener("mousemove", this.mousemove);
      this.stage.addEventListener("touchstart", this.mousedown);
      this.stage.addEventListener("touchend", this.mouseup);
      this.stage.addEventListener("touchmove", this.mousemove);
      this.stage.addEventListener("wheel", this.onMouseWheel);

      this.imageObj = new Image();
      this.imageObj.addEventListener("load", this.imageOnload);
      this.imageObj.src = this.backgroundImage;

      this.loadOperations(this.drawingImage);
    },

    async loadOperations(operationsData) {
      return new Promise((resolve, reject) => {
        const image = new Image();
        image.src = operationsData;
        image.onload = () => {
          const canvas = document.createElement("canvas");
          canvas.width = this.width;
          canvas.height = this.height;
          const context = canvas.getContext("2d");
          const scale = Math.min(
            canvas.width / image.width,
            canvas.height / image.height
          );
          const scaledWidth = image.width * scale;
          const scaledHeight = image.height * scale;
          const offsetX = (canvas.width - scaledWidth) / 2;
          const offsetY = (canvas.height - scaledHeight) / 2;
          context.drawImage(image, offsetX, offsetY, scaledWidth, scaledHeight);
          this.context.drawImage(canvas, 0, 0);
          this.drawingLayer.draw();
          resolve();
        };

        image.onerror = reject;
      });
    },

    imageOnload: async function () {
      // 背景レイヤ
      this.backgroundLayer = new Konva.Layer();

      // 背景イメージ（xとy座標はthis.drawingScopeと同じにする）
      this.backgroundImageScope = new Konva.Image({
        image: this.imageObj,
        x: 0,
        y: 0,
        width: this.canvas.width,
        height: this.canvas.height,
      });

      // 背景レイヤに背景イメージを追加
      this.backgroundLayer.add(this.backgroundImageScope);
      this.stage.add(this.backgroundLayer);

      // 背景イメージを最背面に移動。これをしないとペンの描画が画像の下に潜ってしまう。
      this.backgroundLayer.moveToBottom();
    },

    onMouseWheel(event) {
      event.preventDefault();

      const stage = this.stage;
      const oldScale = stage.scaleX();
      const pointerPos = stage.getPointerPosition();

      const mousePos = {
        x: (pointerPos.x - stage.x()) / oldScale,
        y: (pointerPos.y - stage.y()) / oldScale,
      };

      const scaleBy = 1.05;
      const newScale =
        event.deltaY > 0 ? oldScale / scaleBy : oldScale * scaleBy;

      const minScale = 1; // 最小スケール
      const maxScale = 10; // 最大スケール

      const newPos = {
        x: -(mousePos.x - pointerPos.x / newScale) * newScale,
        y: -(mousePos.y - pointerPos.y / newScale) * newScale,
      };

      if (newScale < minScale) {
        // 縮小した場合に最小スケールを原寸大に調整
        stage.scale({ x: minScale, y: minScale });
        stage.position({ x: 0, y: 0 });
      } else if (newScale > maxScale) {
        // 最大スケールを超える場合に最大スケールに調整
        stage.scale({ x: maxScale, y: maxScale });
        stage.position(newPos);
      } else {
        stage.scale({ x: newScale, y: newScale });
        stage.position(newPos);
      }

      stage.batchDraw();

      this.scale = stage.scaleX();
    },

    mousedown: function (event) {
      if (event.touches && event.touches.length === 2) {
        this.onTwoTouchStart(event);
        return;
      }

      this.initialDistance = null;
      this.isPaint = true;
      this.lastPointerPosition = this.stage.getPointerPosition();
      const currentImageData = this.context.getImageData(
        0,
        0,
        this.canvas.width,
        this.canvas.height
      );

      if (
        this.undoDataStack.length === 0 ||
        !this.isImageDataEqual(
          this.undoDataStack[this.undoDataStack.length - 1],
          currentImageData
        )
      ) {
        this.undoDataStack.push(currentImageData);
      }
    },

    isImageDataEqual: function (imgData1, imgData2) {
      if (
        imgData1.width !== imgData2.width ||
        imgData1.height !== imgData2.height
      ) {
        return false;
      }

      for (let i = 0; i < imgData1.data.length; i++) {
        if (imgData1.data[i] !== imgData2.data[i]) {
          return false;
        }
      }

      return true;
    },

    onTwoTouchStart: function (event) {
      // タッチ操作の初期距離を保存
      const touch1 = event.touches[0];
      const touch2 = event.touches[1];
      const distance = Math.sqrt(
        (touch1.clientX - touch2.clientX) ** 2 +
          (touch1.clientY - touch2.clientY) ** 2
      );
      this.initialDistance = distance;

      // タッチ操作の初期位置を保存
      this.initialPointerPositions = [
        { x: touch1.clientX, y: touch1.clientY },
        { x: touch2.clientX, y: touch2.clientY },
      ];

      // タッチ操作の初期位置をステージ座標系に変換
      this.initialStagePositions = this.initialPointerPositions.map(
        (pointerPos) => {
          return this.stage.getPointerPosition(pointerPos);
        }
      );
    },

    mousemove: function (event) {
      if (event.touches && event.touches.length === 2) {
        this.onTwoTouchMove(event);
        return;
      }

      if (!this.isPaint) {
        return;
      }
      // ペンモード
      if (this.isTargetMode("line")) {
        this.context.globalCompositeOperation = "source-over";
      }
      // 消しゴムモード
      if (this.isTargetMode("eraser")) {
        this.context.globalCompositeOperation = "destination-out";
      }

      this.context.beginPath();

      let stagePos = this.stage.position();
      let stageScale = this.stage.scale();

      this.localPos.x =
        (this.lastPointerPosition.x - this.drawingScope.x() - stagePos.x) /
        stageScale.x;
      this.localPos.y =
        (this.lastPointerPosition.y - this.drawingScope.y() - stagePos.y) /
        stageScale.y;

      // 描画開始座標を指定する
      this.context.moveTo(this.localPos.x, this.localPos.y);

      this.pos = this.stage.getPointerPosition();

      this.localPos.x =
        (this.pos.x - this.drawingScope.x() - stagePos.x) / stageScale.x;
      this.localPos.y =
        (this.pos.y - this.drawingScope.y() - stagePos.y) / stageScale.y;

      // 描画開始座標から、lineToに指定された座標まで描画する
      this.context.lineTo(this.localPos.x, this.localPos.y);
      this.context.closePath();
      this.context.stroke();
      this.drawingLayer.draw();

      this.lastPointerPosition = this.pos;
    },

    onTwoTouchMove: function (event) {
      const touch1 = event.touches[0];
      const touch2 = event.touches[1];
      const currentDistance = Math.sqrt(
        (touch1.clientX - touch2.clientX) ** 2 +
          (touch1.clientY - touch2.clientY) ** 2
      );

      const currentPointerPositions = [
        { x: touch1.clientX, y: touch1.clientY },
        { x: touch2.clientX, y: touch2.clientY },
      ];
      const middlePos = {
        x: (currentPointerPositions[0].x + currentPointerPositions[1].x) / 2,
        y: (currentPointerPositions[0].y + currentPointerPositions[1].y) / 2,
      };
      const oldMiddlePos = {
        x:
          (this.initialPointerPositions[0].x +
            this.initialPointerPositions[1].x) /
          2,
        y:
          (this.initialPointerPositions[0].y +
            this.initialPointerPositions[1].y) /
          2,
      };

      const scaleDiff = (currentDistance - this.initialDistance) * 0.003;
      const newScale = this.scale + scaleDiff;
      const clampedScale = Math.min(5, Math.max(1, newScale));

      let newOffsetX =
        this.stage.x() +
        (middlePos.x - oldMiddlePos.x) + //位置
        ((this.stage.x() - middlePos.x) * scaleDiff) / clampedScale; //スケール
      let newOffsetY =
        this.stage.y() +
        (middlePos.y - oldMiddlePos.y) + //位置
        ((this.stage.y() - middlePos.y) * scaleDiff) / clampedScale; //スケール

      // Constrain stage to its boundaries
      newOffsetX = Math.min(newOffsetX, 0);
      newOffsetX = Math.max(
        newOffsetX,
        -this.stage.width() * clampedScale + this.stage.width()
      );

      newOffsetY = Math.min(newOffsetY, 0);
      newOffsetY = Math.max(
        newOffsetY,
        -this.stage.height() * clampedScale + this.stage.height()
      );

      this.stage.scale({ x: clampedScale, y: clampedScale });
      this.stage.position({ x: newOffsetX, y: newOffsetY });

      this.initialDistance = currentDistance;
      this.initialPointerPositions = currentPointerPositions;
      this.scale = clampedScale;

      this.stage.batchDraw();
    },

    mouseup: function (event) {
      if (event.touches && event.touches.length !== 2) {
        // タッチ操作の終了時に初期値をリセット
        this.initialDistance = null;
        this.initialPointerPositions = null;
        this.initialStagePositions = null;
      }
      this.isPaint = false;
    },

    // 現在のモードが指定されたモードと一致するかどうか
    isTargetMode: function (targetMode) {
      return this.mode === targetMode;
    },

    onUndo: function () {
      if (this.undoDataStack.length <= 0) {
        return;
      }

      this.redoDataStack.push(
        this.context.getImageData(0, 0, this.canvas.width, this.canvas.height)
      );

      // 元に戻す（戻す配列からキャンバスの状態を取り出して反映）
      this.context.putImageData(this.undoDataStack.pop(), 0, 0);
      this.drawingLayer.draw();
    },

    onRedo: function () {
      if (this.redoDataStack.length <= 0) {
        return;
      }

      // やり直す前の状態を戻す配列に保存
      this.undoDataStack.push(
        this.context.getImageData(0, 0, this.canvas.width, this.canvas.height)
      );

      // やり直す（やり直し配列からキャンバスの状態を取り出して反映）
      this.context.putImageData(this.redoDataStack.pop(), 0, 0);
      this.drawingLayer.draw();
    },

    onClearCanvas: function () {
      this.undoDataStack.push(
        this.context.getImageData(0, 0, this.canvas.width, this.canvas.height)
      );

      this.context.globalCompositeOperation = "destination-out";
      this.context.fillRect(0, 0, this.width, this.height);
      this.drawingLayer.draw();
    },
  },
};
</script>
