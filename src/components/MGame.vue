/** màn hình trò chơi created by tdmanh1 06/05/2023 */
<template>
  <div class="m-game" @click="pauseGame">
    <!-- màn hình tạm dừng -->
    <div class="m-pause" :class="isPaused ? 'm-pause-active' : null">
      <div v-if="isGameOver">Game over</div>
      <div v-else>Paused</div>
    </div>
    <!-- màn hình game -->
    <div class="m-fence" :style="gameSizeStyle">
      <!-- tạo ra bảng dựa vào kích thước của gameSize -->
      <template v-for="tempRow in gameGrid" :key="tempRow">
        <template v-for="singleSquare in tempRow" :key="singleSquare">
          <div
            v-if="singleSquare.eatingSquare"
            class="m-cube"
            :class="singleSquare.eatingSquare ? 'm-cube-eating' : null"
          ></div>
          <div
            v-else
            class="m-cube"
            :class="singleSquare.activeSquare ? 'm-cube-active' : null"
          ></div>
        </template>
      </template>
    </div>
  </div>
</template>
<script>
import gameConfig from '../config/gameConfig.js'
import { isProxy } from 'vue'
export default {
  data() {
    return {
      gameGrid: [], // mảng lưu toàn bộ các ô có trong trò chơi
      activeSquares: [], // mảng lưu danh sách các ô vuông sẽ active
      eatingSquare: { x: 0, y: 0 }, // object lưu ô màu đỏ để rắn săn mồi ăn
      movingPosition: gameConfig.position.right, // bien luu huong di chuyen hien tai
      intervalRendering: null, // render lien tuc tu khi mouted
      isPaused: false, // trạng thái của game là tạm dừng hay không
      isGameOver: false // trang thai cua game la ket thuc hay khong
    }
  },
  props: {
    // kích cỡ của trò chơi, mặc định là ô 7x7
    gameSize: {
      type: Number,
      default: gameConfig.gameSize.defaultSize
    },
    //speed của game
    gameSpeed: {
      type: Number,
      default: gameConfig.timeRerender.defaultSpeed
    },
    // hướng di chuyển hiện tại của con rắn
    currentPosition: {
      type: Number,
      default: gameConfig.position.right
    }
  },
  computed: {
    /**
     * tạo ra grid động
     * @author tdmanh1 10-05-2023
     */
    gameSizeStyle() {
      return {
        'grid-template-columns': `repeat(${this.gameSize}, 1fr)`
      }
    }
  },
  /**
   * them su kien khi mouted
   * @author tdmanh1 10-05-2023
   */
  mounted() {
    let me = this
    me.generateGameGrid(true)
    me.changeIntervalGame(true)
    // thêm lắng nghe sự kiện ấn phím
    window.addEventListener('keydown', me.keyDownPauseGame)
  },

  /**
   * clear cac su kien duoc mount vao dom
   * @author tdmanh1 10-05-2023
   */
  beforeUnmount() {
    let me = this
    me.changeIntervalGame()
    // tdmanh1 13/05/2023 xóa sự kiện lắng nghe ấn phím
    window.removeEventListener('keydown', me.keyDownPauseGame)
  },
  watch: {
    /**
     * tự động load lại danh sách mỗi lần thay đổi kích thước
     * created by tdmanh1 09/05/2023
     */
    gameSize() {
      let me = this
      // re-render lại grid mỗi lần bấm vào thay đổi kích thước
      me.generateGameGrid(true)
    },
    /**
     * lắng nghe khi vị trí thay đổi thì tiến hành thay đổi hướng di chuyển của con rắn
     * ngan chan viec di chuyen vi pham vd nhu tu tren chuyen huong quay xuong duoi hoac tu trai chuyen huong quay sang phai
     * @author tdmanh1 10-05-2023
     */
    currentPosition(newValue, oldValue) {
      let me = this
      if (
        newValue &&
        oldValue &&
        ((newValue == gameConfig.position.up && oldValue != gameConfig.position.down) ||
          (newValue == gameConfig.position.down && oldValue != gameConfig.position.up) ||
          (newValue == gameConfig.position.left && oldValue != gameConfig.position.right) ||
          (newValue == gameConfig.position.right && oldValue != gameConfig.position.left))
      ) {
        me.movingPosition = newValue
      } else {
        // case xem lan di chuyen dau neu khong co thi phai set the nay
        if (oldValue) {
          me.movingPosition = oldValue
        } else {
          me.movingPosition = newValue
        }
      }
    },
    /**
     * thay đổi tốc độ game
     */
    gameSpeed() {
      let me = this
      me.changeIntervalGame(true)
    }
  },
  methods: {
    /**
     * lắng nghe sự kiện ấn phím
     * @author tdmanh1 10-05-2023
     */
    keyDownPauseGame(e) {
      let me = this
      if (e && e.keyCode) {
        switch (e.keyCode) {
          case gameConfig.keyCode.space:
          case gameConfig.keyCode.enter:
            me.pauseGame()
            break
          default:
            break
        }
      }
    },
    /**
     * hàm chung xóa và thêm interval
     * @author tdmanh1 10-05-2023
     */
    changeIntervalGame(reset) {
      let me = this
      clearInterval(me.intervalRendering)
      if (reset) {
        me.intervalRendering = setInterval(() => {
          me.caculateNewPosition()
        }, me.gameSpeed)
      }
    },
    /**
     * thực hiện tạm dừng game
     * @author tdmanh1 10-05-2023
     */
    pauseGame() {
      let me = this
      //neu dang la game over thi bo no di
      if (me.isGameOver) {
        me.isGameOver = false
      }
      // thay đổi trạng thái của game
      me.isPaused = !me.isPaused
      // thực hiện tạm dừng hoặc tiếp tục game
      if (me.isPaused) {
        me.changeIntervalGame()
      } else {
        me.changeIntervalGame(true)
      }
    },
    /**
     * vue3 tự động biến array thành 1 object proxy array, dùng hàm này để convert lại về array
     * @author tdmanh1 09-05-2023
     * @param proxyArray array muốn convert
     */
    getArrayFromProxyArr(proxyArray) {
      if (proxyArray && isProxy(proxyArray)) {
        // không dùng toRaw vì toRaw vẫn chỉ là tham chiếu tới proxy array của vue3, gây ra thay đổi dữ liệu của
        // 1 biến copy từ item trong array gây ra thay đổi dữ liệu của chính array đó
        return JSON.parse(JSON.stringify(proxyArray))
      } else {
        return proxyArray
      }
    },
    /**
     * tạo ra bảng game dựa vào dữ liệu truyền vào
     * @author tdmanh1 09-05-2023
     * @param isClear : có muốn xóa toàn bộ dữ liệu đi không
     */
    generateGameGrid(isClear) {
      let me = this
      if (me.gameSize) {
        // nếu là clear đi thì reset toàn bộ các biến trong chương trình
        if (isClear) {
          me.gameGrid = []
          me.activeSquares = []
          // tạo ra 1 ô active ngẫu nhiên
          me.activeSquares.push(me.randomSquare())
          me.generateGameEating()
        }
        me.renderGameGrid()
      }
    },
    /**
     * tao ra o de an moi
     * @author tdmanh1 09-05-2023
     */
    generateGameEating() {
      let me = this
      // tạo ra ô rắn săn mồi ngẫu nhiên
      me.eatingSquare = me.randomSquare()
      if (me.activeSquares && me.gameSize && me.activeSquares.length != me.gameSize * me.gameSize) {
        while (me.checkIncludeSquare(me.activeSquares, me.eatingSquare)) {
          me.eatingSquare = me.randomSquare()
        }
      }
    },
    /**
     * tạo ra ô random muốn active hoặc tô màu
     * @author tdmanh1 09-05-2023
     */
    randomSquare() {
      let me = this
      // mặc định sẽ là ô trên cùng bên trái
      let result = [0, 0]
      if (me.gameSize) {
        let xRandom = Math.floor(Math.random() * me.gameSize)
        let yRandom = Math.floor(Math.random() * me.gameSize)
        result = { x: xRandom, y: yRandom }
      }
      return result
    },
    /**
     * ô cần check có đúng tọa độ không
     * @author tdmanh1 09-05-2023
     * @param coordinate : tọa độ cần kiểm tra
     */
    checkIncludeSquare(testSquares, coordinate) {
      let me = this
      // kiểm tra xem ô vuông hiện tại có cần active không
      let result = false
      let activeSquares = me.getArrayFromProxyArr(testSquares)
      if (
        coordinate &&
        coordinate.x != null &&
        coordinate.x != undefined &&
        coordinate.y != null &&
        coordinate.y != undefined &&
        activeSquares &&
        activeSquares.length > 0 &&
        activeSquares.filter((element) => element.x == coordinate.x && element.y == coordinate.y)
          .length > 0
      ) {
        result = true
      }
      return result
    },
    /**
     * thực hiện render ra màn hình game
     * @author tdmanh1 09-05-2023
     * @param activeSquares : mảng các ô vuông mong muốn sẽ hiển thị
     */
    renderGameGrid() {
      let me = this
      // reset gameGrid
      me.gameGrid = []
      for (let i = 0; i < me.gameSize; i++) {
        // tao ra bang tam luu 1 dong cua game
        let tempRow = []
        for (let k = 0; k < me.gameSize; k++) {
          // kiểm tra xem ô này có cần active không
          let tempActiveSquare = me.checkIncludeSquare(me.activeSquares, { x: i, y: k })
          // kiểm tra xem ô hiện tại có phải là ô màu đỏ ( rắn săn mồi sẽ ăn ô này)
          let tempEatingSquare =
            me.eatingSquare && me.eatingSquare.x == i && me.eatingSquare.y == k ? true : false
          // tạo ra 1 ô vuông
          let singleSquare = {
            activeSquare: tempActiveSquare,
            eatingSquare: tempEatingSquare
          }
          tempRow.push(singleSquare)
        }
        // them 1 dong vao mang de render
        me.gameGrid.push(tempRow)
      }
    },
    /**
     * thay đổi vị trí các ô hiển thị trên màn hình
     * @author tdmanh1 09-05-2023
     * @param newPosition vị trí mới mong muốn hiển thị
     */
    caculateNewPosition() {
      let me = this
      let oldActiveSquares = me.getArrayFromProxyArr(me.activeSquares)
      let temp = me.getArrayFromProxyArr(me.activeSquares)
      if (me.movingPosition && oldActiveSquares && oldActiveSquares.length > 0 && me.gameSize) {
        // tính toán lại vị trí từng thành phần trong mảng
        // lấy ra giá trị cuối cùng
        // json parse vaf json stringify de clone object thay vi reference
        let tempActiveSquare = JSON.parse(JSON.stringify(temp[temp.length - 1]))
        // tinh toan vi tri moi
        let newActiveSquare = me.manipulateCoordinate(tempActiveSquare)
        // kiểm tra xem ô temp này có nằm trong mảng có sẵn không
        // nếu có thì endgame không thì render tiếp
        if (me.checkIncludeSquare(oldActiveSquares, newActiveSquare)) {
          me.generateGameGrid(true)
          // set trang thai game la isGameOver = true
          me.pauseGame()
          me.isGameOver = true
          // reset diem game
          me.$emit('changeScore', 0)
        } else {
          // kiểm tra xem ô tiếp theo là ô ăn hay không
          // nếu là ô ăn thì không xóa ô cuối cùng đi
          if (
            newActiveSquare &&
            me.eatingSquare &&
            newActiveSquare.x == me.eatingSquare.x &&
            newActiveSquare.y == me.eatingSquare.y
          ) {
            // tạo ra ô rắn săn mồi ngẫu nhiên mới
            me.generateGameEating()
            // tang diem cho game
            me.$emit('changeScore', 1)
          } else {
            // xóa ô đầu tiên đi nếu không ăn được thêm ô nào
            oldActiveSquares.shift()
          }

          // thêm giá trị vừa thay đổi vào ô cuối cùng của mảng
          oldActiveSquares.push(newActiveSquare)
          me.activeSquares = oldActiveSquares
          // render lại game theo active mới
          me.renderGameGrid()
        }
      }
    },
    /**
     * refactore code
     * @author tdmanh1 11-05-2023
     * @param newPosition vị trí mới mong muốn hiển thị
     */
    manipulateCoordinate(tempActiveSquare) {
      let me = this
      let newActiveSquare = { x: tempActiveSquare.x, y: tempActiveSquare.y }
      if (tempActiveSquare) {
        switch (me.movingPosition) {
          // di chuyển lên
          case gameConfig.position.up:
            if (
              tempActiveSquare.x != null &&
              tempActiveSquare.x != undefined &&
              tempActiveSquare.x > 0
            ) {
              newActiveSquare.x = tempActiveSquare.x - 1
            } else {
              newActiveSquare.x = me.gameSize - 1
            }
            break
          // di chuyển sang trái
          case gameConfig.position.left:
            if (
              tempActiveSquare.y != null &&
              tempActiveSquare.y != undefined &&
              tempActiveSquare.y > 0
            ) {
              newActiveSquare.y = tempActiveSquare.y - 1
            } else {
              newActiveSquare.y = me.gameSize - 1
            }
            break
          // di chuyển xuống dưới
          case gameConfig.position.down:
            if (
              tempActiveSquare.x != null &&
              tempActiveSquare.x != undefined &&
              tempActiveSquare.x < me.gameSize - 1
            ) {
              newActiveSquare.x = tempActiveSquare.x + 1
            } else {
              newActiveSquare.x = 0
            }
            break
          // di chuyển sang phải
          case gameConfig.position.right:
            if (
              tempActiveSquare.y != null &&
              tempActiveSquare.y != undefined &&
              tempActiveSquare.y < me.gameSize - 1
            ) {
              newActiveSquare.y = tempActiveSquare.y + 1
            } else {
              newActiveSquare.y = 0
            }
            break
          default:
            break
        }
      }
      return newActiveSquare
    }
  }
}
</script>
<style scoped>
.m-game {
  background-color: black;
  height: 100%;
  width: 100%;
  border-radius: var(--border-radius);
  padding: var(--padding-common);
  position: relative;
}
.m-fence {
  width: 100%;
  height: 100%;
  box-sizing: border-box;
  border: 1px solid rgba(255, 255, 255, 0.074);
  /* background-color: cadetblue; */
  display: grid;
  grid-template-columns: repeat(gameSize, 1fr);
}
.m-cube {
  display: flex;
  border: 1px solid rgba(255, 255, 255, 0.074);
}
.m-cube-active {
  background-color: white;
}
.m-cube-eating {
  background-color: rgb(255, 0, 0);
}
.m-pause {
  opacity: 0;
  color: white;
  position: absolute;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  border-radius: var(--border-radius);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 3rem;
}
.m-pause-active {
  opacity: 1;
  background-color: rgba(106, 106, 106, 0.334);
}
</style>
