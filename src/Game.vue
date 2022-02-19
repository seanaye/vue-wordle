<script setup lang="ts">
import { onUnmounted } from 'vue'
import { getWordOfTheDay, allWords,answers } from './words'
import Keyboard from './Keyboard.vue'
import { LetterState } from './types'

// Get word of the day 
const answer = getWordOfTheDay()

//select a random word
function randomWord(answer: string, allowedWords: string[]){
    let guess: string = allowedWords[Math.floor(Math.random() * allowedWords.length)]; 
    return guess;
}

//generate sequence of 5 (yellow/green/grey) based on how the random word and answer match
function info(guess: string, answer: string){
  let sequence: string[] = ['grey', 'grey', 'grey', 'grey', 'grey'];
  let trackLetters: string[] = ['grey', 'grey', 'grey', 'grey', 'grey'];

  const guessLetters: (string | null)[] = guess.split('');
  const answerLetters: (string | null)[] = answer.split('');

  //for each i from 1 to 5
  guessLetters.forEach(function (item, index){
    //if the ith letter (item) of guess == ith letter of answer
    if (item == answerLetters[index] && trackLetters[index] == 'grey'){
      //mark the ith letter as used
      trackLetters[index] = 'green';
      sequence[index] = 'green';
    }
  })

  //for each j from 1 to 5
  guessLetters.forEach(function(item, index){
    //if the jth letter of guess is not equal to the jth letter of answer
    //but there exists some letter in answer that is equal but not used ('grey')
    if (item != answerLetters[index] && answerLetters.includes(item)){
      answerLetters.forEach(function(thing, e){
        if(answerLetters[e] == item && trackLetters[e] == 'grey'){
          trackLetters[e] = 'yellow';
          sequence[index] = 'yellow';
        } 
      })
    } 
  })
  return sequence;
}

function removeItemOnce(array: (string | null)[], value:(string | null)) {
  var index = array.indexOf(value);
  if (index > -1) {
    array.splice(index, 1);
  }
  return array;
}

//Get list of all words X in current allowed word list such that INFO(currentGuess, X) = SEQ1
function createArray(currentGuess: string, allowedWords: string[], sequence: string[]){  
  curList = [];  
  allowedWords.forEach(function(item, index){
      if(info(currentGuess, allowedWords[index]).join('') === sequence.join('')){
        curList = [...curList, allowedWords[index]];
      }
  })
  console.log(curList, 'NL');
  return curList;
}
//solve da game
function solveGame(answer: string, allowedWords: string[], guess: string){
    //append all sequences so this = [[seq1],[seq2], ... , [seqN]]
    let totalSequence: string[] = [];
    let iterations: number = 0;
    
    while(!simulationComplete){
      iterations++;
      console.log(answer, 'ans', guess, 'guess');
      const currentSequence = info(guess, answer);
      totalSequence.push(...currentSequence);

      //if the sequence is all green, terminate the game
      if(currentSequence.length === 5 && currentSequence.every(e => e === "green")) break;

      createArray(guess, allowedWords, currentSequence);
      //update the allowedWords array to include only new words
      allowedWords = curList;

      //update the guess with a word from new word array
      //only allow a max of 6 iteration
      if (iterations >= 5){
        guess = answer;
      } else {
        guess = randomWord(answer, curList);
      }
    }
    return totalSequence;
}

//check the word and compare to sequence
function sequenceCheck(guess:string, allowedWords:string[], answer:string){
    //generate sequence of guess 
    const curLongSequence = info(guess, answer);
    console.log(curLongSequence, 'CLS');
    //list of words Y in long
    createArray(guess,allowedWords,curLongSequence);
}

//check that the word matches the row prompt
function checkTileSequence(guess:string, answer:string){
  let checkSequenceList: string[] = [];
  currentRow.forEach(function(item, index){
    if(currentRow[index].state == "present"){
      checkSequenceList = [...checkSequenceList, 'yellow'];
    } else if (currentRow[index].state == "correct"){
      checkSequenceList = [...checkSequenceList, 'green'];
    } else {
      checkSequenceList = [...checkSequenceList, 'grey'];
    }
  });

  if (info(guess, answer).join('') ===  checkSequenceList.join('')){
    showMessage('Well done!');
    return true;
  } else {
    shake()
    showMessage('Does not match row sequence')
    return false;
  }
}


let simulationComplete: boolean = false;
let curList: string[] = [];
//initial guess & ensure it's not the answer right away
let guess: string = '';
do {
  guess = randomWord(answer, answers);
} while (guess == answer);

//this is the sequence of grey, yellow, green
let prompt: string[] = solveGame(answer, answers, guess);

// Board state. Each tile is represented as { letter, state }
const board = $ref(
  Array.from({ length: (prompt.length/5) }, () =>
    Array.from({ length: 5 }, () => ({
      letter: '',
      state: LetterState.INITIAL
    }))
  )
)

