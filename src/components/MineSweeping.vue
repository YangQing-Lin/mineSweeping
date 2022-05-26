<template>
  <div class="main">
    <!-- 1功能区 -->
    <div class="tool-content-t">
      <div :class="['mark-btn', isMarkStatus ? 'marked' : '']" @click="changeIsMarkStatus">
        标记🚩
      </div>
      <div class="mark-btn right">计时：{{ time }} s</div>
    </div>
    <!-- 中心游戏区 -->
    <div class="game-content">
      <div v-for="col in cols" :key="Math.random() + col" class="game-content-row">
        <span v-for="row in rows" :key="Math.random() + row" class="game-block" :class="[
          lattice[(col - 1) * rows + row - 1].isOpen ? 'open' : '',
          lattice[(col - 1) * rows + row - 1].isMark ? 'mark' : '',
        ]" 
        @click.left="handleClickLattice(lattice[(col - 1) * rows + row - 1])" 
        @click.right.prevent="handleSureMinePoint(lattice[(col - 1) * rows + row - 1])"
        >
          <template v-if="over === 1">
            <span v-if="lattice[(col - 1) * rows + row - 1].isMine">💣</span>
            <span v-else>{{
                lattice[(col - 1) * rows + row - 1].mineNum
            }}</span>
          </template>
          <template v-else>
            <span v-if="lattice[(col - 1) * rows + row - 1].isMark">🚩</span>
            <span v-else>{{
                lattice[(col - 1) * rows + row - 1].mineNum
            }}</span>
          </template>
        </span>
      </div>
    </div>
    <!-- 2功能区 -->
    <div class="tool-content">
      <div>剩余雷数：{{ minePosition.length }}</div>
      <div class="tool">
        <div @click="reStart">重开一局</div>
        <div @click="changeLevel">改变难度</div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: "mineSweeping",
  components: {},
  props: {
    // 是否展示游戏盘
    showGame: {
      required: true,
      type: Boolean,
    },
    // 游戏盘格子数和雷数
    gameInfo: {
      required: true,
      type: Array,
    },
  },
  data() {
    return {
      // 横排格子数
      rows: 0,
      // 纵排格子数
      cols: 0,
      // 格子总数
      latticeNum: 0,
      // 雷点位置（找出一个少一个）
      minePosition: [],
      // 雷点位置备个份
      minePositionBake: [],
      // 格子属性
      lattice: [
        {
          index: 0, // 格子索引
          mineNum: 0, // 周围雷数
          isMine: false, // 是否是雷
          isOpen: false, // 是否已经被点开
          isMark: false, // 是否被标记
        },
      ],
      // 游戏是否结束： 0-未结束 1-点到雷了导致结束  2-雷点被标记完了
      over: 0,
      // 是否处于标记状态（该状态用于移动端替代PC端右键标记地雷事件）
      isMarkStatus: false,
      // 游戏计时
      time: 0,
      // 游戏计时器
      interval: null,
    };
  },
  computed: {
    timeText() {
      return this.getTimeText(this.time);
    },
  },
  watch: {
    over(newVal) {
      switch (newVal) {
        case 1:
          // this.openAllMineLattice();
          this.openAllRest();
          alert("BOOM！！！");
          break;
        case 2:
          const wrongMark = this.judgeWrongMark();
          if (wrongMark) {
            setTimeout(() => {
              alert(`你有${wrongMark}个地雷标记错误`);
            }, 500);
          } else {
            this.openAllRest();
            alert("优秀，找出了全部的🚩");
          }
          break;
        default:
          break;
      }
      if (newVal) {
        window.clearInterval(this.interval);
      }
    },
  },
  created() {
    this.init();
  },
  mounted() {
    this.$nextTick(() => {
      this.setTime();
    });
  },
  methods: {
    // 初始化格子盘隐藏数据
    init() {
      Object.assign(this.$data, this.$options.data());
      // 父组件带过来的格子数和雷数
      this.rows = this.gameInfo[0]; // x轴（列数）
      this.cols = this.gameInfo[1]; // y轴（行数）
      this.latticeNum = this.rows * this.cols;
      // 获取雷点位置
      this.getMinePosition();
      // 给每个格子赋予正确的属性
      this.initLattice();
    },
    // 游戏开始计时
    setTime() {
      this.interval = setInterval(() => {
        this.time++;
      }, 1000);
    },
    // 获取时间
    getTimeText(time) { },
    // 改变标记状态
    changeIsMarkStatus() {
      this.isMarkStatus = !this.isMarkStatus;
    },
    // 随机获取雷点位置
    getMinePosition() {
      // 定义一个数组装不重复的格点
      let mineArr = [];
      // 循环雷数生成不重复的雷点（生成this.gameInfo[2]个雷）
      for (let n = 0; n < this.gameInfo[2]; n++) {
        const random = Math.floor(Math.random() * this.latticeNum);
        if (mineArr.indexOf(random) === -1) {
          mineArr.push(random);
        } else {
          n--;
        }
      }
      this.minePosition = mineArr;
      // 生成一个内容相同但指针不同的数组
      this.minePositionBake = [].concat(mineArr);
    },
    // 格子属性初始化
    initLattice() {
      // 格子列表
      let latticeArr = [];
      for (let n = 0; n < this.latticeNum; n++) {
        let lattice = {
          index: n,
          isOpen: false,
          mineNum: 0,
          isMark: false,
        };
        // n标记是否是雷（雷：true 非雷：false）
        lattice.isMine = this.minePosition.indexOf(n) > -1;
        // 如果不是雷，计算出格子周围8个点的雷数
        if (!lattice.isMine) {
          this.getMineNumAroundLattice(lattice, n);
        }
        latticeArr.push(lattice);
      }
      this.lattice = latticeArr;
    },
    // 打开所有剩下的格子
    openAllRest() {
      this.lattice.forEach((item) => {
        item.isOpen = true;
      });
    },
    // 仅打开所有是雷的格子
    openAllMineLattice() {
      this.lattice.forEach((item) => {
        if (item.isMine) {
          item.isOpen = true;
        }
      });
    },
    // 获取格子周围的雷数
    getMineNumAroundLattice(lattice, index) {
      // 先获取格子周围的有效索引
      const latticeIndexArr = this.getLatticeIndex(index);
      // 循环索引，索引值在雷点数组中的，即为雷，当前格子的雷点数加1
      latticeIndexArr.forEach((i) => {
        if (this.minePosition.indexOf(i) > -1) {
          lattice.mineNum++;
        }
      });
    },
    // 获取格子周围的有效索引
    getLatticeIndex(index) {
      // 0做索引不好算，按正常数字来算
      index++;
      // 存索引值的变量
      let latticeIndexArr = [];
      // 当前格子位于第几行
      const latticeRow = Math.ceil(index / this.rows);
      // 当前格子位于第几列（求余为0说明是最右边一列）**
      const latticeCol = Math.ceil(index % this.rows) || this.rows;
      // 上方向向量
      const up = [1, 1, 0, -1, -1, -1, 0, 1];
      // 右方向向量
      const right = [0, 1, 1, 1, 0, -1, -1, -1];
      // 遍历当前格子周围一圈，如果索引越界就跳过
      for (let i = 0; i < up.length; i++) {
        let testRow = latticeRow + up[i];
        let testCol = latticeCol + right[i];
        // 行范围：[1, this.cols]  列范围：[1, this.rows]
        if (
          testRow < 1 ||
          testRow > this.cols ||
          testCol < 1 ||
          testCol > this.rows
        ) {
          continue;
        }
        // 所有格子索引范围（一维数组）：[0, this.cols * this.rows - 1]
        latticeIndexArr.push((testRow - 1) * this.rows + testCol - 1);
      }
      return latticeIndexArr;
    },
    // 格子左键点击事件
    handleClickLattice(lattice) {
      // 如果置了标记状态，说明是手机端点雷的操作
      if (this.isMarkStatus) {
        this.handleSureMinePoint(lattice);
      } else {
        this.leftClick(lattice);
      }
    },
    // 左键点击
    leftClick(lattice) {
      if (this.over) {
        return false;
      }
      // 是雷，提前结束游戏
      if (!lattice.isOpen && lattice.isMine) {
        lattice.isOpen = true;
        this.over = 1;
      } else if (lattice.mineNum) {
        // 是数字
        if (!lattice.isOpen && !lattice.isMark) {
          lattice.isOpen = true;
        }
      } else {
        // 是空白（数字为0）
        // 获取周围的格子然后展示周围的空白标记
        const latticeIndexArr = this.getLatticeIndex(lattice.index);
        this.showWhiteAround(lattice, latticeIndexArr);
      }
    },
    // 右键确认是雷点
    handleSureMinePoint(lattice) {
      // 格子未被点开
      if (!lattice.isOpen) {
        lattice.isMark = true;
        lattice.isOpen = true;
        // 在雷的数组中找到当前格子并删除他
        // indexOf：返回第一次出现的位置，找不到就返回-1
        // splice：从找到的位置开始删除[1]个元素，在这里标记错的话位置是-1，将会删除数组最后一个元素
        this.minePosition.splice(this.minePosition.indexOf(lattice.index), 1);
        this.judgeIsOver();
      } else {
        // 格子被点开了表示之前被标记过，那么再右键一次就是取消标记
        if (lattice.isMark) {
          lattice.isMark = false;
          lattice.isOpen = false;
          // 把地雷放回数组中（不用检查是不是真的地雷，真地雷数组已经备份了）
          this.minePosition.push(lattice.index);
        }
      }
    },
    // 判断游戏是否结束
    judgeIsOver() {
      this.over = this.minePosition.length === 0 ? 2 : 0;
    },
    // 判断错误的标记
    judgeWrongMark() {
      let wrongMark = 0;
      this.minePositionBake.forEach((item) => {
        if (!this.lattice[item].isMark) {
          wrongMark++;
        }
      });
      return wrongMark;
    },
    // 展示周围的空白标记，直至边缘（格子边缘或者数字）
    showWhiteAround(lattice, latticeIndexArr) {
      // 避免有重复的数据停不下来，去个重
      latticeIndexArr = [...new Set(latticeIndexArr)];
      for (let i = 0; i < latticeIndexArr.length; i++) {
        const item = latticeIndexArr[i];
        // 删除已经检测过的格子，降低后面递归里面的循环次数
        latticeIndexArr.splice(i--, 1);
        // 跳过已经点开的格子
        if (this.lattice[item].isOpen) {
          continue;
        }
        this.lattice[item].isOpen = true;
        // 如果当前格子数字为0就递归展示空白标记
        if (!this.lattice[item].mineNum) {
          const arr = this.getLatticeIndex(this.lattice[item].index);
          this.showWhiteAround(this.lattice[item], latticeIndexArr.concat(arr));
        }
      }
    },
    // 重开一局
    reStart() {
      this.init();
      this.setTime();
    },
    // 改变难度
    changeLevel() {
      this.$emit("update:showGame", false);
    },
  },
};
</script>
<style lang="scss" scoped>
@media (max-width: 767px) {
  #app .main {
    height: 100%;

    .tool-content-t {
      width: 100%;
    }

    .tool-content {
      position: static;
      margin: 20px auto;
      width: 50%;
      text-align: center;

      .tool {
        position: static;
        top: 0;
        transform: none;
      }
    }

    .game-content-row {
      width: max-content;
    }

    .game-content {
      width: 100%;
      max-height: 80%;
      overflow: scroll;
    }
  }
}

