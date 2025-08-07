<script setup>
class VideoToMP3Converter {
  constructor() {
    this.selectedFile = null;
    this.audioContext = null;
    this.initializeElements();
    this.attachEventListeners();
  }

  initializeElements() {
    this.dropZone = document.getElementById("dropZone");
    this.fileInput = document.getElementById("fileInput");
    this.videoPreview = document.getElementById("videoPreview");
    this.videoElement = document.getElementById("videoElement");
    this.fileInfo = document.getElementById("fileInfo");
    this.fileName = document.getElementById("fileName");
    this.fileSize = document.getElementById("fileSize");
    this.videoDuration = document.getElementById("videoDuration");
    this.convertBtn = document.getElementById("convertBtn");
    //this.transcribeBtn = document.getElementById("transcribeBtn");
    this.progressContainer = document.getElementById("progressContainer");
    this.progressFill = document.getElementById("progressFill");
    this.progressText = document.getElementById("progressText");
    this.status = document.getElementById("status");
    /*this.transcriptionContainer = document.getElementById(
      "transcriptionContainer"
    );*/
    //this.transcriptionText = document.getElementById("transcriptionText");
    //this.languageSelect = document.getElementById("languageSelect");
    //this.copyTranscriptionBtn = document.getElementById("copyTranscriptionBtn");
    /*this.downloadTranscriptionBtn = document.getElementById(
      "downloadTranscriptionBtn"
    );*/
  }

  attachEventListeners() {
    // Drop zone events
    this.dropZone.addEventListener("click", () => this.fileInput.click());
    this.dropZone.addEventListener("dragover", (e) => this.handleDragOver(e));
    this.dropZone.addEventListener("dragleave", (e) => this.handleDragLeave(e));
    this.dropZone.addEventListener("drop", (e) => this.handleDrop(e));

    // File input change
    this.fileInput.addEventListener("change", (e) => this.handleFileSelect(e));

    // Convert button
    this.convertBtn.addEventListener("click", () => this.convertToMP3());

    // Transcribe button
    //this.transcribeBtn.addEventListener("click", () => this.transcribeAudio());

    // Transcription controls
    /*this.copyTranscriptionBtn.addEventListener("click", () =>
      this.copyTranscription()
    );
    this.downloadTranscriptionBtn.addEventListener("click", () =>
      this.downloadTranscription()
    );*/

    // Video loaded event
    this.videoElement.addEventListener("loadedmetadata", () =>
      this.updateVideoDuration()
    );
  }

  handleDragOver(e) {
    e.preventDefault();
    this.dropZone.classList.add("dragover");
  }

  handleDragLeave(e) {
    e.preventDefault();
    this.dropZone.classList.remove("dragover");
  }

  handleDrop(e) {
    e.preventDefault();
    this.dropZone.classList.remove("dragover");
    const files = e.dataTransfer.files;
    if (files.length > 0) {
      this.processFile(files[0]);
    }
  }

  handleFileSelect(e) {
    const file = e.target.files[0];
    if (file) {
      this.processFile(file);
    }
  }

  processFile(file) {
    // Validate file type
    if (!file.type.startsWith("video/")) {
      this.showStatus(
        "Por favor selecciona un archivo de video válido.",
        "error"
      );
      return;
    }

    this.selectedFile = file;
    this.displayFileInfo(file);
    this.showVideoPreview(file);
    this.convertBtn.disabled = false;
    //this.transcribeBtn.disabled = false;
    this.showStatus(
      "Archivo cargado correctamente. ¡Listo para convertir!",
      "success"
    );
  }

  displayFileInfo(file) {
    this.fileName.textContent = `📄 ${file.name}`;
    this.fileSize.textContent = `📏 ${this.formatFileSize(file.size)}`;
    this.fileInfo.style.display = "block";
  }

  showVideoPreview(file) {
    const url = URL.createObjectURL(file);
    this.videoElement.src = url;
    this.videoPreview.style.display = "block";
  }

  updateVideoDuration() {
    const duration = this.videoElement.duration;
    if (!isNaN(duration)) {
      this.videoDuration.textContent = `⏱️ ${this.formatDuration(duration)}`;
    }
  }

