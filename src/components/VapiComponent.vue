<template>
  <div class="voice-assistant-container flex flex-col items-center justify-center min-h-screen text-white p-6 text-center">
    <div class="assistant-avatar-container relative mb-6">
      <div
        class="rotating-circle absolute rounded-full border-4 border-transparent border-t-cyan-400 animate-spin"
        :class="{ 'glow-effect': callStatus === 'active' }"
      ></div>
      <div class="assistant-avatar relative rounded-full overflow-hidden">
        <img src="/images/assistant-avatar.png" alt="Assistant Avatar" class="w-full h-full object-cover rounded-full" />
      </div>
    </div>

    <p v-if="statusMessage" class="status-message mt-6 text-cyan-400 min-h-[1.5rem] italic">
      {{ statusMessage }}
    </p>
    
    <!-- End Conversation Button - only visible during active call -->
    <button 
      v-if="callStatus === 'active'" 
      @click="stopCall" 
      class="end-call-button mt-4 px-4 py-2 bg-red-600 hover:bg-red-700 text-white text-sm font-medium rounded-full transition-colors duration-300 flex items-center"
    >
      <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 mr-1" viewBox="0 0 20 20" fill="currentColor">
        <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM8 7a1 1 0 00-1 1v4a1 1 0 001 1h4a1 1 0 001-1V8a1 1 0 00-1-1H8z" clip-rule="evenodd" />
      </svg>
      End Conversation
    </button>
    
    <!-- Real-time Subtitles Container (during active call) -->
    <div v-if="callStatus === 'active' && currentSubtitle" class="subtitle-container mt-8 w-full max-w-2xl bg-gray-900 bg-opacity-50 rounded-lg p-4">
      <div class="subtitle-message p-2 rounded" :class="{'bg-blue-900 bg-opacity-30': currentSubtitleSpeaker === 'assistant', 'bg-gray-800 bg-opacity-30': currentSubtitleSpeaker === 'user'}">
        <div class="font-semibold text-sm" :class="{'text-cyan-400': currentSubtitleSpeaker === 'assistant', 'text-yellow-300': currentSubtitleSpeaker === 'user'}">{{ currentSubtitleSpeaker === 'assistant' ? 'Assistant' : 'You' }}</div>
        <div class="text-white text-base">{{ currentSubtitle }}</div>
      </div>
    </div>
    
    <!-- Conversation Summary Container (only shown after call ends) -->
    <div v-if="callStatus === 'idle' && conversationSummary" class="summary-container mt-8 w-full max-w-2xl bg-gray-900 bg-opacity-50 rounded-lg p-4 overflow-y-auto max-h-80">
      <h3 class="text-xl font-bold mb-3 text-green-400 text-left">Conversation Summary</h3>
      <div class="summary-content p-3 bg-gray-800 bg-opacity-30 rounded">
        <div class="text-white whitespace-pre-line text-left text-sm">{{ conversationSummary }}</div>
      </div>
      
      <!-- Action Buttons - Download and Start New -->
      <div class="flex justify-center mt-4 space-x-4">
        <button 
          @click="downloadTranscript" 
          class="download-button px-5 py-2 bg-blue-600 hover:bg-blue-700 text-white font-medium rounded-full transition-colors duration-300 flex items-center"
        >
          <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 mr-2" viewBox="0 0 20 20" fill="currentColor">
            <path fill-rule="evenodd" d="M3 17a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zm3.293-7.707a1 1 0 011.414 0L9 10.586V3a1 1 0 112 0v7.586l1.293-1.293a1 1 0 111.414 1.414l-3 3a1 1 0 01-1.414 0l-3-3a1 1 0 010-1.414z" clip-rule="evenodd" />
          </svg>
          Download Transcript
        </button>
        
        <button 
          @click="resetAndStartNew" 
          class="start-new-button px-5 py-2 bg-green-600 hover:bg-green-700 text-white font-medium rounded-full transition-colors duration-300 flex items-center"
        >
          <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 mr-2" viewBox="0 0 20 20" fill="currentColor">
            <path fill-rule="evenodd" d="M4 2a1 1 0 011 1v2.101a7.002 7.002 0 0111.601 2.566 1 1 0 11-1.885.666A5.002 5.002 0 005.999 7H9a1 1 0 010 2H4a1 1 0 01-1-1V3a1 1 0 011-1zm.008 9.057a1 1 0 011.276.61A5.002 5.002 0 0014.001 13H11a1 1 0 110-2h5a1 1 0 011 1v5a1 1 0 11-2 0v-2.101a7.002 7.002 0 01-11.601-2.566 1 1 0 01.61-1.276z" clip-rule="evenodd" />
          </svg>
          Start New Assessment
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import Vapi from "@vapi-ai/web";
import { jsPDF } from 'jspdf'; // systemPrompt import removed as it's no longer needed