//go through board and set the letters/tiles as the prompt
function setBoard (prompt:string[]){
  //each row
  for (let i:number = 0; i<prompt.length/5; i++){
    //each column
    for (let j:number = 0; j<5; j++){
      if (prompt[(i*5 + j)] == 'grey'){
        board[i][j].state = LetterState.ABSENT;
      } else if (prompt[(i*5+j)] == 'yellow'){
        board[i][j].state = LetterState.PRESENT;
      } else {
        board[i][j].state = LetterState.CORRECT;
      }
      //show the final row
      if(i == (prompt.length/5 -1)){
        board[i][j].letter = answer.split('')[j];
      }
    }
  }
}
setBoard(prompt);

// Current active row.
let currentRowIndex = $ref(0)
const currentRow = $computed(() => board[currentRowIndex])
// Feedback state: message and shake
let message = $ref('')
let grid = $ref('')
let shakeRowIndex = $ref(-1)
let success = $ref(false)

// Keep track of revealed letters for the virtual keyboard
const letterStates: Record<string, LetterState> = $ref({})

// Handle keyboard input.
let allowInput = true

const onKeyup = (e: KeyboardEvent) => onKey(e.key)

window.addEventListener('keyup', onKeyup)

onUnmounted(() => {
  window.removeEventListener('keyup', onKeyup)
})

function onKey(key: string) {
  if (!allowInput) return
  if (/^[a-zA-Z]$/.test(key)) {
    fillTile(key.toLowerCase())
  } else if (key === 'Backspace') {
    clearTile()
  } else if (key === 'Enter') {
    completeRow()
  }
}

function fillTile(letter: string) {
  for (const tile of currentRow) {
    if (!tile.letter) {
      tile.letter = letter
      break
    }
  }
}

function clearTile() {
  for (const tile of [...currentRow].reverse()) {
    if (tile.letter) {
      tile.letter = ''
      break
    }
  }
}

function clearBoard() {
  while(currentRowIndex >= 0){
    for (const tile of [...currentRow].reverse()) {
      if (tile.letter) {
        tile.letter = ''
      }
    }
    currentRowIndex--;
  }
  currentRowIndex++;
}


//every time enter is hit, run this
let updatedLongList: string[] = [];
function completeRow() {
  if (currentRow.every((tile) => tile.letter)) {
    const guess = currentRow.map((tile) => tile.letter).join('')
    if (!allWords.includes(guess) && guess !== answer) {
      shake()
      showMessage(`Not in word list`)
      return
    } 
    //algorithm 3.a
    if (!checkTileSequence(guess, answer)) return;

    //get Long1 on the first guess
    if (currentRowIndex == 0){
      createArray(guess, allWords, info(guess, answer));
      updatedLongList = curList;
    } else {
      //algorithm part 3.b.ii
      let str = guess;
      if(updatedLongList.some(v => str.includes(v)) && !currentRow.every((tile, index) => tile.letter === answer[index])){
        sequenceCheck(guess, updatedLongList, answer);
        updatedLongList = curList;
      } else {
        shake()
        showMessage('Make sure your answer is consistent with other rows!')
        return
      }    
    }

    const answerLetters: (string | null)[] = answer.split('')

    // first pass: mark correct ones
    currentRow.forEach((tile, i) => {
      if (answerLetters[i] === tile.letter) {
        tile.state = letterStates[tile.letter] = LetterState.CORRECT
        answerLetters[i] = null
      }
    })
    // second pass: mark the present
    currentRow.forEach((tile) => {
      if (tile.state !== LetterState.CORRECT && tile.state !== LetterState.ABSENT && answerLetters.includes(tile.letter)) {
        tile.state = LetterState.PRESENT
        answerLetters[answerLetters.indexOf(tile.letter)] = null
        if (!letterStates[tile.letter]) {
          letterStates[tile.letter] = LetterState.PRESENT
        }
      }
    })
    // 3rd pass: mark absent
    currentRow.forEach((tile) => {
      if (tile.state !== LetterState.PRESENT && tile.state !== LetterState.CORRECT) {
        tile.state = LetterState.ABSENT
        if (!letterStates[tile.letter]) {
          letterStates[tile.letter] = LetterState.ABSENT
        }
      }
    })

    allowInput = false
    if (currentRowIndex == board.length - 2) {
      // yay!
      setTimeout(() => {
        grid = genResultGrid()
        showMessage('Splendid');
        success = true
      }, 1600)
    } else if (currentRowIndex < board.length - 2) {
      currentRowIndex++;
      setTimeout(() => {
        allowInput = true
      }, 1600)
    } else {
      // game over :(
      setTimeout(() => {
        showMessage('This game is hard!', -1)
      }, 1600)
    }
  } else {
    shake()
    showMessage('Not enough letters')
  }
}

function showMessage(msg: string, time = 1600) {
  message = msg
  if (time > 0) {
    setTimeout(() => {
      message = ''
    }, time)
  }
}

function shake() {
  shakeRowIndex = currentRowIndex
  setTimeout(() => {
    shakeRowIndex = -1
  }, 1000)
}

