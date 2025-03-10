<script setup lang="ts">
import { SvgIcon } from '@/components/common';
import an_main from './an_main.vue'
import aiTextSetting from '../mj/aiTextSetting.vue';
import wavSetting from './wavSetting.vue';
import { WavRecorder, WavStreamPlayer } from '@openai/realtime-wavtools';
import { onMounted, ref, watch } from 'vue';
import { mlog,RealtimeEvent,instructions } from '@/api';
import { WavRenderer } from '@/utils/wav_renderer';
import { RealtimeClient } from '@openai/realtime-api-beta';
import { ItemType } from '@openai/realtime-api-beta/dist/lib/client.js';
import { useMessage ,NModal,NButton} from 'naive-ui';
import { gptServerStore } from '@/store';
import { t } from '@/locales';
const wavRecorderRef=  ref<WavRecorder>( new  WavRecorder({ sampleRate: 24000 })) 
const wavStreamPlayerRef=  ref<WavStreamPlayer>( new WavStreamPlayer({ sampleRate: 24000 })) 
const clientCanvasRef = ref<HTMLCanvasElement|null>(null);
const serverCanvasRef = ref<HTMLCanvasElement|null>(null);
const items= ref<ItemType[]>([]); 
const realtimeEvents= ref<RealtimeEvent[]>([]);
const clientRef= ref<RealtimeClient>();
const ms= useMessage();
const st= ref({apikey:'', isConnect:false,baseUrl:'',isRealtime:true,msg:'Waiting',isClosed:false,showSetting:false })
const edmit= defineEmits(['close'])

watch( ()=> wavRecorderRef.value,() => {
    const wavRecorder= wavRecorderRef.value;  
    
        const clientCanvas = clientCanvasRef.value;
        const wavStreamPlayer = wavStreamPlayerRef.value;
        let clientCtx: CanvasRenderingContext2D | null = null;
        if (clientCanvas) {
          if (!clientCanvas.width || !clientCanvas.height) {
            clientCanvas.width = clientCanvas.offsetWidth;
            clientCanvas.height = clientCanvas.offsetHeight;
          }
          clientCtx = clientCtx || clientCanvas.getContext('2d');
          if (clientCtx) {
            clientCtx.clearRect(0, 0, clientCanvas.width, clientCanvas.height);
            const result = wavRecorder.recording
              ? wavRecorder.getFrequencies('voice')
              : { values: new Float32Array([0]) };
            WavRenderer.drawBars(
              clientCanvas,
              clientCtx,
              result.values,
              '#0099ff',
              20,
              0,
              2
            );
          }
        }

        const serverCanvas = serverCanvasRef.value;
        let serverCtx: CanvasRenderingContext2D | null = null;
        if (serverCanvas) {
          if (!serverCanvas.width || !serverCanvas.height) {
            serverCanvas.width = serverCanvas.offsetWidth;
            serverCanvas.height = serverCanvas.offsetHeight;
          }
          serverCtx = serverCtx || serverCanvas.getContext('2d');
          if (serverCtx) {
            serverCtx.clearRect(0, 0, serverCanvas.width, serverCanvas.height);
            const result = wavStreamPlayer.analyser
              ? wavStreamPlayer.getFrequencies('voice')
              : { values: new Float32Array([0]) };
             
            WavRenderer.drawBars(
              serverCanvas,
              serverCtx,
              result.values,
              '#009900',
              20,
              0,
              2
            );
          }
        }

},{deep:true,immediate:true});