  async convertToMP3() {
    if (!this.selectedFile) return;

    try {
      this.showProgress(true);
      this.convertBtn.disabled = true;
      this.showStatus("Iniciando conversión...", "info");

      // Create a hidden video element
      const video = document.createElement("video");
      video.style.display = "none";
      video.muted = true;
      video.crossOrigin = "anonymous";
      document.body.appendChild(video);

      // Load the video file
      const videoURL = URL.createObjectURL(this.selectedFile);
      video.src = videoURL;

      // Wait for video to load
      await new Promise((resolve, reject) => {
        video.addEventListener("loadeddata", resolve);
        video.addEventListener("error", reject);
        video.load();
      });

      this.showStatus("Extrayendo audio del video...", "info");

      // Create audio context for processing
      const audioContext = new (window.AudioContext ||
        window.webkitAudioContext)();

      // Decode the entire video file to get audio data
      const arrayBuffer = await this.selectedFile.arrayBuffer();

      // We need to extract audio using a different approach
      // Create an OfflineAudioContext to process the entire audio
      const offlineContext = new OfflineAudioContext(
        2,
        audioContext.sampleRate * video.duration,
        audioContext.sampleRate
      );

      // Alternative approach: Use MediaRecorder first, then convert to MP3
      await this.extractAndConvertAudio(video, videoURL);
    } catch (error) {
      console.error("Error durante la conversión:", error);
      this.showStatus(`Error: ${error.message}`, "error");
      this.convertBtn.disabled = false;
      this.showProgress(false);
    }
  }

  async extractAndConvertAudio(video, videoURL) {
// oxlint-disable-next-line no-async-promise-executor
    return new Promise(async (resolve, reject) => {
      try {
        // Capture video stream
        let stream;
        if (video.captureStream) {
          stream = video.captureStream();
        } else if (video.mozCaptureStream) {
          stream = video.mozCaptureStream();
        } else {
          throw new Error("captureStream no está soportado en este navegador");
        }

        // Get only audio tracks
        const audioTracks = stream.getAudioTracks();
        if (audioTracks.length === 0) {
          throw new Error("No se encontraron pistas de audio en el video");
        }

        const audioStream = new MediaStream(audioTracks);

        // Setup MediaRecorder to capture raw audio data
        const mediaRecorder = new MediaRecorder(audioStream, {
          mimeType: "audio/webm;codecs=pcm",
        });

        const audioChunks = [];
        //let recordingStartTime = Date.now();

        mediaRecorder.ondataavailable = (event) => {
          if (event.data.size > 0) {
            audioChunks.push(event.data);
          }
        };

        mediaRecorder.onstop = async () => {
          try {
            this.showStatus("Convirtiendo a MP3...", "info");

            // Create blob from recorded data
            const audioBlob = new Blob(audioChunks, { type: "audio/webm" });

            // Convert to MP3
            await this.convertWebMToMP3(audioBlob);

            // Cleanup
            document.body.removeChild(video);
            URL.revokeObjectURL(videoURL);
            resolve();
          } catch (error) {
            reject(error);
          }
        };

        mediaRecorder.onerror = (event) => {
          reject(new Error("Error al grabar el audio: " + event.error));
        };

        // Start recording
        mediaRecorder.start(100);

        // Play video to generate stream
        video.currentTime = 0;
        await video.play();

        // Track progress
        const progressInterval = setInterval(() => {
          if (video.duration && video.currentTime) {
            const progress = (video.currentTime / video.duration) * 50; // 50% for extraction
            this.updateProgress(progress);
          }
        }, 100);

        // Stop recording when video ends
        video.addEventListener("ended", () => {
          clearInterval(progressInterval);
          setTimeout(() => {
            mediaRecorder.stop();
          }, 500);
        });

        // Handle video errors
        video.addEventListener("error", () => {
          clearInterval(progressInterval);
          mediaRecorder.stop();
          reject(new Error("Error al reproducir el video"));
        });
      } catch (error) {
        reject(error);
      }
    });
  }

