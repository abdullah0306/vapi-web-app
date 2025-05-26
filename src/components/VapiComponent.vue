<template>
  <div class="voice-assistant-container flex flex-col items-center justify-center min-h-screen text-white p-6 text-center">
    <div class="button-wrapper relative w-44 h-44 cursor-pointer" @click="toggleCall">
      <div
        class="rotating-circle absolute -top-4 -left-4 w-52 h-52 rounded-full border-4 border-transparent border-t-cyan-400 animate-spin"
        :class="{ 'glow-effect': callStatus === 'active' }"
      ></div>
      <button
        :class="buttonClasses"
        class="talk-button relative w-44 h-44 rounded-full font-semibold text-lg shadow-lg focus:outline-none transition-colors duration-300 overflow-hidden flex items-center justify-center"
      >
        <img src="/images/assistant-avatar.jpg.jpg" alt="Assistant Avatar" class="w-full h-full object-cover rounded-full" />
        <div class="hover-text absolute inset-0 flex items-center justify-center text-white font-bold text-sm opacity-0 hover:opacity-100 bg-black bg-opacity-30 transition-opacity duration-300">
          Click to talk
        </div>
      </button>
    </div>

    <p v-if="statusMessage" class="status-message mt-6 text-cyan-400 min-h-[1.5rem] italic">
      {{ statusMessage }}
    </p>
    
    <!-- Real-time Subtitles Container (during active call) -->
    <div v-if="callStatus === 'active' && currentSubtitle" class="subtitle-container mt-8 w-full max-w-2xl bg-gray-900 bg-opacity-50 rounded-lg p-4 text-center">
      <div class="subtitle-message p-2 rounded" :class="{'bg-blue-900 bg-opacity-30': currentSubtitleSpeaker === 'assistant', 'bg-gray-800 bg-opacity-30': currentSubtitleSpeaker === 'user'}">
        <div class="font-semibold" :class="{'text-cyan-400': currentSubtitleSpeaker === 'assistant', 'text-yellow-300': currentSubtitleSpeaker === 'user'}">{{ currentSubtitleSpeaker === 'assistant' ? 'Assistant' : 'You' }}</div>
        <div class="text-white text-lg">{{ currentSubtitle }}</div>
      </div>
    </div>
    
    <!-- Conversation Summary Container (only shown after call ends) -->
    <div v-if="callStatus === 'idle' && conversationSummary" class="summary-container mt-8 w-full max-w-2xl bg-gray-900 bg-opacity-50 rounded-lg p-4 overflow-y-auto max-h-80">
      <h3 class="text-xl font-bold mb-3 text-green-400">Conversation Summary</h3>
      <div class="summary-content p-3 bg-gray-800 bg-opacity-30 rounded">
        <div class="text-white whitespace-pre-line">{{ conversationSummary }}</div>
      </div>
    </div>
  </div>
</template>

<script>
import Vapi from "@vapi-ai/web";
import systemPrompt from '../assets/systemPrompt.js';

let vapi = null;

const initializeVapiClient = () => {
  if (!process.env.VUE_APP_VAPI_PUBLIC_KEY) {
    throw new Error('VUE_APP_VAPI_PUBLIC_KEY is not set');
  }
  vapi = new Vapi(process.env.VUE_APP_VAPI_PUBLIC_KEY);
  vapi.baseURL = 'https://api.vapi.ai';
  vapi.wsURL = 'wss://api.vapi.ai';
  vapi.debug = true;
  return vapi;
};