const go= async()=>{
    st.value.msg=  t('mj.rtconecting')
    if(st.value.isConnect){
        //mlog("isConnect yes!"  )
        ms.info("isConnect yes!");
        return;
    }
    
    if(!clientRef.value || !st.value.isConnect ){
        if(!st.value.apikey){
            
            ms.error("api key is null");
            return;
        }
        if(!st.value.baseUrl){ 
            ms.error("baseUrl is null");
            return;
        }
        //ms.info("go");
        //console.log("RealtimeClient", st.value.apikey )

        clientRef.value= new RealtimeClient( { 
            apiKey:st.value.apikey,
            dangerouslyAllowAPIKeyInBrowser: true,
            baseUrl: st.value.baseUrl,
            model: gptServerStore.myData.REALTIME_MODEL?gptServerStore.myData.REALTIME_MODEL: 'gpt-4o-realtime-preview-2024-10-01' 
          }
        )
    }
    //mlog("go", st.value.apikey )
    const client= clientRef.value
    const wavRecorder= wavRecorderRef.value
    const wavStreamPlayer= wavStreamPlayerRef.value
   
    try{
    // Connect to microphone
        await wavRecorder.begin();
    }catch(e){
        st.value.msg=t('mj.rtservererror2') 
        ms.error(st.value.msg);
        return 
    }
    // Connect to realtime API
    try{
        await client.connect(); 
    }catch(e ){
        st.value.msg= t('mj.rtservererror')
        ms.error( st.value.msg);

        return 
    }

    // Connect to audio output
    await wavStreamPlayer.connect();

    st.value.isConnect=true

    client.sendUserMessageContent([
      {
        type: `input_text`,
        text: `hello`,
      },
    ]);
    

    client.updateSession({
      turn_detection:  { type: 'server_vad' },
    });
    

    await wavRecorder.record((data: { mono: Int16Array | ArrayBuffer; }) => {
        try{ 
            client.appendInputAudio(data.mono)
            st.value.msg=  t('mj.rtsuccess')
        }catch(e){
            disconnectConversation();
            // st.value.msg= t('mj.checkkey')
            // ms.error(st.value.msg);
            // mlog("appendInputAudio error", e )
            return
        }
    });

    myListen();
}

const disconnectConversation= async()=>{
    const wavRecorder= wavRecorderRef.value
    const wavStreamPlayer= wavStreamPlayerRef.value
    //clientRef.value?.disconnect();
    st.value.isConnect=false
    const client= clientRef.value;
    //client?.reset();
    client?.disconnect();
    await wavRecorder.end();
    await wavStreamPlayer.interrupt();
    st.value.msg=t('mj.rjcloded')
    ms.success( st.value.msg);
}

/**
 * Type for all event logs
 */


