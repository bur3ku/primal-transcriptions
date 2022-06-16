<script>
export let editor;
export let terms;
export let close;

const text = editor.getText();
const textArr = text.split(" ");

const traverseNWords = (initialCharIndex, n,log=false) => {
  let charIndex = 0;
  let wordIndex = 0;

  while(charIndex < initialCharIndex){
    charIndex += textArr[wordIndex].length + 1;
    wordIndex+=1;
  }
  
  const direction = n > 0 ? 1 : -1;
  let distance = 0;
  let curWordIndex = wordIndex;

  if(direction === 1){
    while(curWordIndex < textArr.length && curWordIndex-wordIndex < n){
      if(log)console.log("adding",text.substr(charIndex,textArr[curWordIndex].length + 1));
      charIndex += textArr[curWordIndex].length + 1;
      distance += textArr[curWordIndex].length + 1;
      curWordIndex+=direction;
    }
  }else{

    while(curWordIndex>0 && wordIndex-curWordIndex<Math.abs(n)){
      charIndex -= (textArr[curWordIndex-1].length + 1);
      distance += (textArr[curWordIndex-1].length + 1);
      curWordIndex+=direction;
    }
  }

  return {"index":charIndex,"distance":distance,"text":text.substr(direction===1?initialCharIndex:charIndex,distance)};
}

let curTermsKeys = [];
let curTermsKeysIndex = 0;


const prevTerm = () => {
  curTermsKeysIndex--;
  excerpts.setTermIndexes();
  excerpts.curIndex = 0;
}

const nextTerm = () => {
  curTermsKeysIndex++;
  excerpts.setTermIndexes();
  excerpts.curIndex = 0;
}

Object.keys(terms).forEach((key)=>{
  let matches = [];
  textArr.forEach((word, index)=>{
    if(word.toLowerCase() === key.toLowerCase()) matches.push(index);
  })
  terms[key].synonyms.forEach((synonym)=>{
    textArr.forEach((word, index)=>{
      if(word.toLowerCase() === synonym.toLowerCase()) matches.push(index);
    })
  })
  if(matches.length > 0) curTermsKeys.push(key);
  matches = matches;
  curTermsKeys = curTermsKeys;
})

document.addEventListener('keydown',(e)=>{
  if(e.key === "<") excerpts.navigate(-1,true);
  else if(e.key === ">") excerpts.navigate(1,true);
  else if(e.key==="a") excerpts.indexes[excerpts.curIndex].leftSide-=defaultSurroundingTermLength;
  else if(e.key==="A") excerpts.indexes[excerpts.curIndex].leftSide-=1;
  else if(e.key==="d") excerpts.indexes[excerpts.curIndex].leftSide=Math.min(0,excerpts.indexes[excerpts.curIndex].leftSide + defaultSurroundingTermLength);
  else if(e.key==="D") excerpts.indexes[excerpts.curIndex].leftSide=Math.min(0,excerpts.indexes[excerpts.curIndex].leftSide + 1);
  else if(e.key==="j") excerpts.indexes[excerpts.curIndex].rightSide=Math.max(0,excerpts.indexes[excerpts.curIndex].rightSide - defaultSurroundingTermLength);
  else if(e.key==="J") excerpts.indexes[excerpts.curIndex].rightSide=Math.max(0,excerpts.indexes[excerpts.curIndex].rightSide - 1);
  else if(e.key==="l") excerpts.indexes[excerpts.curIndex].rightSide+=defaultSurroundingTermLength;
  else if(e.key==="L") excerpts.indexes[excerpts.curIndex].rightSide+=1;
})

const surroundingPreviewLength = 10; //how many words before and after the excerpt we want to be shown. The excerpt itself has black letters while the surrounding preview has gray letters
const defaultSurroundingTermLength = 10; //also serves as the jump length (bound to a, d, j, l)

