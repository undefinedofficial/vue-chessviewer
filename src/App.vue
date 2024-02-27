<template>
  <ElContainer>
    <ElMain>
      <div class="chessboard">
        <Chessboard :fen="fen" :orientation="orientation">
          <ChessboardMarkers :markers="markers" />
          <ChessboardControl
            :enableColor="turn"
            @beforeMove="onBeforeMove"
            @afterMove="onAfterMove"
            @cancelMove="onCancelMove"
          />
        </Chessboard>
      </div>
    </ElMain>
    <ElAside width="25%">
      <ElSpace direction="vertical" fill wrap>
        <ElCard>
          <template #header>
            <ElText size="large">PGN</ElText>
          </template>
          <ElSpace direction="vertical" fill>
            <ElInput type="textarea" v-model="pgn" size="large" style="width: 100%" rows="14" />

            <ElSpace>
              <ElButton type="default" bg @click="clear"> Очистить </ElButton>
              <ElButton type="primary" @click="loadPgn"> Загрузить </ElButton>
            </ElSpace>
          </ElSpace>
        </ElCard>
        <ElCard>
          <template #header>
            <ElText size="large">Список ходов</ElText>
          </template>
          <ElSpace wrap>
            <ElText
              v-for="(move, i) in history.concat(nextMoves)"
              :key="move + i"
              :type="i === history.length - 1 ? 'success' : 'info'"
            >
              {{ move }}
            </ElText>
          </ElSpace>
          <template #footer>
            <ElButton type="info" @click="onUndo" :disabled="history.length === 0">
              Отменить
            </ElButton>
            <ElButton type="primary" @click="onRedo" :disabled="nextMoves.length === 0">
              Вернуть
            </ElButton>
          </template>
        </ElCard>
      </ElSpace>
    </ElAside>
  </ElContainer>
</template>

<script setup lang="ts">
import {
  ElAside,
  ElButton,
  ElCard,
  ElContainer,
  ElInput,
  ElMain,
  ElMessageBox,
  ElSpace,
  ElText,
} from "element-plus";

import {
  Chessboard,
  ChessboardControl,
  ChessboardMarkers,
  MARKER,
  type Marker,
  type MarkerPoint,
} from "cw-chessboard";
import { ref } from "vue";
import { Chess, type Color } from "chess.ts";

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

const pgn = ref(`[Event "?"]
[Site "?"]
[Date "????.??.??"]
[Round "?"]
[White "01"]
[Black "Вилка конём"]
[Result "1-0"]
[SetUp "1"]
[FEN "r1bk3Q/pp1p4/1bp2qp1/4N3/3P1p2/6P1/PPP4P/R1B2RK1 b - - 0 1"]
[PlyCount "4"]

1... Qxh8 2. Nf7+ Ke8 3. Nxh8 1-0
`);

const games = ref<string[]>([]);
const gameindex = ref<number>(0);

let chess = new Chess();
const turn = ref<Color>("w");
const fen = ref(chess.fen());
const orientation = ref<"w" | "b">("w");
const history = ref(chess.history());
const nextMoves = ref<string[]>([]);

// синхронизируем состояние chess.ts с доской
const syncChess = () => {
  fen.value = chess.fen();
  turn.value = chess.turn();
  history.value = chess.history();
};

const onUndo = () => {
  if (history.value.length === 0) return;

  const move = chess.undo()!;
  nextMoves.value.unshift(move.san);

  syncChess();
};

const onRedo = () => {
  if (nextMoves.value.length === 0) return;

  chess.move(nextMoves.value.shift()!);
  syncChess();
};

const markers = ref<(Marker & { id?: string })[]>([]);

const loadPgn = async () => {
  games.value = pgn.value.split("\r\n\r\n[").map((x, i) => (i === 0 ? x : "[" + x));
  gameindex.value = 0;

  markers.value = [];
  chess.loadPgn(games.value[gameindex.value]);
  orientation.value = chess.turn();
  syncChess();

  await sleep(1000);
};
loadPgn();

const clear = () => {
  pgn.value = "";
  chess = new Chess();
  syncChess();
};

const selectMarkers = (
  id: string,
  marker?: MarkerPoint["type"],
  squares?: string[],
  color?: MarkerPoint["color"]
): void => {
  markers.value = markers.value.filter((m) => m.id !== id);
  if (squares)
    squares.forEach((square) => markers.value.push({ id, type: marker!, square, color }));
};

// вызывается перед перемещением фигуры или фокусом на фигуру
const onBeforeMove = (square: string, done: (accept: boolean) => void) => {
  console.log("BeforeMove: ", square);

  // получаем возможные ходы
  const moves = chess.moves({ square, verbose: true });

  // если нет ходов - отменяем перемещение
  if (moves.length === 0) return done(false);

  // отмечаем возможные ходы
  selectMarkers(
    "selected",
    MARKER.DOT,
    moves.map((m) => m.to)
  );

  // разрешаем перемещение
  done(true);
};

// вызывается после перемещения фигуры
const onAfterMove = async (
  fromSquare: string,
  toSquare: string,
  done: (accept: boolean) => void
) => {
  console.log("AfterMove: ", fromSquare, toSquare);

  // сбрасываем маркеры
  selectMarkers("selected");
  selectMarkers("active");

  // делаем ход
  const move = chess.move({ from: fromSquare, to: toSquare });

  // если ход невозможный - отменяем
  if (!move) return done(false);

  // разрешаем ход
  done(true);

  // синхронизируем состояние chess.ts с доской
  syncChess();
};
const onCancelMove = (square: string) => {
  console.log("CancelMove: ", square);
  selectMarkers("selected");
};
</script>

<style scoped>
.chessboard {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 100%;
}
</style>
