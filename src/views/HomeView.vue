<template>
  <div>
    <div class="container">
      <canvas id="stars"></canvas>
      <div id="obstacle-container"></div>
      <div class="rabbit" :class="{ jumpRabbit: jumping }"></div>
      <div class="moon"></div>
      <div class="change">
        <img src="../images/change.gif" />
      </div>
    </div>
    <div v-show="state === 'end'" class="gameover">gameOver</div>
    <div class="models">
      <button :style="{ color: !model ? 'green' : '#000' }" @click="model = 0">
        原版模式(空格跳跃)
      </button>
      <button :style="{ color: model ? 'green' : '#000' }" @click="model = 1">
        声音模式（空格开启，声音跳跃）
      </button>
    </div>
  </div>
</template>

<script>
import "./HomeView.scss";
import { Star, MeteorRain } from "../utils";
export default {
  data() {
    return {
      state: "stop",
      context: null,
      starCount: 1600,
      rainCount: 25,
      rabbit: null,
      rains: [],
      arr: [],
      obstacle: null,
      obstacleContainer: null,
      timer: null,
      time: 0,
      jumping: false,
      jumpHeight: 0,
      jumpInterval: null,
      fallInterval: null,
      audioContext: null,
      audioStream: null,
      isLisening: false,
      model: 1, // 0:空格跳跃、1:声音跳跃
    };
  },
  mounted() {
    // 渲染星星背景
    this.renderStar();
    this.obstacleContainer = document.getElementById("obstacle-container");
    this.rabbit = document.getElementsByClassName("rabbit")[0];
    document.addEventListener("keydown", (event) => {
      if (event.code === "Space") {
        if (this.model) {
          this.voiceJump();
        } else {
          this.jump();
        }
      }
    });
  },
  methods: {
    // 声控模式
    voiceJump() {
      console.log("voiceJump", this.state);
      if (this.state === "playing") return;
      if (this.state !== "playing") this.start();
      // 使用Web浏览器的音频API获取麦克风输入
      navigator.mediaDevices
        .getUserMedia({ audio: true })
        .then((stream) => {
          console.log("voiceJump");
          this.audioStream = stream;
          if (this.isLisening) {
            return;
          } else {
            this.isLisening = true;
            this.audioContext = new AudioContext();
            const microphone =
              this.audioContext.createMediaStreamSource(stream);
            const analyser = this.audioContext.createAnalyser();

            microphone.connect(analyser);
            analyser.connect(this.audioContext.destination);

            analyser.fftSize = 256;
            const bufferLength = analyser.frequencyBinCount;
            const dataArray = new Uint8Array(bufferLength);

            analyser.getByteFrequencyData(dataArray);

            // 检测音量是否超过阈值，并触发跳跃
            const detectJump = () => {
              if (this.state !== "playing") {
                clearInterval(voiceInterval);
              }
              analyser.getByteFrequencyData(dataArray);
              const volume =
                dataArray.reduce((acc, val) => acc + val, 0) / bufferLength;
              console.log(volume > 50 ? volume : "");
              if (volume > 50) {
                // 调整此阈值以匹配声音强度
                if (this.state !== "playing") {
                  this.start();
                }
                this.jump();
              }
            };

            let voiceInterval = setInterval(detectJump, 100);
          }
        })
        .catch(function (err) {
          console.error("获取麦克风权限失败: ", err);
        });
    },

    jump() {
      if (this.state !== "playing") this.start();
      this.jumping = true;
      clearInterval(this.fallInterval);
      clearInterval(this.jumpInterval);
      this.jumpInterval = setInterval(() => {
        if (this.jumpHeight >= 280 || this.state !== "playing") {
          clearInterval(this.jumpInterval);
          this.rabbit.style.transform = `rotate(0deg) translateY(-${this.jumpHeight}px)`;
          this.fall();
        } else {
          this.jumpHeight += 10;
          this.rabbit.style.transform = `rotate(-10deg) translateY(-${this.jumpHeight}px)`;
        }
      }, 20);
    },
    fall() {
      this.fallInterval = setInterval(() => {
        if (this.jumpHeight === 0 || this.state !== "playing") {
          if (!this.jumpHeight) {
            this.rabbit.style.transform = `rotate(0deg) translateY(0px)`;
          }
          clearInterval(this.fallInterval);
          this.jumping = false;
        } else {
          this.rabbit.style.transform = `rotate(10deg) translateY(-${this.jumpHeight}px)`;
          this.jumpHeight -= 10;
        }
      }, 20);
    },
    judge() {
      const domList = document.getElementsByClassName("obstacle");
      const len = domList.length;
      let isOverlap = false;
      for (let i = 0; i < len; i++) {
        isOverlap = this.isOverlap(this.rabbit, domList[i]);
        if (isOverlap) return isOverlap;
      }
      return isOverlap;
    },
    isOverlap(element1, element2) {
      const rect1 = element1.getBoundingClientRect();
      const rect2 = element2.getBoundingClientRect();
      // 判断是否有交叠
      const overlapX = rect1.left < rect2.right && rect1.right > rect2.left;
      const overlapY = rect1.top < rect2.bottom && rect1.bottom > rect2.top;

      return overlapX && overlapY;
    },
    gameOver() {
      console.log("gameover");
      if (this.audioStream) {
        const tracks = this.audioStream.getTracks();
        tracks.forEach((track) => track.stop());
      }
      this.audioContext && this.audioContext.close && this.audioContext.close();
      this.isLisening = false;
      this.state = "end";
    },
    start() {
      if (this.state == "playing") return;
      this.obstacleContainer.innerHTML = "";
      this.state = "playing";
      this.startGame();
      const gameObserver = () => {
        if (this.judge()) {
          clearTimeout(this.timer);
          this.timer = null;
          return this.gameOver();
        } else {
          window.requestAnimationFrame(gameObserver);
        }
      };
      window.requestAnimationFrame(gameObserver);
    },
    startGame() {
      if (this.state !== "playing") return;
      // 3s内随机生成月饼堆
      this.timer = setTimeout(() => {
        if (this.state !== "playing") return;
        this.createObstacle();
        this.time = Math.random() * 2 + 1;
        this.startGame();
      }, this.time * 1000);
    },

    // 创建月饼堆
    createObstacle() {
      const obstacle = document.createElement("div");
      obstacle.classList.add("obstacle");

      // 设置月饼堆的初始位置，这里的Math.random()用于生成随机位置
      const num = Math.ceil(
        (Math.random() * (this.obstacleContainer.clientHeight - 20) + 20) / 25
      );
      for (let i = 0; i < num; i++) {
        const mooncake = document.createElement("div");
        mooncake.classList.add("mooncake");
        mooncake.style.zIndex = num - i;
        mooncake.id = "mooncake" + i;
        obstacle.appendChild(mooncake);
      }

      obstacle.style.right = 0; // 初始位置在容器的右侧

      // 将月饼堆添加到容器中
      this.obstacleContainer.appendChild(obstacle);

      // 让月饼堆向左移动
      this.moveObstacle(obstacle);
    },
    moveObstacle(obstacle) {
      const obstacleSpeed = 3; // 月饼堆移动速度，可以根据需要调整
      const moveInterval = () => {
        if (this.state !== "playing") return;
        const obstacleRight = parseInt(obstacle.style.right);
        // 移动月饼堆
        obstacle.style.right = obstacleRight + obstacleSpeed + "px";
        // 如果月饼堆移出容器，则移除它并创建新的月饼堆
        if (obstacleRight > this.obstacleContainer.clientWidth) {
          obstacle.remove(); // 移除月饼堆
        } else {
          window.requestAnimationFrame(moveInterval);
        }
      };
      moveInterval();
    },
    // 绘制星星背景
    renderStar() {
      var stars = document.getElementById("stars");
      stars.width = window.innerWidth;
      stars.height = window.innerHeight;
      this.context = stars.getContext("2d");
      for (var i = 0; i < this.starCount; i++) {
        var star = new Star(this.context);
        star.init();
        star.draw();
        this.arr.push(star);
      }
      for (var i = 0; i < this.rainCount; i++) {
        var rain = new MeteorRain(this.context);
        rain.init();
        rain.draw();
        this.rains.push(rain);
      }
      this.playStars(); //绘制闪动的星星
      this.playRains(); //绘制流星
    },
    playStars() {
      for (var n = 0; n < this.starCount; n++) {
        this.arr[n].getColor();
        this.arr[n].draw();
      }
      window.requestAnimationFrame(this.playStars);
    },
    playRains() {
      for (var n = 0; n < this.rainCount; n++) {
        var rain = this.rains[n];
        rain.move(); //移动
        if (rain.y > window.innerHeight) {
          //超出界限后重来
          this.context.clearRect(
            rain.x,
            rain.y - rain.height,
            rain.width,
            rain.height
          );
          this.rains[n] = new MeteorRain(this.context);
          this.rains[n].init();
        }
      }
      window.requestAnimationFrame(this.playRains);
    },
  },
};
</script>
