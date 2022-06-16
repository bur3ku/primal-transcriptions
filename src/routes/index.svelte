
<script>
  import { onMount } from "svelte";
  import {Howl} from 'howler';
  import SetExcerpts from '../component/SetExcerpts.svelte';

  let terms = {}

  let deletedTerms = [];

  let resetBackground = false;
  let audio;
  let paused = true;
  let altIsPressedDown = false;
  let controlIsPressedDown = false;
  let audioBarWidth = "0%";
  let term = {name:"",parents:[],children:[],synonyms:[],excerpts: []};
  let newSynonym = "";
  let newParent = "";
  let editor;
  let settingExcerpts = false;
  let settingTerm = false;

  let updateProgressBar = () => {
    const newWidth = (audio.seek() / audio.duration()) * 100;
    audioBarWidth = newWidth + "px";
    setTimeout(() => {
      updateProgressBar();
    }, 1000);
  }
  
  function handleFiles(event) {
    var files = event.target.files;
    audio = new Howl({
      src: URL.createObjectURL(files[0]),
      format: 'mp3'
    });
    audio.once('load',()=>{
      updateProgressBar();
    })
    audio.on('pause',()=>{
      paused = true;
    })
    audio.on('play', ()=>{
      paused=false;
    })
    audio.play();
  }
  const removeHilightsInEditor = (term) => {
    const text = editor.getText();
    let counter = 0;
    for(let i = editor.scroll.length()-1; i > 0; i--){
      if(text[i] === " "){
        const curWord = text.substr(i+1, counter).trim();
        term.synonyms.forEach((synonym)=>{
        })
        if(curWord === term.name || term.synonyms.some((synonym)=>synonym === curWord)) {
          editor.formatText(i+1, counter, {background:"#dbd8d1"});
        }
        counter = 0;
      }else{
        counter++;
      }
    }
  }

  const addHilightsInEditor = (term) => {
    //hilight all instances of the term and its siblings
    const text = editor.getText(0, editor.scroll.length()-1);
    const arr = text.split(" ");
    let index = 0;
    for(let word of arr){
      if(word === term.name || term.synonyms.some((sibling)=>sibling===word)){
        editor.formatText(index, word.length, {background:"#ae9e8b"});
      }
      resetBackground = true;
      index += word.length + 1;
    }
  }
onMount(()=>{
  editor = new Quill('#editor', {
    modules: { 
      toolbar: [
          {'size': ['small', false, 'large', 'huge']}]
    },
    theme: 'snow',
    placeholder: "Plain text. Sizes are for viewing convenience and will be ignored on export"
  });
  editor.setSelection(editor.scroll.length()-1,0);
  editor.on('text-change', (delta, oldDelta, source) => {
    if(source === 'user'){
      const selectionIndex = editor.getSelection()?.index;

      if(resetBackground){
        editor.formatText(selectionIndex - 1, 1, {background:"#dbd8d1"});
        resetBackground = false;
      }
      const text = editor.getText(0, editor.scroll.length()-1);
      const typedLetter = delta.ops[0].insert || delta.ops[1]?.insert;
      
      //get beginning index of word surrounding cursor
      const getBeginningIndex = (init) => {
        for(let i = init; i >= 0; i--){
          if(text[i]===" "){
            return i+1;
          }
        }
        return 0;
      }
      
      //get end index of word surrounding cursor
      const getEndIndex = (init) => {
        for(let i = init; i < editor.scroll.length()-1; i++){
          if(text[i]===" "){
            return i;
          }
        }
        return editor.scroll.length()-1;
      }

      const find = (beginningIndex,endIndex) => {
        let found = false;
        let curWord = text.substr(beginningIndex, endIndex - beginningIndex);
        Object.keys(terms).forEach((key)=>{
          if(key === curWord || terms[key].synonyms.some((synonym)=>synonym===curWord)){
            editor.formatText(beginningIndex, endIndex-beginningIndex, {background:"#ae9e8b"})
            resetBackground = true;
            found = true;
            return;
          }
        })

        if(!found)editor.formatText(beginningIndex, endIndex-beginningIndex, {background:"#dbd8d1"})


      }

      if(typedLetter === " "){
        find(getBeginningIndex(selectionIndex-2),selectionIndex - 1);
        find(selectionIndex, getEndIndex(selectionIndex))
      }else{
        find(getBeginningIndex(selectionIndex===0?0:selectionIndex-1),getEndIndex(selectionIndex===0?0:selectionIndex-1));
      }
    }
  })

  audio = document.getElementById("audio");
  document.getElementById("upload").addEventListener("change", handleFiles, false);
  document.addEventListener("keyup", (e)=>{
    if(e.key==="Alt")altIsPressedDown=false;
    if(e.key==="Control")controlIsPressedDown=false;
  })
  document.addEventListener("keydown",(e)=>{
    //if closing the term editor by pressing "Enter"
    if(e.key==="Control")controlIsPressedDown = true;
    
    //add or modify a term by pressing "alt" + "+"
    else if(e.key==="+" && altIsPressedDown){
      document.getElementById('container').style.opacity = 0.2;
      const text = editor.getText(0, editor.scroll.length()-1);
      const selectionIndex = editor.getSelection().index;
      let counter = 0;
      let wordBeginningIndex;
      for(let i = selectionIndex; i >= 0; i--){
        if(text?.[i-1]===" " || i===0){
          wordBeginningIndex = i;
          break;
        }
        counter++;
      }
      for(let i = selectionIndex; i<text.length;i++){
        if(text[i]===" " || i===text.length-1){
          break;
        }
        counter++;

      }
      term.name = text.substr(wordBeginningIndex,counter);
      Object.keys(terms).forEach((key)=>{
        if(key===term.name || terms[key].synonyms.some((synonym)=>synonym===term.name)){
          term = terms[key];
          term.name=key;
        }
      });
      setTimeout(()=>document.getElementById("synonym-input").select(),100);
    }
    else if(e.key==="Alt") altIsPressedDown=true;
    else if(e.key==="<"){
      if(settingExcerpts) return;
      e.preventDefault();
      updateProgressBar();
      audio.seek(Math.max(audio.seek() - 5, 0));
      audio.play();
    } else if(e.key===">") {
      if(settingExcerpts) return;
      e.preventDefault();
      updateProgressBar();
      audio.seek(Math.min(audio.seek() + 5, audio.duration()));
    } else if (e.key==="Tab"){
      if(settingExcerpts || term.name.length>0) return;
      e.preventDefault();
      updateProgressBar();
      paused?audio.play():audio.pause();
    }
  });
})

