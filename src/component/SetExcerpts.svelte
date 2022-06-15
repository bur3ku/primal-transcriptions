<script>
export let editor;
export let terms;
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
  excerpts.setIndexes();
  excerpts.curIndex = 0;
}

const nextTerm = () => {
  curTermsKeysIndex++;
  excerpts.setIndexes();
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
  setIndexes: function() {
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
          console.log("direction",direction);
          if(direction === 1 && this.indexes[i].disabledFromLeft) {
            this.indexes[i].disabledFromLeft = false;
            console.log(i,"disabledFromLeft = false");
            excerpts = excerpts;
          }
          else if(direction === -1 && this.indexes[i].disabledFromRight) {
            console.log(i,"disabledFromRight = false");
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
          console.log(i,"disabledFromLeft = true");
          this.indexes[i].disabledFromLeft = true;
          excerpts = excerpts;
        }
        else if(direction === -1 && !this.indexes[i].disabledFromRight) {
          console.log(i,"disabledFromRight = true");
          this.indexes[i].disabledFromRight = true;
          excerpts = excerpts;
        }
      }
    }
    return false;
  }
}
excerpts.setIndexes();

let beginningIndex;
let endIndex;
beginningIndex = traverseNWords(excerpts.indexes[excerpts.curIndex].index, excerpts.indexes[excerpts.curIndex].leftSide).index;
endIndex = traverseNWords(excerpts.indexes[excerpts.curIndex].index, excerpts.indexes[excerpts.curIndex].rightSide+1).index;

</script>
<div class="absolute flex w-screen h-screen justify-center items-center">
  <div class="flex flex-col items-center max-w-[400px] border-black border-2 z-20">
    <div class="flex">
      {#if (excerpts.navigate(-1) !== false)}
        <button on:click={()=>excerpts.navigate(-1,true)}>Prev.</button>
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
          console.log(text.substr(endIndex,20));
          whiteExcerpt = excerpt.substr(whiteExcerptStartingIndex,endIndex - rightExcerptOverlap);
        }
        
        return `
          <div>
            <span class="opacity-50">${leftPreview.index > 0 ? "..." : ""}<span class="bg-black text-white">${beforeExcerptBlack}</span>${beforeExcerptWhite}</span><span class="bg-[red] text-black">${beforeExcerptRed}</span>${whiteExcerpt}<span class="bg-[red] text-black">${afterExcerptRed}</span><span class="opacity-50">${afterExcerptWhite}<span class="bg-black text-white">${afterExcerptBlack}</span>${rightPreview.index < text.length-1 ? "..." : ""}</span>
          </div>
        `
      })()}
      {#if (excerpts.navigate(1) !== false)}
        <button on:click={()=>excerpts.navigate(1,true)}>Next</button>
      {/if}
    </div>
    <div class="flex">
      <button on:click={prevTerm}>{curTermsKeysIndex > 0 ? "Prev." : ""}</button>
      <span>{curTermsKeys[curTermsKeysIndex]}</span>
      <button on:click={nextTerm}>{curTermsKeysIndex<curTermsKeys.length-1?"Next":""}</button>
    </div>
  </div>
</div>