let excerpts = {
  curIndex: 0,
  disabledFromRight:false,
  disabledFromLeft:false,
  indexes: [],
  setTermIndexes: function() {
    if(curTermsKeys.length===0)return;
    const curTermName = curTermsKeys[curTermsKeysIndex];
    let matches = [...text.matchAll(new RegExp("(^|[^a-zA-Z])"+curTermName+"(?=([^a-zA-Z]|$))",'g'))].map(a => a[0].length > curTermName.length ? a.index+1:a.index);
    terms[curTermName].synonyms.forEach((synonym)=>{
      matches = matches.concat([...text.matchAll(new RegExp("(^|[^a-zA-Z])" + synonym + "(?=([^a-zA-Z]|$))",'g'))].map(a => a[0].length > curTermName.length ? a.index+1:a.index));
    })
    this.indexes = matches.sort(function(a, b) {
      return a - b;
    }).map((index)=>({index:index,leftSide:defaultSurroundingTermLength*-1,rightSide:defaultSurroundingTermLength,prevIndex:false,nextIndex:false}));
  },
  navigate: function(direction,move=false) {
    beginningIndex = traverseNWords(excerpts.indexes[excerpts.curIndex].index, excerpts.indexes[excerpts.curIndex].leftSide).index;
        endIndex = traverseNWords(excerpts.indexes[excerpts.curIndex].index, excerpts.indexes[excerpts.curIndex].rightSide+1).index;

    let i;
    for(i = this.curIndex; i < this.indexes.length && i >= 0; i += direction){
      if((this.indexes[i].index < beginningIndex || this.indexes[i].index >= endIndex)) {
        if((direction === 1 && this.indexes[i].disabledFromLeft) || (direction === -1 && this.indexes[i].disabledFromRight)) {
          if(direction === 1 && this.indexes[i].disabledFromLeft) {
            this.indexes[i].disabledFromLeft = false;
            excerpts = excerpts;
          }
          else if(direction === -1 && this.indexes[i].disabledFromRight) {
            this.indexes[i].disabledFromRight = false;
            excerpts = excerpts;
          }
        }else if((direction === 1 && this.indexes[i].disabledFromRight) || (direction === -1 && this.indexes[i].disabledFromLeft)){
        }else{
          if(move) {
            this.curIndex = i;
            excerpts = excerpts;
          }
          return i;
        }
      }else if(i !== this.curIndex){
        if(direction === 1 && !this.indexes[i].disabledFromLeft) {
          this.indexes[i].disabledFromLeft = true;
          excerpts = excerpts;
        }
        else if(direction === -1 && !this.indexes[i].disabledFromRight) {
          this.indexes[i].disabledFromRight = true;
          excerpts = excerpts;
        }
      }
    }
    return false;
  }
}
excerpts.setTermIndexes();

let beginningIndex;
let endIndex;
if(curTermsKeys.length>0){
  beginningIndex = traverseNWords(excerpts.indexes[excerpts.curIndex].index, excerpts.indexes[excerpts.curIndex].leftSide).index;
  endIndex = traverseNWords(excerpts.indexes[excerpts.curIndex].index, excerpts.indexes[excerpts.curIndex].rightSide+1).index;
}