  async convertWebMToMP3(audioBlob) {
    try {
      // Create audio context
      const audioContext = new (window.AudioContext ||
        window.webkitAudioContext)();

      // Convert blob to array buffer
      const arrayBuffer = await audioBlob.arrayBuffer();

      // Decode audio data
      const audioBuffer = await audioContext.decodeAudioData(arrayBuffer);

      // Get audio samples
      const leftChannel = audioBuffer.getChannelData(0);
      const rightChannel =
        audioBuffer.numberOfChannels > 1
          ? audioBuffer.getChannelData(1)
          : leftChannel;

      // Convert to 16-bit PCM
      const sampleRate = audioBuffer.sampleRate;
      const samples = this.interleave(leftChannel, rightChannel);

      // Initialize LAME encoder
      const mp3encoder = new lamejs.Mp3Encoder(2, sampleRate, 128); // 2 channels, sample rate, 128kbps

      const mp3Data = [];
      const sampleBlockSize = 1152; // LAME requirement

      // Encode in chunks
      for (let i = 0; i < samples.length; i += sampleBlockSize * 2) {
        const sampleChunk = samples.subarray(i, i + sampleBlockSize * 2);
        const left = [];
        const right = [];

        // Separate left and right channels
        for (let j = 0; j < sampleChunk.length; j += 2) {
          left.push(sampleChunk[j]);
          right.push(sampleChunk[j + 1] || 0);
        }

        const mp3buf = mp3encoder.encodeBuffer(left, right);
        if (mp3buf.length > 0) {
          mp3Data.push(mp3buf);
        }

        // Update progress (50% to 100%)
        const progress = 50 + (i / samples.length) * 50;
        this.updateProgress(progress);
      }

      // Flush remaining data
      const mp3buf = mp3encoder.flush();
      if (mp3buf.length > 0) {
        mp3Data.push(mp3buf);
      }

      // Create final MP3 blob
      const mp3Blob = new Blob(mp3Data, { type: "audio/mp3" });

      // Create download link
      this.createMP3DownloadLink(mp3Blob);

      this.updateProgress(100);
      this.showStatus("¡Conversión a MP3 completada exitosamente!", "success");
      this.convertBtn.disabled = false;
    } catch (error) {
      throw new Error("Error al convertir a MP3: " + error.message);
    }
  }

  // Helper function to interleave stereo samples
  interleave(leftChannel, rightChannel) {
    const length = leftChannel.length + rightChannel.length;
    const result = new Int16Array(length);

    let index = 0;
    for (let i = 0; i < leftChannel.length; i++) {
      // Convert from float32 to int16
      result[index++] = Math.max(-1, Math.min(1, leftChannel[i])) * 0x7fff;
      result[index++] = Math.max(-1, Math.min(1, rightChannel[i])) * 0x7fff;
    }

    return result;
  }

  // Speech Recognition Methods
  /*async transcribeAudio() {
                if (!this.selectedFile) return;

                // Check if Speech Recognition is supported
                if (!('webkitSpeechRecognition' in window) && !('SpeechRecognition' in window)) {
                    this.showStatus('❌ El reconocimiento de voz no está soportado en este navegador. Usa Chrome, Edge o Safari.', 'error');
                    return;
                }

                try {
                    this.showProgress(true);
                    this.transcribeBtn.disabled = true;
                    this.convertBtn.disabled = true;
                    this.showStatus('🎤 Iniciando transcripción de audio...', 'info');
                    this.transcriptionContainer.style.display = 'block';

                    // Clear previous transcription
                    this.transcriptionText.innerHTML = '<p class="transcription-placeholder">Procesando audio...</p>';

                    // Create audio element from video
                    const audioElement = document.createElement('audio');
                    audioElement.src = URL.createObjectURL(this.selectedFile);
                    audioElement.controls = false;
                    document.body.appendChild(audioElement);

                    // Initialize Speech Recognition
                    const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
                    const recognition = new SpeechRecognition();
                    
                    // Configure recognition
                    recognition.continuous = true;
                    recognition.interimResults = true;
                    recognition.lang = this.languageSelect.value;
                    recognition.maxAlternatives = 1;

                    let finalTranscript = '';
                    let interimTranscript = '';
                    let segmentCount = 0;
                    let isRecognitionActive = false;

                    recognition.onstart = () => {
                        isRecognitionActive = true;
                        this.showStatus('🎤 Escuchando audio... Habla claramente', 'info');
                        this.transcriptionText.innerHTML = '<div class="transcription-segment transcription-interim">🎤 Escuchando...</div>';
                    };

                    recognition.onresult = (event) => {
                        let currentInterim = '';
                        
                        for (let i = event.resultIndex; i < event.results.length; i++) {
                            const transcript = event.results[i][0].transcript;
                            const confidence = event.results[i][0].confidence;
                            
                            if (event.results[i].isFinal) {
                                finalTranscript += transcript + ' ';
                                this.addTranscriptionSegment(transcript, confidence, true, ++segmentCount);
                            } else {
                                currentInterim += transcript;
                            }
                        }
                        
                        if (currentInterim) {
                            this.updateInterimTranscription(currentInterim);
                        }
                    };

                    recognition.onerror = (event) => {
                        console.error('Speech recognition error:', event.error);
                        let errorMessage = 'Error en el reconocimiento de voz';
                        
                        switch (event.error) {
                            case 'no-speech':
                                errorMessage = 'No se detectó audio. Verifica que el video tenga sonido.';
                                break;
                            case 'audio-capture':
                                errorMessage = 'Error al capturar el audio del video.';
                                break;
                            case 'not-allowed':
                                errorMessage = 'Permisos de micrófono denegados. Habilita el micrófono y recarga.';
                                break;
                            case 'network':
                                errorMessage = 'Error de conexión durante la transcripción.';
                                break;
                        }
                        
                        this.showStatus('❌ ' + errorMessage, 'error');
                        this.resetTranscriptionButtons();
                    };

                    recognition.onend = () => {
                        isRecognitionActive = false;
                        
                        if (finalTranscript.trim()) {
                            this.showStatus('✅ Transcripción completada exitosamente', 'success');
                            this.updateProgress(100);
                            this.showProgress(false);
                        } else {
                            this.showStatus('⚠️ No se pudo transcribir audio. Verifica que el video tenga voz clara.', 'error');
                        }
                        
                        this.resetTranscriptionButtons();
                        document.body.removeChild(audioElement);
                    };

                    // Start audio playback and recognition simultaneously
                    await audioElement.play();
                    recognition.start();

                    // Monitor audio progress
                    const progressInterval = setInterval(() => {
                        if (audioElement.duration && audioElement.currentTime) {
                            const progress = (audioElement.currentTime / audioElement.duration) * 100;
                            this.updateProgress(progress);
                        }
                        
                        if (audioElement.ended && isRecognitionActive) {
                            recognition.stop();
                            clearInterval(progressInterval);
                        }
                    }, 500);

                } catch (error) {
                    console.error('Error during transcription:', error);
                    this.showStatus('❌ Error durante la transcripción: ' + error.message, 'error');
                    this.resetTranscriptionButtons();
                }
            }*/