export default {
  name: "VapiComponent",
  data() {
    return {
      vapiInitialized: false,
      statusMessage: "",
      callStatus: 'idle',
      conversationTranscript: [],
      lastSpeaker: null,
      conversationSummary: null,
      currentSubtitle: null,
      currentSubtitleSpeaker: null,
    };
  },
  computed: {
    buttonClasses() {
      const base = 'bg-gradient-to-br from-cyan-500 to-blue-600 text-white shadow-cyan-700 hover:from-cyan-600 hover:to-blue-700';
      const loading = 'bg-gradient-to-br from-orange-500 to-yellow-400 text-white shadow-orange-600 hover:from-orange-600 hover:to-yellow-500';
      const active = 'bg-gradient-to-br from-red-600 to-red-700 text-white shadow-red-800 hover:from-red-700 hover:to-red-800';
      switch(this.callStatus) {
        case 'loading': return loading;
        case 'active': return active;
        default: return base;
      }
    }
  },
  methods: {
    toggleCall() {
      if (this.callStatus === 'idle' || this.callStatus === 'loading') {
        this.startCall();
      } else if (this.callStatus === 'active') {
        this.stopCall();
      }
    },
    async startCall() {
      this.callStatus = 'loading';
      this.statusMessage = "Initializing conversation...";
      try {
        await navigator.mediaDevices.getUserMedia({ audio: true });
        vapi = initializeVapiClient();
        this.vapiInitialized = true;

        this.setupVapiEventListeners();
        this.statusMessage = "Setting up assistant...";
        const assistantResponse = await this.createOrUpdateAssistant();

        if (assistantResponse && assistantResponse.id) {
          this.statusMessage = "Starting conversation...";
          await vapi.start(assistantResponse.id);
          this.callStatus = 'active';
          this.statusMessage = "Conversation in progress...";
        } else {
          throw new Error('Failed to get valid assistant response');
        }
      } catch (error) {
        this.statusMessage = `Error: ${error.message || 'Failed to start call'}`;
        this.callStatus = 'idle';
        this.vapiInitialized = false;
        vapi = null;
      }
    },
    stopCall() {
      if (this.vapiInitialized) {
        vapi.stop();
        this.callStatus = 'idle';
        this.statusMessage = "Conversation stopped.";
        
        // Generate conversation summary when user manually stops the call
        this.generateConversationSummary();
      }
    },
    async createOrUpdateAssistant() {
      try {
        const response = await fetch(process.env.VUE_APP_POST_URL, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            backgroundSound: "off",
            clientMessages: ["transcript","hang","speech-update","metadata","conversation-update"],
            dialKeypadFunctionEnabled: false,
            endCallFunctionEnabled: false,
            endCallMessage: "Bye Bye!",
            firstMessage: "Let's talk about work. How may I assist?",
            hipaaEnabled: false,
            llmRequestDelaySeconds: 0.1,
            maxDurationSeconds: 600,
            metadata: {},
            model: {
              maxTokens: 200,
              model: "gpt-3.5-turbo",
              provider: "openai",
              temperature: 0,
              systemPrompt: systemPrompt,
            },
            name: "AssistantPy2",
            numWordsToInterruptAssistant: 2,
            recordingEnabled: true,
            responseDelaySeconds: 0.4,
            serverMessages: ["end-of-call-report"],
            silenceTimeoutSeconds: 20,
            transcriber: {
              language: "en-US",
              model: "nova-2",
              provider: "deepgram",
            },
            voice: {
              provider: "deepgram",
              voiceId: "luna",
            },
            voicemailDetectionEnabled: false,
          }),
        });
        if (!response.ok) {
          this.statusMessage = "Error creating/updating assistant";
          return null;
        }
        return await response.json();
      } catch (err) {
        this.statusMessage = "Failed to create/update assistant";
        return null;
      }
    },
    setupVapiEventListeners() {
      if (!vapi) return;

      const callStartHandler = () => {
        this.statusMessage = "Conversation started";
        this.callStatus = 'active';
        // Clear previous transcript and subtitles when starting a new call
        this.conversationTranscript = [];
        this.lastSpeaker = null;
        this.currentSubtitle = null;
        this.currentSubtitleSpeaker = null;
        this.conversationSummary = null;
      };
      const callEndHandler = () => {
        this.statusMessage = "Conversation ended";
        this.callStatus = 'idle';
        this.vapiInitialized = false;
        
        // Generate conversation summary when call ends
        this.generateConversationSummary();

        vapi.off("call-start", callStartHandler);
        vapi.off("call-end", callEndHandler);
        vapi.off("error", errorHandler);
        vapi.off("message", messageHandler);
      };
      const errorHandler = (error) => {
        this.statusMessage = `Error: ${error.message || 'Unknown error occurred'}`;
        this.callStatus = 'idle';
        this.vapiInitialized = false;

        vapi.off("call-start", callStartHandler);
        vapi.off("call-end", callEndHandler);
        vapi.off("error", errorHandler);
        vapi.off("message", messageHandler);
      };
      
      const messageHandler = (msg) => {
        // Log all message types for debugging
        console.log('Vapi message received:', msg);
        
        // Handle transcript messages
        if (msg.type === "transcript") {
          // Process transcript message
          const speaker = msg.role || 'unknown';
          const text = msg.transcript || '';
          const isPartial = msg.transcriptType === 'partial';
          
          if (text.trim()) {
            // Update current subtitle for display
            this.currentSubtitle = text;
            this.currentSubtitleSpeaker = speaker;
            
            // Only add to transcript if it's a final transcript
            // (to avoid flooding the transcript with partial results)
            if (!isPartial) {
              // Store in transcript for summary generation later
              this.conversationTranscript.push({
                speaker: speaker,
                text: text
              });
              this.lastSpeaker = speaker;
            }
          }
        }
        
        // Handle subtitle messages if they have a different format
        if (msg.type === "subtitle") {
          const speaker = msg.role || msg.speaker || 'unknown';
          const text = msg.text || msg.transcript || '';
          
          if (text.trim()) {
            // Update current subtitle for display
            this.currentSubtitle = text;
            this.currentSubtitleSpeaker = speaker;
            
            // Store in transcript for summary generation later
            this.conversationTranscript.push({
              speaker: speaker,
              text: text
            });
            this.lastSpeaker = speaker;
          }
        }
      };

      vapi.on("call-start", callStartHandler);
      vapi.on("call-end", callEndHandler);
      vapi.on("error", errorHandler);
      vapi.on("message", messageHandler);
    },
    
    // Method to generate a summary of the conversation
    generateConversationSummary() {
      if (this.conversationTranscript.length === 0) {
        this.conversationSummary = null;
        return;
      }
      
      // Create a formatted summary
      let summary = "Conversation Summary:\n\n";
      
      // Group consecutive messages from the same speaker
      let currentSpeaker = null;
      let currentMessages = [];
      let summaryParts = [];
      
      this.conversationTranscript.forEach((message) => {
        if (message.speaker !== currentSpeaker) {
          // Add previous speaker's messages to summary
          if (currentMessages.length > 0) {
            const speakerName = currentSpeaker === 'assistant' ? 'Assistant' : 'User';
            summaryParts.push(`${speakerName}: ${currentMessages.join(' ')}`);
          }
          
          // Start new speaker
          currentSpeaker = message.speaker;
          currentMessages = [message.text];
        } else {
          // Continue with current speaker
          currentMessages.push(message.text);
        }
      });
      
      // Add the last speaker's messages
      if (currentMessages.length > 0) {
        const speakerName = currentSpeaker === 'assistant' ? 'Assistant' : 'User';
        summaryParts.push(`${speakerName}: ${currentMessages.join(' ')}`);
      }
      
      // Join all parts with line breaks
      summary += summaryParts.join('\n\n');
      
      // Add timestamp
      const now = new Date();
      summary += `\n\nCall ended: ${now.toLocaleString()}`;
      
      this.conversationSummary = summary;
    }
  },
  beforeUnmount() {
    if (this.vapiInitialized) {
      vapi.stop();
      // Make sure to remove all event listeners
      vapi.off("call-start");
      vapi.off("call-end");
      vapi.off("error");
      vapi.off("message");
    }
  }
};
</script>