const myListen=()=>{
    const client= clientRef.value;
    const wavRecorder= wavRecorderRef.value
    const wavStreamPlayer= wavStreamPlayerRef.value

    if( !client){
        return
    }
    // Set instructions
    client.updateSession({ 
        instructions:  gptServerStore.myData.REALTIME_SYSMSG?  gptServerStore.myData.REALTIME_SYSMSG: instructions,
     });

    if( gptServerStore.myData.TTS_VOICE && ['alloy','shimmer','echo'].indexOf( gptServerStore.myData.TTS_VOICE)>-1) {
        client.updateSession({ voice: gptServerStore.myData.TTS_VOICE });
        mlog('log','voice', gptServerStore.myData.TTS_VOICE)

    }
    // Set transcription, otherwise we don't get user transcriptions back

    if(gptServerStore.myData.REALTIME_IS_WHISPER) client.updateSession({ input_audio_transcription: { model: 'whisper-1' } });

    // Add tools
    client.addTool(
      {
        name: 'set_memory',
        description: 'Saves important data about the user into memory.',
        parameters: {
          type: 'object',
          properties: {
            key: {
              type: 'string',
              description:
                'The key of the memory value. Always use lowercase and underscores, no other characters.',
            },
            value: {
              type: 'string',
              description: 'Value can be anything represented as a string',
            },
          },
          required: ['key', 'value'],
        },
      },
      async ({ key, value }: { [key: string]: any }) => {
        // setMemoryKv((memoryKv) => {
        //   const newKv = { ...memoryKv };
        //   newKv[key] = value;
        //   return newKv;
        // });
        return { ok: true };
      }
    );
    client.addTool(
      {
        name: 'get_weather',
        description:
          'Retrieves the weather for a given lat, lng coordinate pair. Specify a label for the location.',
        parameters: {
          type: 'object',
          properties: {
            lat: {
              type: 'number',
              description: 'Latitude',
            },
            lng: {
              type: 'number',
              description: 'Longitude',
            },
            location: {
              type: 'string',
              description: 'Name of the location',
            },
          },
          required: ['lat', 'lng', 'location'],
        },
      },
      async ({ lat, lng, location }: { [key: string]: any }) => {
        // setMarker({ lat, lng, location });
        // setCoords({ lat, lng, location });
        const result = await fetch(
          `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lng}&current=temperature_2m,wind_speed_10m`
        );
        const json = await result.json();
        // const temperature = {
        //   value: json.current.temperature_2m as number,
        //   units: json.current_units.temperature_2m as string,
        // };
        // const wind_speed = {
        //   value: json.current.wind_speed_10m as number,
        //   units: json.current_units.wind_speed_10m as string,
        // };
        // setMarker({ lat, lng, location, temperature, wind_speed });
        return json;
      }
    );


    // handle realtime events from client + server for event logging
    client.on('realtime.event', (realtimeEvent: RealtimeEvent) => {
        setRealtimeEvents(realtimeEvent);
    });
    client.on('error', (event: any) =>{
         ms.error('发生错误：'+event);
         console.error('error.event>>',event);
    });
    client.on('conversation.interrupted', async () => {
      const trackSampleOffset = await wavStreamPlayer.interrupt();
      if (trackSampleOffset?.trackId) {
        const { trackId, offset } = trackSampleOffset;
        await client.cancelResponse(trackId, offset);
      }
    });
    client.on('conversation.updated', async ({ item, delta }: any) => {
      const items = client.conversation.getItems();
      if (delta?.audio) {
        wavStreamPlayer.add16BitPCM(delta.audio, item.id);
      }
      if (item.status === 'completed' && item.formatted.audio?.length) {
        const wavFile = await WavRecorder.decode(
          item.formatted.audio,
          24000,
          24000
        );
        item.formatted.file = wavFile;
      }
      setItems(items);
    });
}
const setItems=(iitems: ItemType[])=>{
    //mlog("setItems", iitems.length, iitems  )
    items.value=iitems
}
const setMemoryKv=(kv: { [key: string]: any }) => {
    
}
const setRealtimeEvents=(realtimeEvent: RealtimeEvent )=>{
     //mlog("setRealtimeEvents", realtimeEvent.event ,  realtimeEvent  )
     let ev= {...realtimeEvent.event}
     if(ev.type=="error" && ev.error && ev.error.message){
        ms.error(ev.error.message)
     }
    
      const lastEvent =  realtimeEvents.value[ realtimeEvents.value.length - 1];
        if (lastEvent?.event.type === realtimeEvent.event.type) {
          // if we receive multiple events in a row, aggregate them for display purposes
          lastEvent.count = (lastEvent.count || 0) + 1;
          return  realtimeEvents.value.slice(0, -1).concat(lastEvent);
        } else {
          return  realtimeEvents.value.concat(realtimeEvent);
        }
}
const loadConfig=()=>{
    let base=gptServerStore.myData.OPENAI_API_BASE_URL;
    const key=gptServerStore.myData.OPENAI_API_KEY;
    st.value.apikey=key
    if(base){
        base= base.replaceAll('https://','wss://').replaceAll('http://','ws://')
        st.value.baseUrl= base+'/v1/realtime'
    }
    //mlog('baseUrl', st.value.baseUrl, key )
    if( st.value.baseUrl && st.value.apikey){
        go()
    }
}
   
 
onMounted(()=>{ 
    loadConfig();
})
const close=()=>{
    st.value.isClosed=true
    try {
      disconnectConversation();
    } catch (error) {
        
    }

    //edmit('close')
    setTimeout(() => {
        edmit('close')
    }, 1000);
    
   
}
</script>
<template>
<div class="w-full h-full fixed top-0 left-0 bottom-0 right-0 z-[1001]  bg-pan-bottom opacity-98 scale-in-tr text-white/80"
:class="st.isClosed?['scale-out-tr']:[]">
    <div class="w-full h-full relative" style="--vh:80px;--vw:400px">
        <div class="absolute top-0 left-0">
                <canvas ref="clientCanvasRef" class="h-[var(--vh)]  w-[var(--vw)] " style="transform: rotate(90deg); transform-origin: 0 var(--vh); "/>
        </div>
        <div class="absolute top-0 right-0">
                <canvas ref="serverCanvasRef" class="h-[var(--vh)]  w-[var(--vw)] " style="transform: rotate(-90deg); transform-origin: var(--vw) var(--vh);  "/>
        </div> 

        <div class="flex flex-col justify-around items-center w-full h-full">
            <section>
                <!-- <aiTextSetting @close="loadConfig"  :msgInfo="$t('mj.rtsetting')" v-if="!st.apikey||!st.baseUrl"/> -->
                <div v-if="!st.apikey||!st.baseUrl">
                    <div v-html="$t('mj.rtsetting')" class="p-5 text-center"> </div>
                    <div class="text-center">
                        <NButton type="primary" @click="st.showSetting=true">{{ $t('setting.setting') }} </NButton> 
                    </div>
                </div>
                <an_main v-else-if="clientRef?.isConnected"/>
                <div v-else> {{ st.msg }}</div>
            </section>
            <section >
               <div  class="flex justify-center items-center space-x-4">
                    <div class="flex flex-col justify-center items-center cursor-pointer" @click="st.showSetting=true,disconnectConversation() " >
                        <div class=" bg-white rounded-full p-2"><SvgIcon icon="ri:settings-3-line" class="text-3xl text-orange-500/75"></SvgIcon></div>
                        <div class="pt-1">{{ $t('setting.setting') }}</div>
                    </div>
                    <div class="flex flex-col justify-center items-center cursor-pointer" @click="close()">
                        <!-- <div class="bg-orange-600 rounded-full p-2"><SvgIcon icon="tdesign:close" class="text-3xl"></SvgIcon></div> -->
                        <div class="bg-red-500 rounded-full p-2"><SvgIcon icon="majesticons:phone-hangup" class="text-3xl text-white"></SvgIcon></div>
                        <div class="pt-1">{{ $t('mj.mCanel') }}</div>
                    </div>
                    <div class="flex flex-col justify-center items-center cursor-pointer" @click="disconnectConversation()" v-if="st.isConnect">
                        <div class=" bg-white rounded-full p-2"><SvgIcon icon="ri:wechat-line" class="text-3xl text-orange-500/75"></SvgIcon></div>
                        <div class="pt-1">{{ $t('mj.mPause') }}</div>
                    </div>
                     <div class="flex flex-col justify-center items-center cursor-pointer" @click="go()" v-else>
                        <div class=" bg-white rounded-full p-2"><SvgIcon icon="ri:wechat-line" class="text-3xl text-orange-500/75"></SvgIcon></div>
                        <div class="pt-1">{{ $t('mj.mStart') }}</div>
                    </div>
                    
                </div>
                <div class="text-[12px] pt-5 text-center">
                    {{ st.msg }}
                    
                </div>
            </section>
        </div>
    </div>
