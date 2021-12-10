<template>
  <div id="app">
    <input
      type="file"
      @change="onFileChange"
    />

    <img
      v-if="url"
      :src="url"
    />
    <button v-on:click='onUpload' v-if="url">Upload</button>
    <ul id="example-1">
  <li v-for="item in filesInfo" :key="item.name">
    {{ item.name }}
      <button v-on:click='loadChunks(item)' v-if="filesInfo">View</button>
  </li>
   
</ul>
  
  </div>
</template>

<script>
import { ref, onMounted } from "vue"
import { counter,  idlFactory, canisterId } from "canisters/counter"
import { Actor, HttpAgent } from '@dfinity/agent';
const MAX_CHUNK_SIZE = 1024 * 500; 
const getReverseFileExtension = (type)  => {
  switch(Object.keys(type)[0]) {
    case 'jpeg':
      return  'image/jpeg';
    case 'gif':
      return  'image/gif'; 
    case 'jpg':
      return  'image/jpg';       
    case 'png':
      return  'image/png';
    case 'svg':
      return  'image/svg';
    case 'avi':
      return  'video/avi';
    case 'mp4':
      return  'video/mp4';
    case 'aac':
      return  'video/aac';
    case 'wav':
      return  'audio/wav';
    case 'mp3':
      return  'audio/mp3';                                                                                                              
    default :
    return "";
  }
};

const getFileExtension = (type)  => {
  switch(type) {
    case 'image/jpeg':
      return { 'jpeg' : null };
    case 'image/gif':
      return { 'gif' : null };
    case 'image/jpg':
      return { 'jpg' : null };
    case 'image/png':
      return { 'png' : null };          
    case 'image/svg':
      return { 'svg' : null };          
    case 'video/avi':
      return { 'avi' : null };                            
    case 'video/aac':
      return { 'aac' : null };
    case 'video/mp4':
      return { 'mp4' : null };        
    case 'audio/wav':
      return { 'wav' : null };                         
    case 'audio/mp3':
      return { 'mp3' : null };
    default :
    return null;
  }
};
let agentOptions = {};
if (process.env.NODE_ENV === 'development') {
  agentOptions = { ...agentOptions,  host: 'http://localhost:7000' };
}
 async function getBackendActor() {
  const agent = new HttpAgent(agentOptions);
  // for local development only, this must not be used for production
  if (process.env.NODE_ENV === 'development') {
    await agent.fetchRootKey();
  }

  const backend = Actor.createActor(idlFactory, { agent, canisterId: canisterId });

  return backend;
}
function b64toBlob(b64Data, contentType = "", sliceSize = 512) {
  const byteCharacters = atob(b64Data)
  const byteArrays = []

  for (let offset = 0; offset < byteCharacters.length; offset += sliceSize) {
    const slice = byteCharacters.slice(offset, offset + sliceSize)

    const byteNumbers = new Array(slice.length)
    for (let i = 0; i < slice.length; i++) {
      byteNumbers[i] = slice.charCodeAt(i)
    }

    const byteArray = new Uint8Array(byteNumbers)
    byteArrays.push(byteArray)
  }
  const blob = new Blob(byteArrays, { type: contentType })
  return blob
}
const encodeArrayBuffer = (file) => Array.from(new Uint8Array(file))
const processAndUploadChunk = async (
  blob,
  byteStart,
  fileId,
  chunk,
  fileSize,
) => {
  const blobSlice = blob.slice(
    byteStart,
    Math.min(Number(fileSize), byteStart + MAX_CHUNK_SIZE),
    blob.type,
  )

  const bsf = await blobSlice.arrayBuffer()
  const ba = await getBackendActor()
  // console.log(fileId);
  // console.log(chunk);
  // console.log(fileSize);
  // console.log(encodeArrayBuffer(bsf));
  return ba.putFileChunks(fileId, chunk, fileSize, encodeArrayBuffer(bsf))
}