const icons = {
  [LetterState.CORRECT]: '🟩',
  [LetterState.PRESENT]: '🟨',
  [LetterState.ABSENT]: '⬜',
  [LetterState.INITIAL]: null
}

function genResultGrid() {
  return board
    .slice(0, currentRowIndex + 2)
    .map((row) => {
      return row.map((tile) => icons[tile.state]).join('')
    })
    .join('\n')
}
</script>

<template>
  <Transition>
    <div class="message" v-if="message">
      {{ message }}
      <pre v-if="grid">{{ grid }}</pre>
    </div>
  </Transition>
  <header>
    <h1>INVERDLE</h1>
    <!-- Link to source code
    <a
      id="source-link"
      href="https://github.com/yyx990803/vue-wordle"
      target="_blank"
      >Source</a
    >-->
  </header>
  <div id="board">
    <div
      v-for="(row, index) in board"
      :class="[
        'row',
        shakeRowIndex === index && 'shake',
        success && currentRowIndex === index && 'jump'
      ]"
    >
      <div
        v-for="(tile, index) in row"
        :class="['tile', tile.letter && 'filled', tile.state && 'revealed']"
      >
        <div class="front" :style="{ transitionDelay: `${index * 300}ms` }">
          {{ tile.letter }}
        </div>
        <div
          :class="['back', tile.state]"
          :style="{
            transitionDelay: `${index * 300}ms`,
            animationDelay: `${index * 100}ms`
          }"
        >
          {{ tile.letter }}
        </div>
      </div>
    </div>
  </div>
  <div class="clear">
     <button 
      id="clearBtn"
      @click="clearBoard">
      Clear Board
    </button>
  </div>
  <Keyboard @key="onKey" :letter-states="letterStates" />
</template>

<style scoped>
#board {
  display: grid;
  grid-template-rows: repeat(6, 1fr);
  grid-gap: 5px;
  padding: 10px;
  box-sizing: border-box;
  --height: min(420px, calc(var(--vh, 100vh) - 310px));
  height: var(--height);
  width: min(350px, calc(var(--height) / 6 * 5));
  margin: 0px auto;
}
#clearBtn {
  padding: 12px;
  margin: 0px 8px 8px 8px;
  font-family: inherit;
  font-weight: bold;
  border: 0;
  height: auto;
  background-color: #1a1a1b;
  color: white;
  border-radius: 4px;
  cursor: pointer;
  user-select: none;
  flex: 1;
  text-transform: uppercase;
  -webkit-tap-highlight-color: rgba(0, 0, 0, 0.3);
  transition: all 0.2s 1.5s;
}
.clear {
  display: flex;
  justify-content: center;
  align-items: center;
}

.message {
  position: absolute;
  left: 50%;
  top: 80px;
  color: #fff;
  background-color: rgba(0, 0, 0, 0.85);
  padding: 16px 20px;
  z-index: 2;
  border-radius: 4px;
  transform: translateX(-50%);
  transition: opacity 0.3s ease-out;
  font-weight: 600;
}
.message.v-leave-to {
  opacity: 0;
}
.row {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  grid-gap: 5px;
}
.tile {
  width: 100%;
  font-size: 2rem;
  line-height: 2rem;
  font-weight: bold;
  vertical-align: middle;
  text-transform: uppercase;
  user-select: none;
  position: relative;
}
.tile.filled {
  animation: zoom 0.2s;
}
.tile .front,
.tile .back {
  box-sizing: border-box;
  display: inline-flex;
  justify-content: center;
  align-items: center;
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  transition: transform 0.6s;
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
}
.tile .front {
  border: 2px solid #d3d6da;
}
.tile.filled .front {
  border-color: #999;
}
.tile .back {
  transform: rotateX(180deg);
}
.tile.revealed .front {
  transform: rotateX(180deg);
}
.tile.revealed .back {
  transform: rotateX(0deg);
}

@keyframes zoom {
  0% {
    transform: scale(1.1);
  }
  100% {
    transform: scale(1);
  }
}

.shake {
  animation: shake 0.5s;
}

@keyframes shake {
  0% {
    transform: translate(1px);
  }
  10% {
    transform: translate(-2px);
  }
  20% {
    transform: translate(2px);
  }
  30% {
    transform: translate(-2px);
  }
  40% {
    transform: translate(2px);
  }
  50% {
    transform: translate(-2px);
  }
  60% {
    transform: translate(2px);
  }
  70% {
    transform: translate(-2px);
  }
  80% {
    transform: translate(2px);
  }
  90% {
    transform: translate(-2px);
  }
  100% {
    transform: translate(1px);
  }
}

.jump .tile .back {
  animation: jump 0.5s;
}

@keyframes jump {
  0% {
    transform: translateY(0px);
  }
  20% {
    transform: translateY(5px);
  }
  60% {
    transform: translateY(-25px);
  }
  90% {
    transform: translateY(3px);
  }
  100% {
    transform: translateY(0px);
  }
}

@media (max-height: 680px) {
  .tile {
    font-size: 3vh;
  }
}
</style>