  /*addTranscriptionSegment(text, confidence, isFinal, segmentNumber) {
                // Remove interim placeholder if exists
                const placeholder = this.transcriptionText.querySelector('.transcription-placeholder');
                if (placeholder) placeholder.remove();

                // Remove existing interim segments
                const interimSegments = this.transcriptionText.querySelectorAll('.transcription-interim');
                interimSegments.forEach(segment => segment.remove());

                const segment = document.createElement('div');
                segment.className = `transcription-segment ${isFinal ? 'transcription-final' : 'transcription-interim'}`;
                
                const timestamp = new Date().toLocaleTimeString();
                const confidencePercent = confidence ? Math.round(confidence * 100) : 0;
                
                segment.innerHTML = `
                    <div class="transcription-timestamp">
                        Segmento ${segmentNumber} - ${timestamp}
                        ${confidence ? `<span class="transcription-confidence">${confidencePercent}% confianza</span>` : ''}
                    </div>
                    <div>${text.trim()}</div>
                `;

                this.transcriptionText.appendChild(segment);
                this.transcriptionText.scrollTop = this.transcriptionText.scrollHeight;
            }*/

  /*updateInterimTranscription(text) {
                // Remove existing interim segments
                const interimSegments = this.transcriptionText.querySelectorAll('.transcription-interim');
                interimSegments.forEach(segment => segment.remove());

                if (text.trim()) {
                    const segment = document.createElement('div');
                    segment.className = 'transcription-segment transcription-interim';
                    segment.innerHTML = `
                        <div class="transcription-timestamp">Procesando...</div>
                        <div>${text.trim()}</div>
                    `;
                    this.transcriptionText.appendChild(segment);
                    this.transcriptionText.scrollTop = this.transcriptionText.scrollHeight;
                }
            }*/

  /*resetTranscriptionButtons() {
                this.transcribeBtn.disabled = false;
                this.convertBtn.disabled = false;
                this.showProgress(false);
            }*/

  /*copyTranscription() {
                const segments = this.transcriptionText.querySelectorAll('.transcription-final');
                let fullText = '';
                
                segments.forEach((segment, index) => {
                    const textContent = segment.querySelector('div:last-child').textContent;
                    fullText += textContent.trim() + ' ';
                });

                if (fullText.trim()) {
                    navigator.clipboard.writeText(fullText.trim()).then(() => {
                        this.showStatus('📋 Transcripción copiada al portapapeles', 'success');
                        setTimeout(() => this.showStatus('', ''), 2000);
                    }).catch(err => {
                        console.error('Error copying to clipboard:', err);
                        this.showStatus('❌ Error al copiar al portapapeles', 'error');
                    });
                } else {
                    this.showStatus('❌ No hay transcripción para copiar', 'error');
                }
            }*/