.main {
  padding: 10px;
  position: relative;
  top: 50%;
  transform: translateY(-50%);
  border: 2px dashed #0b7777;
  border-radius: 5px;

  .game-content-row {
    margin: 0 auto;
    width: fit-content;
    font-size: 0;

    .game-block {
      display: inline-block;
      margin: 2px;
      background-color: #bbb;
      width: 30px;
      height: 30px;
      line-height: 30px;
      font-size: 14px;
      color: #fff;
      text-align: center;
      cursor: pointer;

      &>span {
        visibility: hidden;
      }

      &.open {
        background-color: #ddd;

        &>span {
          visibility: visible;
        }
      }

      &.mark {
        color: red;
      }
    }
  }

  .tool-content-t {
    margin: 0 auto;
    width: 300px;

    .mark-btn {
      display: inline-block;
      margin: 10px 0;
      padding: 8px 12px;
      background-color: #0b7777;
      border-radius: 2px;
      width: 100px;
      text-align: center;
      color: #fff;
      cursor: pointer;

      &.marked {
        background-color: #a2891b;
      }

      &.right {
        float: right;
      }
    }
  }

  .tool-content {
    position: absolute;
    top: 20px;
    right: 10px;
    height: 100%;

    .tool {
      position: absolute;
      top: 50%;
      transform: translateY(-50%);

      &>div {
        margin: 10px 0;
        padding: 8px 12px;
        background-color: #0b7777;
        border-radius: 2px;
        color: #fff;
        cursor: pointer;
      }
    }
  }
}
</style>
