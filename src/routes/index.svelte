<script>
  import { onMount } from "svelte";
  import {Howl} from 'howler';
  let audio;
  let paused = true;
  let shiftIsPressedDown = false;
  let width = "0%";
  let updateProgressBar = () => {
    console.log("test");
    console.log((audio.seek() / audio.duration()) * 100);
    const newWidth = (audio.seek() / audio.duration()) * 100;
    width = newWidth + "px";
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
onMount(()=>{
  audio = document.getElementById("audio");
  document.getElementById("upload").addEventListener("change", handleFiles, false);
  document.addEventListener("keyup", (e)=>{
    if(e.key==="Shift")shiftIsPressedDown=false;
  })
  document.addEventListener("keydown",(e)=>{
    if(e.key==="Shift"){
      e.preventDefault();
      shiftIsPressedDown=true;
    } else if(e.key==="<" && shiftIsPressedDown){
      e.preventDefault();
      updateProgressBar();
      audio.seek(Math.max(audio.seek() - 5, 0));
      audio.play();
    } else if(e.key===">" && shiftIsPressedDown) {
      e.preventDefault();
      updateProgressBar();
      audio.seek(Math.min(audio.seek() + 5, audio.duration()));
    } else if (e.key==="Tab"){
      e.preventDefault();
      updateProgressBar();
      console.log(audio.seek())
      paused?audio.play():audio.pause();
    }
  });
})

</script>

<div id="container">
  <div class="flex items-center justify-center w-1/5 h-1/1">
    <div class="flx flex-col w-1/1">
      <label for="upload">
        <div class="border-black border-2 inline-block p-2">Upload Audio File...</div>
      </label>
      <input type="file" id="upload" />
      <div class="flex justify-center">
        <div id="progressBarContainer" >
          <div id="progressBar" style="--the-width:{width}"></div>
        </div>
      </div>
     
    </div>
    
  </div>
  <div id="right">
    <h1 class="text-3xl font-bold underline">hello world</h1>
  </div>
</div>


<style>
  #progressBarContainer{
    height: 20px;
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
  #right{
    width:80%;
    height:100%;
    background-color: rgb(174, 158, 139);
  }
  :global(body){
    margin:0;
    padding:0;
    background-color:#dbd8d1
  }

</style>