const setExcerpts = () => {
  document.getElementById('container').style.opacity = 0.2;
  settingExcerpts = true;
}

const dontSetExcerpts = () => {
  document.getElementById('container').style.opacity = 1;
  settingExcerpts = false;
}
</script>
<div>
  {#if settingExcerpts}
    <SetExcerpts close={dontSetExcerpts} editor={editor} terms={terms}/>
  {/if}
  {#if term.name.length > 0}
  
  <div class="absolute flex w-screen h-screen justify-center items-center">
    <div class="flex flex-col items-center max-w-[400px] border-black border-2 z-20">
      <p class="m-2">Term: <span class="rounded-sm bg-[#ae9e8b] pr-1 pl-1">{term.name}</span></p>
      <input placeholder="parent" bind:value={newParent} on:keypress={(e)=>{
        if(e.key==="Enter"){
          if(newParent in terms) {
            terms[newParent].children.push(term.name);
          }
          else {
            terms[newParent] = {name:newParent,parents:[],children:[term.name],synonyms:[],excerpts: []};
          }
          term = terms[newParent];
          newParent = "";
        }
      }}/>
      <input id="synonym-input" placeholder="synonym(s)" bind:value={newSynonym} on:keypress={(e)=>{
        if(e.key==="Enter"){
          if(controlIsPressedDown || newSynonym.length===0){
            e.preventDefault();
            addHilightsInEditor(term);
            terms[term.name] = {synonyms: term.synonyms, parents: term.parents,excerpts: term.excerpts}
            term = {name:"",parents:[],children:[],synonyms:[],excerpts: []};
            editor.setSelection(editor.scroll.length()-1,0);
            document.getElementById('container').style.opacity = 1;
          }
          newSynonym.split(",").forEach((synonym)=>{
            term.synonyms.push(synonym.trim());
          })
          term = term;
          newSynonym = "";
        }
      }}/>
      <div class="flex flex-wrap gap-2 p-2">
        Synonyms:
        {#each term.synonyms as synonym}
          <p class="rounded-sm bg-[#ae9e8b] pr-1 pl-1">{synonym}</p>
        {/each}
      </div>
      <div class="flex flex-wrap gap-2 p-2">
        Children:
        {#each term.children as child}
          <p class="cursor-pointer rounded-sm bg-[#ae9e8b] pr-1 pl-1">{child}</p>
        {/each}
      </div>
      <button on:click={()=>{
        deletedTerms.push(term);
        deletedTerms = deletedTerms;
        delete terms[term.name];
        removeHilightsInEditor(term);
        term={name:"",synonyms:[],parents:[],children:[],excerpts: []}
      }}>Delete</button>
    </div>
  </div>
{/if}
<div class="p-5" id="container">

    
  <div class="flex items-center justify-center w-1/5 h-1/1">
    <div class="flex flex-col w-1/1 grow">
      <label id="uploadLabel" class="flex justify-center m-2" for="upload">
        <div id="uploadFile" class="border-black border-2 p-2 flex justify-center items-center">Upload Audio File...</div>
      </label>
      <input type="file" id="upload" />
     
    </div>
    
  </div>
  <div class="flex-col m-10 w-4/5 p-0 h-1/1">
    <div class="flex justify-center mb-2">
      <div id="progressBarContainer" >
        <div id="progressBar" style="--the-width:{audioBarWidth}"></div>
      </div>
    </div>

    <div class="bg-[#dbd8d1] h-[90%]">
      <div id="editor">
        <p>test neurological 1 neurological 2 neurological 3 neurological 4 neurological 5 test neurological 6 neurological 7 neurological 8 neurological 9 neurological 10 neurological 11 neurological 12 neurological 13 neurological 14 neurological 15 neurological 16 test</p>
      </div>
    </div>
    <div class="flex justify-center items-center h-[10%]">
      <button on:click={setExcerpts}>Set Excerpts</button>
    </div>

  </div>

</div>
</div>


<style>
  #uploadFile{
    box-shadow: 3px 2px;
  }
  #editor{
    height: calc(100% - 40px);
  }
  #uploadLabel{
    height: 40px;
  }
  #progressBarContainer{
    height: 40px;
    border: 2px solid black;
    width: 100%;
  }
  #progressBar{
    width:var(--the-width);
    background-color: blueviolet;
    height:100%;
  }
  #upload{
    display:none;
  }
  #container{
    display:flex;
    width:100%;
    height:100vh;
  }
  :global(body){
    margin:0;
    padding:0;
    background-color:#ae9e8b
  }

</style>