</script>
<div class="absolute flex w-screen h-screen justify-center items-center">
  <div class="bg-[#DBD8D1] flex flex-col items-center max-w-[400px] border-black border-2 z-20">
    {#if curTermsKeys.length>0}
      <div class="flex justify-between w-[100%] items-center">
        <div>
          {#if curTermsKeysIndex > 0}
            <button class="navigateButton" on:click={prevTerm}>←</button>
          {/if}
        </div>
        <span style="font-size: 25px">{curTermsKeys[curTermsKeysIndex]}</span>
        <div>
          {#if curTermsKeysIndex<curTermsKeys.length-1}
            <button class="navigateButton" on:click={nextTerm}>→</button>
          {/if}
        </div>
      </div>
      <hr class="hr"/>
      <div class="flex mb-3 mt-3">
          {#if (excerpts.navigate(-1) !== false)}
            <button class="navigateButton" on:click={()=>excerpts.navigate(-1,true)}>←</button>
          {/if}
          {@html (()=>{
            beginningIndex = traverseNWords(excerpts.indexes[excerpts.curIndex].index, excerpts.indexes[excerpts.curIndex].leftSide).index;
            endIndex = traverseNWords(excerpts.indexes[excerpts.curIndex].index, excerpts.indexes[excerpts.curIndex].rightSide+1).index;

            let beforeExcerptWhite = "";
            let beforeExcerptBlack = "";
            let beforeExcerptRed = "";

            let afterExcerptWhite = "";
            let afterExcerptBlack = "";
            let afterExcerptRed = "";

            let whiteExcerpt = "";
            let whiteExcerptStartingIndex = 0;

            let excerpt = text.substr(beginningIndex,endIndex-beginningIndex);
            const leftPreview = traverseNWords(beginningIndex, surroundingPreviewLength*-1);

            if(excerpts.curIndex > 0){
              const leftExcerptIndex = excerpts.navigate(-1);
              const leftExcerptEndIndex = leftExcerptIndex === false ? 0 : traverseNWords(excerpts.indexes[leftExcerptIndex].index,excerpts.indexes[leftExcerptIndex].rightSide+1).index;
              const leftPreviewOverlap = Math.max(0,leftExcerptEndIndex-leftPreview.index);
              beforeExcerptBlack = leftPreview.text.substr(0, leftPreviewOverlap);
              beforeExcerptWhite = leftPreview.text.substr(leftPreviewOverlap,leftPreview.text.length-leftPreviewOverlap);

              const leftExcerptOverlap = Math.max(0,leftExcerptEndIndex-beginningIndex);
              beforeExcerptRed = excerpt.substr(0,leftExcerptOverlap);

              whiteExcerptStartingIndex = leftExcerptOverlap;
            }

            const rightPreview = traverseNWords(endIndex, surroundingPreviewLength);
            whiteExcerpt = excerpt.substr(whiteExcerptStartingIndex,excerpt.length-whiteExcerptStartingIndex);
            if(excerpts.curIndex < excerpts.indexes.length - 1){
              let rightExcerptIndex = excerpts.navigate(1);
              const rightExcerptBeginningIndex = rightExcerptIndex === false ? text.length : traverseNWords(excerpts.indexes[rightExcerptIndex].index,excerpts.indexes[rightExcerptIndex].leftSide).index;

              const rightPreviewOverlap = Math.min(rightPreview.text.length,Math.max(0,rightPreview.index-rightExcerptBeginningIndex));
              afterExcerptBlack = rightPreview.text.substr(rightPreview.text.length-rightPreviewOverlap,rightPreviewOverlap);
              afterExcerptWhite = rightPreview.text.substr(0,rightPreview.text.length-rightPreviewOverlap);
              const rightExcerptOverlap = Math.max(0, endIndex - rightExcerptBeginningIndex);
              afterExcerptRed = excerpt.substr(excerpt.length - rightExcerptOverlap, rightExcerptOverlap);
              whiteExcerpt = excerpt.substr(whiteExcerptStartingIndex,endIndex - rightExcerptOverlap);
              let termFirstIndex = whiteExcerpt.indexOf(curTermsKeys[curTermsKeysIndex]);
              whiteExcerpt = `<span>${whiteExcerpt.substr(0,termFirstIndex)}</span><span class="bg-[#ae9e8b]">${whiteExcerpt.substr(termFirstIndex,curTermsKeys[curTermsKeysIndex].length)}</span><span>${whiteExcerpt.substr(termFirstIndex+curTermsKeys[curTermsKeysIndex].length,whiteExcerpt.length-termFirstIndex+curTermsKeys[curTermsKeysIndex].length)}</span>`
            }
            
            return `
              <div>
                <span class="opacity-50">${leftPreview.index > 0 ? "..." : ""}<span class="bg-[gray] ">${beforeExcerptBlack}</span>${beforeExcerptWhite}</span><span class="bg-[#ffa0a0] text-black">${beforeExcerptRed}</span>${whiteExcerpt}<span class="bg-[#ffa0a0] text-black">${afterExcerptRed}</span><span class="opacity-50">${afterExcerptWhite}<span class="bg-[gray]">${afterExcerptBlack}</span>${rightPreview.index < text.length-1 ? "..." : ""}</span>
              </div>
            `
          })()}
          {#if (excerpts.navigate(1) !== false)}
            <button class="navigateButton" on:click={()=>excerpts.navigate(1,true)}>→</button>
          {/if}

      </div>
      <hr class="hr"/>
    {:else}
      No terms
    {/if}
    <button class="block" on:click={close}>Close</button>
  </div>
</div>
<style>
  .hr{
    background-color:black;
    width:100%;
    height: 2px;
  }
  .navigateButton{
    font-size: 40px;
  }
  *{
    font-family: Arial;
    font-size: 17px;
  }
</style>