export default {
  name: "Intro",
  setup: () => {
   
    

  },
  data() {
    return {
      url: null,
      view: null,
      filesInfo: null,
      fileData: "Drag and drop a file or select add File",
      fileI: {
        name: "",
        type: "",
        size: 0,
        blob: new Blob(),
        width: 0,
        file: 0,
        height: 0,
      },
    }
  },

  methods: {
    onFileChange(e) {
      const file = e.target.files[0]
      this.url = URL.createObjectURL(file)
      const reader = new FileReader()
      reader.readAsDataURL(file)
      reader.onloadend = () => {
        if (reader.result === null) {
          throw new Error("file empty...")
        }
        let encoded = reader.result.toString().replace(/^data:(.*,)?/, "")
        if (encoded.length % 4 > 0) {
          encoded += "=".repeat(4 - (encoded.length % 4))
        }
        const blob = b64toBlob(encoded, file.type)
        console.log(blob)
        const fileInfo = {
          name: file.name,
          type: file.type,
          size: file.size,
          blob: blob,
          file: file,
          width: file.width,
          height: file.height,
        }
        this.fileData = file.name + " | " + Math.round(file.size / 1000) + " kB"
        this.fileI = fileInfo
      }
    },
      async getFilesInfo   ()  {
    const ba = await getBackendActor();
    const files = await ba.getAllFiles();
    
    this.filesInfo=files
console.log(this.filesInfo); 
 

  },
    async onUpload(event){
           event.preventDefault();
      const fileExtension = getFileExtension(this.fileI.type);
      console.log(fileExtension);
      const errors = [];
      if (this.fileI === null || this.fileI === undefined || fileExtension === null) {
        errors.push("File not valid!");
      }
      if (this.fileI.size > 10550000) {
        errors.push("File size shouldn't be bigger than 10mb");
      }

      if (errors.length > 0) {
        setErrros(errors);
        return;
      }
      
      const t0 = performance.now();
      console.log('upload started...');
      
      const fileInfo  = {
        name: Math.random().toString(36).substring(2),
        createdAt: BigInt(Number(Date.now() * 1000)),
        size: BigInt(this.fileI.size),
        chunkCount: BigInt(Number(Math.ceil(this.fileI.size / MAX_CHUNK_SIZE))),
        // @ts-ignore
        extension: fileExtension,
      };
      const ba = await getBackendActor();
    
      // const authenticated = await authClient.isAuthenticated();
      // console.log(authenticated);
      
       const fileId = (await ba.putFileInfo(fileInfo))[0] ;
      // console.log(fileId);
   
      const blob = this.fileI.blob;
      const putChunkPromises = [];
      let chunk = 1;
      for (let byteStart = 0; byteStart < blob.size; byteStart += MAX_CHUNK_SIZE, chunk++ ) {
        putChunkPromises.push(
          processAndUploadChunk(blob, byteStart, fileId, chunk, this.fileI.size)
        );
      }
      await Promise.all(putChunkPromises);
      await ba.updateStatus();
    
      this.fileData= 'Drag and drop a file or select add File';
      const t1 = performance.now();
      console.log("Upload took " + (t1 - t0) / 1000 + " seconds.")
      this.view= 'uploaded';

    },
    
   async loadChunks  (fi)  {
    
   
    
    const ba = await getBackendActor();
    // const chunk = await ba.getFileChunk(fi.fileId, BigInt(1));
    // console.log(chunk);
    const chunks = [];
    for (let i = 1; i <= Number(fi.chunkCount); i++) {
      const chunk = await ba.getFileChunk(fi.fileId, BigInt(i), fi.cid);
      chunks.push(new Uint8Array(chunk[0]).buffer);
    }
    const blob = new Blob(chunks, { type: getReverseFileExtension(fi.extension)} );
    const url = URL.createObjectURL(blob);
    
    console.log(url)
    alert(url);
  }
  },
   created: function(){
        this.getFilesInfo()
    }
}
</script>

<style scoped>
.App-logo {
  height: 15vmin;
  pointer-events: none;
}

.App-header {
  margin-top: 150px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  font-size: calc(10px + 2vmin);
}

.App-link {
  color: rgb(26, 117, 255);
}

.demo-button {
  background: #a02480;
  padding: 0 1.3em;
  margin-top: 1em;
  border-radius: 60px;
  font-size: 0.7em;
  height: 35px;
  outline: 0;
  border: 0;
  cursor: pointer;
  color: white;
}

.demo-button:active {
  color: white;
  background: #979799;
}
</style>