  /*downloadTranscription() {
                const segments = this.transcriptionText.querySelectorAll('.transcription-final');
                let fullText = `Transcripción de: ${this.selectedFile.name}\n`;
                fullText += `Fecha: ${new Date().toLocaleString()}\n`;
                fullText += `Idioma: ${this.languageSelect.options[this.languageSelect.selectedIndex].text}\n`;
                fullText += '='.repeat(50) + '\n\n';
                
                segments.forEach((segment, index) => {
                    const timestamp = segment.querySelector('.transcription-timestamp').textContent;
                    const textContent = segment.querySelector('div:last-child').textContent;
                    fullText += `[${timestamp}]\n${textContent.trim()}\n\n`;
                });

                if (segments.length > 0) {
                    const blob = new Blob([fullText], { type: 'text/plain;charset=utf-8' });
                    const url = URL.createObjectURL(blob);
                    const fileName = this.selectedFile.name.replace(/\.[^/.]+$/, '') + '_transcripcion.txt';
                    
                    const downloadLink = document.createElement('a');
                    downloadLink.href = url;
                    downloadLink.download = fileName;
                    downloadLink.click();
                    
                    URL.revokeObjectURL(url);
                    this.showStatus('💾 Transcripción descargada como archivo de texto', 'success');
                } else {
                    this.showStatus('❌ No hay transcripción para descargar', 'error');
                }
            }*/

  createMP3DownloadLink(mp3Blob) {
    const url = URL.createObjectURL(mp3Blob);
    const fileName = this.selectedFile.name.replace(/\.[^/.]+$/, "") + ".mp3";

    // Remove existing download link
    const existingLink = document.querySelector(".download-link");
    if (existingLink) {
      existingLink.remove();
    }

    const downloadLink = document.createElement("a");
    downloadLink.href = url;
    downloadLink.download = fileName;
    downloadLink.className = "download-link";
    downloadLink.innerHTML = "🎵 Descargar MP3";

    this.status.appendChild(downloadLink);
    this.showProgress(false);

    // Auto-trigger download
    downloadLink.click();

    // Show file size info
    const fileSizeMB = (mp3Blob.size / (1024 * 1024)).toFixed(2);
    const sizeInfo = document.createElement("p");
    sizeInfo.style.marginTop = "10px";
    sizeInfo.style.color = "#666";
    sizeInfo.textContent = `Archivo MP3 generado: ${fileSizeMB} MB`;
    this.status.appendChild(sizeInfo);
  }

  showProgress(show) {
    this.progressContainer.style.display = show ? "block" : "none";
    if (!show) {
      this.updateProgress(0);
    }
  }

  updateProgress(percentage) {
    this.progressFill.style.width = percentage + "%";
    this.progressText.textContent = Math.round(percentage) + "%";
  }

  showStatus(message, type = "") {
    this.status.textContent = message;
    this.status.className = "status " + type;
  }

  formatFileSize(bytes) {
    if (bytes === 0) return "0 Bytes";
    const k = 1024;
    const sizes = ["Bytes", "KB", "MB", "GB"];
    const i = Math.floor(Math.log(bytes) / Math.log(k));
    return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + " " + sizes[i];
  }

  formatDuration(seconds) {
    const hours = Math.floor(seconds / 3600);
    const minutes = Math.floor((seconds % 3600) / 60);
    const secs = Math.floor(seconds % 60);

    if (hours > 0) {
      return `${hours}:${minutes.toString().padStart(2, "0")}:${secs
        .toString()
        .padStart(2, "0")}`;
    }
    return `${minutes}:${secs.toString().padStart(2, "0")}`;
  }
}

// Initialize the converter when DOM is loaded
document.addEventListener("DOMContentLoaded", () => {
  new VideoToMP3Converter();
});
</script>

<template>
  <div class="container">
    <h1>🎵 Video a MP3</h1>
    <p class="subtitle">
      Extrae audio de videos y convierte a texto automáticamente
    </p>

    <div class="drop-zone" id="dropZone">
      <span class="drop-icon">📹</span>
      <h3>Arrastra tu video aquí</h3>
      <p>o haz clic para seleccionar un archivo MP4</p>
      <input
        type="file"
        class="file-input"
        id="fileInput"
        accept="video/mp4,video/*"
      />
    </div>

    <div class="video-preview" id="videoPreview">
      <video controls id="videoElement"></video>
    </div>

    <div class="file-info" id="fileInfo">
      <h4>Información del archivo:</h4>
      <p id="fileName"></p>
      <p id="fileSize"></p>
      <p id="videoDuration"></p>
    </div>

    <button class="btn" id="convertBtn" disabled>🎵 Convertir a MP3</button>
    <div class="progress-container" id="progressContainer">
      <p>Procesando audio...</p>
      <div class="progress-bar">
        <div class="progress-fill" id="progressFill"></div>
      </div>
      <p id="progressText">0%</p>
    </div>

    <div class="status" id="status"></div>
  </div>
</template>

<style scoped></style>