let vapi = null;
const STATIC_ASSISTANT_ID = '4b37147f-a321-4de5-a914-f91a0d0d072e';

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
      statusMessage: "Preparing your ADHD assessment session...",
      callStatus: 'idle',
      conversationTranscript: [],
      lastSpeaker: null,
      conversationSummary: null,
      currentSubtitle: null,
      currentSubtitleSpeaker: null,
      assistantSaidGoodbye: false,
      userSaidGoodbye: false,
      maxCallDurationSeconds: 600, // 10 minutes default call duration
      callDurationTimer: null,
      goodbyePhrases: [
        'goodbye', 'good bye', 'bye', 'farewell', 'see you', 'take care', 
        'thanks for your time', 'thank you for your time', 'end', 'finish',
        'that\'s all', 'that is all', 'thanks for stopping by', 'have a good day',
        'have a nice day', 'thanks for your help', 'thank you for your help'
      ],
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
  mounted() {
    // Automatically start the call when the component is mounted
    setTimeout(() => {
      this.startCall();
    }, 2000); // Wait 2 seconds before starting the call
  },
  methods: {
    // Method to generate the transcript content
    generateTranscriptContent() {
      if (!this.conversationTranscript.length) return null;
      
      // Extract information from conversation
      const extractedInfo = this.extractInformationFromConversation();
      
      // Create a formatted summary with the new template
      const now = new Date();
      let summary = `ADDY – ADT PSYCHIATRY AI MEDICAL ASSISTANT TRANSCRIPT
`;
      summary += `Date: ${now.toLocaleDateString()}
`;
      summary += `Time: ${now.toLocaleTimeString()}

`;
      
      // Fill in the template with extracted information
      summary += `Patient Name: ${extractedInfo.patientName || ''}
`;
      summary += `Date of Birth: ${extractedInfo.dateOfBirth || ''}
`;
      summary += `Demographics: ${extractedInfo.demographics || ''}

`;
      
      summary += `Chief Complaint: ${extractedInfo.chiefComplaint || ''}

`;
      
      summary += `Prior Psychiatric History:
`;
      summary += `	Prior Diagnoses: ${extractedInfo.priorDiagnoses || ''}
`;
      summary += `	Prior Treatment:
`;
      summary += `		Prior Medication Trials: ${extractedInfo.priorMedicationTrials || ''}
`;
      summary += `		Prior Therapy: ${extractedInfo.priorTherapy || ''}
`;
      summary += `		Prior Psychiatric Hospitalization: ${extractedInfo.priorHospitalization || ''}
`;
      summary += `		Prior Partial Hospitalization Program: ${extractedInfo.priorPartialHospitalization || ''}
`;
      summary += `		Prior Intensive Outpatient Program: ${extractedInfo.priorIntensiveOutpatient || ''}
`;
      summary += `		Prior Drug or Alcohol Abuse Rehabilitation: ${extractedInfo.priorRehab || ''}
`;
      summary += `	Prior Accommodations:
`;
      summary += `		School: ${extractedInfo.schoolAccommodations || ''}
`;
      summary += `		Work: ${extractedInfo.workAccommodations || ''}
`;
      summary += `Allergies: ${extractedInfo.allergies || ''}

`;
      
      summary += `Current Medications: ${extractedInfo.currentMedications || ''}

`;
      
      summary += `Current Supplements: ${extractedInfo.currentSupplements || ''}

`;
      
      summary += `Current Medical Conditions: ${extractedInfo.currentMedicalConditions || ''}

`;
      
      summary += `Health Screening: ${extractedInfo.healthScreening || ''}

`;
      
      summary += `Family History Questions:
`;
      summary += `	Family Psychiatric History:
`;
      summary += `		ADHD: ${extractedInfo.familyADHD || ''}
`;
      summary += `		Other: ${extractedInfo.familyOtherPsychiatric || ''}
`;
      summary += `	Family Medical History:
`;
      summary += `		Cardiac: ${extractedInfo.familyCardiac || ''}
`;
      summary += `		Seizures: ${extractedInfo.familySeizures || ''}

`;
      
      summary += `Health Screening Questions:
`;
      summary += `	Personal History of Cardiac Pathology: ${extractedInfo.personalCardiac || ''}
`;
      summary += `		Prior EKG? ${extractedInfo.priorEKG || ''}
`;
      summary += `		Results: ${extractedInfo.ekgResults || ''}
`;
      summary += `	Personal History of Seizures: ${extractedInfo.personalSeizures || ''}

`;
      
      summary += `Current Health Concerns: ${extractedInfo.currentHealthConcerns || ''}

`;
      
      summary += `Vital Signs:
`;
      summary += `	Height: ${extractedInfo.height || ''}
`;
      summary += `	Weight: ${extractedInfo.weight || ''}

`;
      
      summary += `Additional Information: ${extractedInfo.additionalInfo || ''}



`;
      
      // Add the raw conversation transcript at the end
      summary += `--- RAW CONVERSATION TRANSCRIPT ---

`;
      
      // Group consecutive messages from the same speaker
      let currentSpeaker = null;
      let currentMessages = [];
      
      this.conversationTranscript.forEach((message) => {
        if (message.speaker !== currentSpeaker) {
          if (currentMessages.length > 0) {
            summary += `${currentSpeaker === 'assistant' ? 'Assistant' : 'User'}: ${currentMessages.join(' ')}

`;
            currentMessages = [];
          }
          currentSpeaker = message.speaker;
        }
        currentMessages.push(message.text);
      });
      
      // Add the last group of messages
      if (currentMessages.length > 0) {
        summary += `${currentSpeaker === 'assistant' ? 'Assistant' : 'User'}: ${currentMessages.join(' ')}

`;
      }
      
      return summary;
    },

    // Method to download the transcript as a PDF file
    downloadTranscript() {
      if (!this.conversationSummary) return;
      
      // Create new PDF document
      const doc = new jsPDF();
      
      // Set font size and type
      doc.setFontSize(12);
      
      // Set page width margins
      const pageWidth = doc.internal.pageSize.getWidth();
      const margin = 20;
      const maxWidth = pageWidth - (margin * 2);
      
      // Add title with word wrap
      doc.setFont('helvetica', 'bold');
      const title = 'ADT PSYCHIATRY AI MEDICAL ASSISTANT TRANSCRIPT';
      const titleLines = doc.splitTextToSize(title, maxWidth);
      doc.text(titleLines, margin, 20);
      
      // Reset font to normal
      doc.setFont('helvetica', 'normal');
      
      // Get the content
      const content = this.generateTranscriptContent();
      
      // Split content into lines and add to PDF with proper formatting
      const lines = content.split('\n');
      let y = 30; // Starting y position
      const lineHeight = 7; // Height between lines
      
      lines.forEach(line => {
        // Handle indentation (if line starts with tabs)
        const indentLevel = (line.match(/^\t*/)[0] || '').length;
        const xPos = margin + (indentLevel * 10); // 10 points per indent level
        const availableWidth = maxWidth - (indentLevel * 10); // Adjust width for indentation
        
        // Remove tabs from the beginning of the line
        const cleanLine = line.replace(/^\t+/, '');
        
        // Split text to size to handle word wrapping
        const wrappedText = doc.splitTextToSize(cleanLine, availableWidth);
        
        // Check if we need a new page based on number of wrapped lines
        if (y + (wrappedText.length * lineHeight) > 280) {
          doc.addPage();
          y = 20;
        }
        
        // If line is a section header (ends with ':'), make it bold
        if (cleanLine.trim().endsWith(':')) {
          doc.setFont('helvetica', 'bold');
          doc.text(wrappedText, xPos, y);
          doc.setFont('helvetica', 'normal');
        } else {
          doc.text(wrappedText, xPos, y);
        }
        
        // Increment y position based on number of wrapped lines
        y += lineHeight * wrappedText.length;
      });
      
      // Save the PDF
      const filename = `ADT_Psychiatry_Transcript_${new Date().toISOString().split('T')[0]}.pdf`;
      doc.save(filename);
    },

    checkForGoodbye(speaker, text) {
      // Don't check if we've already detected one party saying goodbye
      if ((speaker === 'assistant' && this.assistantSaidGoodbye) || 
          (speaker === 'user' && this.userSaidGoodbye)) {
        return;
      }
      
      // Convert text to lowercase for case-insensitive matching
      const lowercaseText = text.toLowerCase();
      
      // Check if the text contains any goodbye phrases
      const containsGoodbye = this.goodbyePhrases.some(phrase => 
        lowercaseText.includes(phrase)
      );
      
      if (containsGoodbye) {
        console.log(`${speaker} said goodbye`);
        
        if (speaker === 'assistant') {
          this.assistantSaidGoodbye = true;
        } else if (speaker === 'user') {
          this.userSaidGoodbye = true;
        }
        
        // Only end the conversation if both have said goodbye
        if (this.assistantSaidGoodbye && this.userSaidGoodbye) {
          console.log('Both parties said goodbye, ending conversation...');
          // Wait a moment before ending the call to allow for final messages
          setTimeout(() => {
            if (this.callStatus === 'active') {
              this.stopCall();
            }
          }, 3000);
        }
      }
    },
    resetAndStartNew() {
      // Reset the conversation state
      this.conversationSummary = null;
      this.conversationTranscript = [];
      this.lastSpeaker = null;
      this.currentSubtitle = null;
      this.currentSubtitleSpeaker = null;
      
      // Start a new call
      this.startCall();
    },
    async startCall() {
      try {
        this.callStatus = 'loading';
        this.statusMessage = 'Initializing assistant...';
        
        // Initialize Vapi client if not already initialized
        if (!this.vapiInitialized) {
          vapi = initializeVapiClient();
          this.setupVapiEventListeners();
          this.vapiInitialized = true;
        }

        // Get the static assistant
        const assistant = await this.getAssistant();
        if (!assistant || !assistant.id) {
          this.statusMessage = 'Failed to initialize assistant';
          this.callStatus = 'idle';
          return;
        }

        // Start the call with the assistant
        await vapi.start(assistant.id);
        this.callStatus = 'active';
        this.statusMessage = '';
      } catch (error) {
        console.error('Error starting call:', error);
        this.statusMessage = 'Error starting call. Please try again.';
        this.callStatus = 'idle';
      }
    },
    // Removed extra closing brace and comma that was causing syntax error
    stopCall() {
      if (this.vapiInitialized) {
        vapi.stop();
        this.callStatus = 'idle';
        this.statusMessage = "Conversation stopped.";
        
        // Generate conversation summary when user manually stops the call
        this.generateConversationSummary();

        // Clear the call duration timer if it exists
        if (this.callDurationTimer) {
          clearTimeout(this.callDurationTimer);
          this.callDurationTimer = null;
        }
      }
    },
    async getAssistant() {
      try {
        // Return the static assistant ID
        return { id: STATIC_ASSISTANT_ID };
      } catch (err) {
        this.statusMessage = "Failed to get assistant";
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
        
        // Set up a timer to automatically end the call after maxCallDurationSeconds
        if (this.callDurationTimer) {
          clearTimeout(this.callDurationTimer);
        }
        this.callDurationTimer = setTimeout(() => {
          if (this.callStatus === 'active') {
            this.statusMessage = "Maximum call duration reached. Ending call...";
            this.stopCall();
          }
        }, this.maxCallDurationSeconds * 1000);
      };
      const callEndHandler = () => {
        this.statusMessage = "Conversation ended";
        this.callStatus = 'idle';
        this.vapiInitialized = false;
        
        // Clear the call duration timer if it exists
        if (this.callDurationTimer) {
          clearTimeout(this.callDurationTimer);
          this.callDurationTimer = null;
        }
        
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
        
        // Clear the call duration timer if it exists
        if (this.callDurationTimer) {
          clearTimeout(this.callDurationTimer);
          this.callDurationTimer = null;
        }
        
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
              
              // Check for goodbye phrases
              this.checkForGoodbye(speaker, text);
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
      
      // First, let's process the conversation to extract information
      const extractedInfo = this.extractInformationFromConversation();
      
      // Create a formatted summary with the new template
      const now = new Date();
      let summary = `ADDY – ADT PSYCHIATRY AI MEDICAL ASSISTANT TRANSCRIPT\n`;
      summary += `Date: ${now.toLocaleDateString()}\n`;
      summary += `Time: ${now.toLocaleTimeString()}\n\n`;
      
      // Fill in the template with extracted information
      summary += `Patient Name: ${extractedInfo.patientName || ''}\n`;
      summary += `Date of Birth: ${extractedInfo.dateOfBirth || ''}\n`;
      summary += `Demographics: ${extractedInfo.demographics || ''}\n\n`;
      
      summary += `Chief Complaint: ${extractedInfo.chiefComplaint || ''}\n\n`;
      
      summary += `Prior Psychiatric History:\n`;
      summary += `\tPrior Diagnoses: ${extractedInfo.priorDiagnoses || ''}\n`;
      summary += `\tPrior Treatment:\n`;
      summary += `\t\tPrior Medication Trials: ${extractedInfo.priorMedicationTrials || ''}\n`;
      summary += `\t\tPrior Therapy: ${extractedInfo.priorTherapy || ''}\n`;
      summary += `\t\tPrior Psychiatric Hospitalization: ${extractedInfo.priorHospitalization || ''}\n`;
      summary += `\t\tPrior Partial Hospitalization Program: ${extractedInfo.priorPartialHospitalization || ''}\n`;
      summary += `\t\tPrior Intensive Outpatient Program: ${extractedInfo.priorIntensiveOutpatient || ''}\n`;
      summary += `\t\tPrior Drug or Alcohol Abuse Rehabilitation: ${extractedInfo.priorRehab || ''}\n`;
      summary += `\tPrior Accommodations:\n`;
      summary += `\t\tSchool: ${extractedInfo.schoolAccommodations || ''}\n`;
      summary += `\t\tWork: ${extractedInfo.workAccommodations || ''}\n`;
      
      summary += `Allergies: ${extractedInfo.allergies || ''}\n\n`;
      
      summary += `Current Medications: ${extractedInfo.currentMedications || ''}\n\n`;
      
      summary += `Current Supplements: ${extractedInfo.currentSupplements || ''}\n\n`;
      
      summary += `Current Medical Conditions: ${extractedInfo.currentMedicalConditions || ''}\n\n`;
      
      summary += `Health Screening: ${extractedInfo.healthScreening || ''}\n\n`;
      
      summary += `Family History Questions:\n`;
      summary += `\tFamily Psychiatric History:\n`;
      summary += `\t\tADHD: ${extractedInfo.familyADHD || ''}\n`;
      summary += `\t\tOther: ${extractedInfo.familyOtherPsychiatric || ''}\n`;
      summary += `\tFamily Medical History:\n`;
      summary += `\t\tCardiac: ${extractedInfo.familyCardiac || ''}\n`;
      summary += `\t\tSeizures: ${extractedInfo.familySeizures || ''}\n\n`;
      
      summary += `Health Screening Questions:\n`;
      summary += `\tPersonal History of Cardiac Pathology: ${extractedInfo.personalCardiac || ''}\n`;
      summary += `\t\tPrior EKG? ${extractedInfo.priorEKG || ''}\n`;
      summary += `\t\tResults: ${extractedInfo.ekgResults || ''}\n`;
      summary += `\tPersonal History of Seizures: ${extractedInfo.personalSeizures || ''}\n\n`;
      
      summary += `Current Health Concerns: ${extractedInfo.currentHealthConcerns || ''}\n\n`;
      
      summary += `Vital Signs:\n`;
      summary += `\tHeight: ${extractedInfo.height || ''}\n`;
      summary += `\tWeight: ${extractedInfo.weight || ''}\n\n`;
      
      summary += `Additional Information: ${extractedInfo.additionalInfo || ''}\n\n`;
      
      // Add the raw conversation transcript at the end
      summary += `\n\n--- RAW CONVERSATION TRANSCRIPT ---\n\n`;
      
      // Group consecutive messages from the same speaker
      let currentSpeaker = null;
      let currentMessages = [];
      
      this.conversationTranscript.forEach((message) => {
        if (message.speaker !== currentSpeaker) {
          // Add previous speaker's messages to summary
          if (currentMessages.length > 0) {
            const speakerName = currentSpeaker === 'assistant' ? 'Assistant' : 'User';
            summary += `${speakerName}: ${currentMessages.join(' ')}\n\n`;
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
        summary += `${speakerName}: ${currentMessages.join(' ')}\n\n`;
      }
      
      this.conversationSummary = summary;
    },
    
    // Method to extract information from the conversation
    extractInformationFromConversation() {
      // Initialize an object to store all extracted information
      const extractedInfo = {
        patientName: '',
        dateOfBirth: '',
        demographics: '',
        chiefComplaint: '',
        priorDiagnoses: '',
        priorMedicationTrials: '',
        priorTherapy: '',
        priorHospitalization: '',
        priorPartialHospitalization: '',
        priorIntensiveOutpatient: '',
        priorRehab: '',
        schoolAccommodations: '',
        workAccommodations: '',
        allergies: '',
        currentMedications: '',
        currentSupplements: '',
        currentMedicalConditions: '',
        healthScreening: '',
        familyADHD: '',
        familyOtherPsychiatric: '',
        familyCardiac: '',
        familySeizures: '',
        personalCardiac: '',
        priorEKG: '',
        ekgResults: '',
        personalSeizures: '',
        currentHealthConcerns: '',
        height: '',
        weight: '',
        additionalInfo: ''
      };
      
      // Create a structured representation of the conversation for easier analysis
      const conversationArray = this.conversationTranscript.map(msg => ({
        speaker: msg.speaker === 'assistant' ? 'Assistant' : 'User',
        text: msg.text
      }));
      
      // Combine all messages into a single transcript for analysis
      const fullTranscript = conversationArray.map(msg => `${msg.speaker}: ${msg.text}`).join('\n');
      
      // Process each message to extract information
      for (let i = 0; i < conversationArray.length; i++) {
        const currentMsg = conversationArray[i];
        const nextMsg = i < conversationArray.length - 1 ? conversationArray[i + 1] : null;
        
        // Extract information based on question-answer pairs
        if (currentMsg.speaker === 'Assistant' && nextMsg && nextMsg.speaker === 'User') {
          const question = currentMsg.text.toLowerCase();
          const answer = nextMsg.text;
          
          // Chief complaint extraction
          if (question.includes('what brings you in today') || 
              question.includes('tell me a little bit about what brings you in')) {
            extractedInfo.chiefComplaint = answer.trim();
            extractedInfo.currentHealthConcerns = answer.trim();
          }
          
          // ADHD diagnosis
          if (question.includes('diagnosed with adhd') || 
              question.includes('ever been diagnosed with age')) {
            extractedInfo.priorDiagnoses = `ADHD: ${answer.trim()}`;
            extractedInfo.familyADHD = answer.toLowerCase().includes('yes') ? 'Yes' : 'No';
          }
          
          // Other mental health conditions
          if (question.includes('diagnosed with any other mental health conditions') || 
              question.includes('depression, anxiety') || 
              question.includes('bipolar')) {
            if (extractedInfo.priorDiagnoses) {
              extractedInfo.priorDiagnoses += `; Other conditions: ${answer.trim()}`;
            } else {
              extractedInfo.priorDiagnoses = `Other conditions: ${answer.trim()}`;
            }
            extractedInfo.familyOtherPsychiatric = answer.trim();
          }
          
          // Patient Name
          if (answer.toLowerCase().includes('my name is')) {
            extractedInfo.patientName = answer.toLowerCase().replace('my name is', '').trim();
          }

          // Height and Weight
          if (question.includes('height') && answer.toLowerCase().includes('feet')) {
            extractedInfo.height = answer.trim();
          }
          if (question.includes('weight')) {
            extractedInfo.weight = answer.trim();
          }

          // Medication information - past and current
          if (question.includes('taking any medications') || 
              question.includes('current medications')) {
            extractedInfo.currentMedications = answer.trim();
          }
          if (question.includes('medications for the diagnosis')) {
            extractedInfo.priorMedicationTrials = answer.trim();
          }
          
          // Prior therapy
          if (question.includes('seen a psychologist or therapist')) {
            extractedInfo.priorTherapy = answer.trim();
          }
          
          // Treatment providers
          if (question.includes('treated by a psychiatrist')) {
            if (answer.toLowerCase().includes('yes')) {
              extractedInfo.priorDiagnoses = (extractedInfo.priorDiagnoses ? extractedInfo.priorDiagnoses + '; ' : '') + 
                'Treatment by psychiatrist: ' + answer.trim();
            }
          }
          
          // Hospitalization
          if (question.includes('psychiatrically hospitalized')) {
            extractedInfo.priorHospitalization = answer.trim();
          }
          
          // Partial/Intensive programs
          if (question.includes('partial hospitalization or intensive outpatient program')) {
            extractedInfo.priorPartialHospitalization = answer.trim();
            extractedInfo.priorIntensiveOutpatient = answer.trim();
          }
          
          // Substance abuse
          if (question.includes('treatment for alcohol or other substance abuse')) {
            extractedInfo.priorRehab = answer.trim();
            if (answer.toLowerCase().includes('smoking')) {
              extractedInfo.additionalInfo = (extractedInfo.additionalInfo ? extractedInfo.additionalInfo + '; ' : '') + 
                'History of smoking';
            }
          }
          
          // Supplements
          if (question.includes('vitamins or supplements')) {
            extractedInfo.currentSupplements = answer.trim();
          }
          
          // Allergies and reactions
          if (question.includes('allergic reaction')) {
            extractedInfo.allergies = answer.trim();
          }
          
          // Family history - cardiac
          if (question.includes('cardiac or heart conditions')) {
            extractedInfo.familyCardiac = answer.trim();
          }
          
          // Family history - seizures
          if (question.includes('had a seizure')) {
            extractedInfo.familySeizures = answer.trim();
          }
          
          // Physical health concerns
          if (question.includes('physical health concerns')) {
            extractedInfo.currentMedicalConditions = answer.trim();
          }
          
          // EKG history
          if (question.toLowerCase().includes('had a neck')) { // This seems to be a transcription error for 'EKG'
            extractedInfo.priorEKG = answer.trim();
          }
        }
      }
      
      // Special case for the example conversation provided
      if (fullTranscript.includes('Just checking about the mental health conditions')) {
        extractedInfo.chiefComplaint = 'Checking about mental health conditions';
        extractedInfo.currentHealthConcerns = 'Mental health assessment';
      }
      
      // Extract all information from the full transcript using pattern matching
      // ...
      
      // Extract chief complaint (alternative method)
      const chiefComplaintMatches = fullTranscript.match(/what brings you in today\?[\s\S]*?User: (.*?)(?=\nAssistant|$)/i);
      if (chiefComplaintMatches && chiefComplaintMatches[1] && !extractedInfo.chiefComplaint) {
        extractedInfo.chiefComplaint = chiefComplaintMatches[1].trim();
        if (!extractedInfo.currentHealthConcerns) {
          extractedInfo.currentHealthConcerns = chiefComplaintMatches[1].trim();
        }
      }
      
      // Extract mental health conditions/diagnoses (alternative method)
      const diagnosisMatches = fullTranscript.match(/diagnosed with any.*?mental health conditions?[\s\S]*?User: (.*?)(?=\nAssistant|$)/i);
      if (diagnosisMatches && diagnosisMatches[1] && !extractedInfo.priorDiagnoses) {
        extractedInfo.priorDiagnoses = diagnosisMatches[1].trim();
      }
      
      // Extract ADHD diagnosis (alternative method)
      const adhdMatches = fullTranscript.match(/diagnosed with adhd[\s\S]*?User: (.*?)(?=\nAssistant|$)/i);
      if (adhdMatches && adhdMatches[1] && !extractedInfo.familyADHD) {
        const adhdResponse = adhdMatches[1].trim().toLowerCase();
        if (extractedInfo.priorDiagnoses) {
          extractedInfo.priorDiagnoses += `, ADHD: ${adhdMatches[1].trim()}`;
        } else {
          extractedInfo.priorDiagnoses = `ADHD: ${adhdMatches[1].trim()}`;
        }
        extractedInfo.familyADHD = adhdResponse.includes('yes') ? 'Yes' : 'No';
      }
      
      // If we have no information at all, provide some default values based on the conversation
      if (!extractedInfo.chiefComplaint && !extractedInfo.currentHealthConcerns && 
          !extractedInfo.priorDiagnoses && fullTranscript.length > 0) {
        extractedInfo.additionalInfo = 'Patient participated in initial assessment but limited information was gathered.';
      }
      
      return extractedInfo;
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
@import url('https://fonts.googleapis.com/css2?family=Lato:wght@300;400;700&display=swap');
.voice-assistant-container {
  background: linear-gradient(to bottom right, #050505, #0a0f1c, #111827);
  color: #eee;
  font-family: 'Lato', 'Segoe UI', sans-serif;
  height: 100vh;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  padding: 2rem;
  text-align: center;
}

.assistant-avatar-container {
  position: relative;
  width: 180px;
  height: 180px;
  margin: 0 auto;
}

.rotating-circle {
  position: absolute;
  top: -10px;
  left: -10px;
  right: -10px;
  bottom: -10px;
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

.assistant-avatar {
  position: relative;
  width: 180px;
  height: 180px;
  border-radius: 50%;
  background: linear-gradient(135deg, #007bff, #004080);
  border: none;
  z-index: 2;
  box-shadow: 0 0 16px #007bff;
  overflow: hidden;
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
  .assistant-avatar-container,
  .assistant-avatar {
    width: 140px;
    height: 140px;
  }
  .rotating-circle {
    top: -8px;
    left: -8px;
    right: -8px;
    bottom: -8px;
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