</div>

<NModal v-model:show="st.showSetting" title="RealTime Setting" preset="card"  style="width: 95%; max-width: 640px">
    <wavSetting @close="st.showSetting=false, loadConfig()"  />
</NModal>
</template>

<style lang="css" >

@-webkit-keyframes bg-pan-bottom {
  0% {
    background-position: 50% 0%;
  }
  100% {
    background-position: 50% 100%;
  }
}
@keyframes bg-pan-bottom {
  0% {
    background-position: 50% 0%;
  }
  100% {
    background-position: 50% 100%;
  }
}
 .bg-pan-bottom {
	-webkit-animation: 10s ease 0s 1 normal both running bg-pan-bottom;;
	        animation: 10s ease 0s 1 normal both running bg-pan-bottom;;
    background-image: -webkit-gradient(linear, left bottom, left top, from(#cc6aa5), color-stop(#3e91cc), to(#2dcca7));
    background-image: linear-gradient(15deg, #cc6aa5, #3e91cc, #2dcca7);
    background-size: 100% 600%;
}


  
@keyframes scale-in-tr {
  0% {
    -webkit-transform: scale(0);
            transform: scale(0);
    -webkit-transform-origin: 100% 0%;
            transform-origin: 100% 0%;
    opacity:0.5;
    border-radius: 0% 0 0 100%;
  }
  80% {
    -webkit-transform: scale(1);
            transform: scale(1);
    -webkit-transform-origin: 100% 0%;
            transform-origin: 100% 0%;
    opacity: 1;
    border-radius: 0% 0 0 100%;
  }
  100% {
    -webkit-transform: scale(1);
            transform: scale(1);
    -webkit-transform-origin: 100% 0%;
            transform-origin: 100% 0%;
    opacity: 1;
    border-radius:0;
  }
}

.scale-in-tr {
    
	-webkit-animation: scale-in-tr 0.5s cubic-bezier(0.250, 0.460, 0.450, 0.940) both;
	        animation: scale-in-tr 0.5s cubic-bezier(0.250, 0.460, 0.450, 0.940) both;
}
 
@keyframes scale-out-tr {
  0% {
    -webkit-transform: scale(1);
            transform: scale(1);
    -webkit-transform-origin: 100% 0%;
            transform-origin: 100% 0%;
    opacity: 1;
  }
  20% {
    -webkit-transform: scale(1);
            transform: scale(1);
    -webkit-transform-origin: 100% 0%;
            transform-origin: 100% 0%;
    opacity: 1;
    border-radius: 0% 0 0 100%;
  }
  100% {
    -webkit-transform: scale(0);
            transform: scale(0);
    -webkit-transform-origin: 100% 0%;
            transform-origin: 100% 0%;
    opacity: 1;
     border-radius: 0% 0 0 100%;
  }
}

.scale-out-tr {
	-webkit-animation: scale-out-tr 0.5s cubic-bezier(0.550, 0.085, 0.680, 0.530) both;
	        animation: scale-out-tr 0.5s cubic-bezier(0.550, 0.085, 0.680, 0.530) both;
}
</style>