<style scoped>
.voice-assistant-container {
  background: linear-gradient(to bottom right, #050505, #0a0f1c, #111827);
  color: #eee;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  height: 100vh;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  padding: 2rem;
  text-align: center;
}

.button-wrapper {
  position: relative;
  width: 180px;
  height: 180px;
  cursor: pointer;
  margin: 0 auto;
}

.rotating-circle {
  position: absolute;
  top: -10px;
  left: -10px;
  width: 200px;
  height: 200px;
  border: 5px solid transparent;
  border-top-color: #0ff;
  border-radius: 50%;
  animation: spin 3s linear infinite;
  z-index: 1;
}

.glow-effect {
  box-shadow: 0 0 15px #00e6ff, 0 0 25px #00cfff, 0 0 40px #00bfff;
  filter: drop-shadow(0 0 15px #00cfff);
}

.talk-button {
  position: relative;
  width: 180px;
  height: 180px;
  border-radius: 50%;
  background: linear-gradient(135deg, #007bff, #004080);
  border: none;
  font-size: 1.2rem;
  font-weight: 700;
  color: white;
  z-index: 2;
  box-shadow: 0 0 16px #007bff;
  transition: background 0.3s ease, box-shadow 0.3s ease;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
}

.hover-text {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.3);
  opacity: 0;
  transition: opacity 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
}

.talk-button:hover .hover-text {
  opacity: 1;
}

.status-message {
  margin-top: 1.5rem;
  font-size: 1.1rem;
  color: #0ff;
  min-height: 1.5rem;
  font-style: italic;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

@media (max-width: 600px) {
  .button-wrapper,
  .talk-button {
    width: 140px;
    height: 140px;
  }
  .rotating-circle {
    width: 160px;
    height: 160px;
    top: -8px;
    left: -8px;
  }
}
.transcript-container {
  background: rgba(10, 15, 28, 0.7);
  border: 1px solid rgba(0, 200, 255, 0.3);
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 200, 255, 0.1);
  transition: all 0.3s ease;
  backdrop-filter: blur(5px);
  scrollbar-width: thin;
  scrollbar-color: rgba(0, 200, 255, 0.5) rgba(10, 15, 28, 0.3);
}

.transcript-container::-webkit-scrollbar {
  width: 6px;
}

.transcript-container::-webkit-scrollbar-track {
  background: rgba(10, 15, 28, 0.3);
  border-radius: 3px;
}

.transcript-container::-webkit-scrollbar-thumb {
  background-color: rgba(0, 200, 255, 0.5);
  border-radius: 3px;
}

.transcript-message {
  transition: background-color 0.3s ease;
  border-left: 3px solid transparent;
}

.transcript-message:hover {
  background-color: rgba(255, 255, 255, 0.1) !important;
}
</style>
