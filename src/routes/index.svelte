
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
  let loadingAudio = false;
  let audioFileName = "";
  let showInstructions = false;

  let updateProgressBar = () => {
    const newWidth = (audio.seek() / audio.duration()) * 100;
    audioBarWidth = newWidth + "%";
    setTimeout(() => {
      updateProgressBar();
    }, 1000);
  }
  
  function handleFiles(event) {
    var files = event.target.files;
    audio?.pause();
    audio = new Howl({
      src: URL.createObjectURL(files[0]),
      format: 'mp3'
    });
    audioFileName = files[0].name;
    loadingAudio = true;
    audio.once('load',()=>{
      loadingAudio=false;
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
    for(let i = editor.scroll.length()-1; i >= 0; i--){
      if(text[i] === " "){
        const curWord = text.substr(i+1, counter).trim();
        if(curWord === term.name || term.synonyms.some((synonym)=>synonym === curWord)) {
          editor.formatText(i+1, counter, {background:"#dbd8d1"});
        }
        counter = 0;
      }else if(i===0){
        const curWord = text.substr(i,counter+1).trim();
        if(curWord === term.name || term.synonyms.some((synonym)=>synonym === curWord)) {
          editor.formatText(i, counter+1, {background:"#dbd8d1"});
        }
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
      console.log("term",term);
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
  {#if showInstructions}
  <div class="absolute flex w-screen h-screen justify-center items-center">
    <div class="overflow-scroll bg-[#DBD8D1] border-black border-2 z-20 h-[80vh]">
        <h1 class="text-lg text-center mt-5">Instructions</h1>

        <h1 class="text-lg mt-2">Audio Navigation Keybindings</h1>
        <ul>
          <li>Play/Pause: <span class="rounded-sm bg-[#dbd8d1]">Tab</span></li>
          <li>Seek Left: <span class="rounded-sm bg-[#dbd8d1]">Shift</span> <span class="rounded-sm bg-[#dbd8d1]">{"<"}</span></li>
          <li>Seek Right: <span class="rounded-sm bg-[#dbd8d1]">Shift</span> <span class="rounded-sm bg-[#dbd8d1]">{">"}</span></li>
        </ul>
        <h1 class="text-lg mt-2">Terms</h1>
        <ul>
          <li>Put the cursor on a word and press <span class="rounded-sm bg-[#dbd8d1]">alt</span> <span class="rounded-sm bg-[#dbd8d1]">shift</span> <span class="rounded-sm bg-[#dbd8d1]">+</span> to add it as a term</li>
        </ul>
        <h1 class="text-lg mt-2">Excerpts</h1>
        <ul>
          <li>Press "Set Excerpts" below the textbox.</li>
  
          <li class="mt-3">Next excerpt: <span class="rounded-sm bg-[#dbd8d1]">shift</span> <span class="rounded-sm bg-[#dbd8d1]">{">"}</span></li>
          <li>Previous excerpt: <span class="rounded-sm bg-[#dbd8d1]">shift</span> <span class="rounded-sm bg-[#dbd8d1]">{"<"}</span></li>
  
          <li class="mt-3">Expanding excerpts TL;DR: <span class="rounded-sm bg-[#dbd8d1]">a</span>/<span class="rounded-sm bg-[#dbd8d1]">A</span> and <span class="rounded-sm bg-[#dbd8d1]">d</span>/<span class="rounded-sm bg-[#dbd8d1]">D</span> for left side of excerpt, <span class="rounded-sm bg-[#dbd8d1]">j</span>/<span class="rounded-sm bg-[#dbd8d1]">J</span> and <span class="rounded-sm bg-[#dbd8d1]">l</span>/<span class="rounded-sm bg-[#dbd8d1]">L</span> for right side</li>
          <li class="mt-3">Expand left side of excerpt by 1 word: <span class="rounded-sm bg-[#dbd8d1]">A</span></li>
          <li>Expand left side of excerpt by 10 words:  <span class="rounded-sm bg-[#dbd8d1]">a</span></li>
          <li>Contract left side of except by 1 word: <span class="rounded-sm bg-[#dbd8d1]">D</span></li>
          <li>Contract left side of except by 10 words: <span class="rounded-sm bg-[#dbd8d1]">d</span></li>
          
          <li class="mt-3">Expand right side of excerpt by 1 word: <span class="rounded-sm bg-[#dbd8d1]">L</span></li>
          <li>Expand right side of excerpt by 10 words:  <span class="rounded-sm bg-[#dbd8d1]">l</span></li>
          <li>Contract right side of except by 1 word: <span class="rounded-sm bg-[#dbd8d1]">J</span></li>
        </ul>
        <div class="flex justify-center w-full">
          <button class="bg-red-300" on:click={()=>{document.getElementById('container').style.opacity = 1; showInstructions=false}}>Close</button>

        </div>
    </div>  
  </div>
  {/if}
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
            terms[term.name] = {name:term.name, synonyms: term.synonyms, parents: term.parents, children:term.children,excerpts: term.excerpts}
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
        document.getElementById('container').style.opacity = 1;
      }}>Delete</button>
    </div>
  </div>
{/if}
</div>
<div class="p-5" id="container">

    
  <div class="flex items-center justify-center w-1/5 h-1/1">
    <div class="flex flex-col w-1/1 grow">
      <label id="uploadLabel" class="flex justify-center m-2" for="upload">
        <div id="uploadFile" class="cursor-pointer border-black border-2 p-2 flex justify-center items-center">Upload Audio File...</div>
      </label>
      {#if loadingAudio}<h1 class="text-lg bg-[green]">Wait, loading {audioFileName}...</h1>{/if}
      {#if !loadingAudio && audioFileName.length>0}<h1 class="text-lg bg-[green]">Loaded {audioFileName}</h1>{/if}
      <input type="file" id="upload" />
      <div class="flex justify-center">
        <button style="box-shadow:2px 3px;border:2px solid black" on:click={()=>{document.getElementById('container').style.opacity = 0.2;showInstructions=true}}>Instructions</button>
      </div>
    </div>
    
  </div>
  <div class="flex-col m-10 w-4/5 p-0 h-1/1">
    <div class="flex justify-center mb-2">
      <div id="progressBarContainer" >
        <div id="progressBar" style="--the-width:{audioBarWidth}"></div>
      </div>
    </div>
    <div>
      Terms: 
      {#each Object.keys(terms) as key (key)}
        <span class="cursor-pointer rounded-sm bg-[#dbd8d1]" on:click={()=>{
          removeHilightsInEditor(terms[key]);
          deletedTerms.push(terms[key]);
          deletedTerms = deletedTerms;
          delete terms[key];
        }}>{key}</span>
        {#if terms[key].synonyms.length>0}→{/if}
        {#each terms[key].synonyms as synonym, i}
          {synonym}{i<terms[key].synonyms.length-1?",":""}
        {/each}
        <span>.&nbsp;</span>
      {/each}
    </div>
    <div class="bg-[#dbd8d1] h-[90%]">
      <div id="editor">
      </div>
    </div>
    <div class="flex justify-center items-center h-[10%]">
      <button on:click={setExcerpts}>Set Excerpts</button>
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