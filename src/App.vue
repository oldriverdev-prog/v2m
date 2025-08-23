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
        "Please select a valid video file.",
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
      "File uploaded successfully. Ready to convert!",
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
      this.showStatus("Starting conversion...", "info");

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

      this.showStatus("Extracting audio from video...", "info");

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
      console.error("Error during conversion:", error);
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
          throw new Error("captureStream is not supported in this browser");
        }

        // Get only audio tracks
        const audioTracks = stream.getAudioTracks();
        if (audioTracks.length === 0) {
          throw new Error("No audio tracks were found in the video");
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
            this.showStatus("Converting to MP3...", "info");

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
          reject(new Error("Error recording audio: " + event.error));
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
          reject(new Error("Error playing video"));
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
      this.showStatus("MP3 conversion completed successfully!", "success");
      this.convertBtn.disabled = false;
    } catch (error) {
      throw new Error("Error converting to MP3: " + error.message);
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
    downloadLink.innerHTML = "🎵 Download MP3";

    this.status.appendChild(downloadLink);
    this.showProgress(false);

    // Auto-trigger download
    downloadLink.click();

    // Show file size info
    const fileSizeMB = (mp3Blob.size / (1024 * 1024)).toFixed(2);
    const sizeInfo = document.createElement("p");
    sizeInfo.style.marginTop = "10px";
    sizeInfo.style.color = "#666";
    sizeInfo.textContent = `File MP3 generated: ${fileSizeMB} MB`;
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
      Extract audio from your videos.
    </p>

    <div class="drop-zone" id="dropZone">
      <span class="drop-icon">📹</span>
      <h3>Drag your video here</h3>
      <p>or click to select an MP4 file</p>
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
      <h4>File information:</h4>
      <p id="fileName"></p>
      <p id="fileSize"></p>
      <p id="videoDuration"></p>
    </div>

    <button class="btn" id="convertBtn" disabled>🎵 Convert to MP3</button>
    <div class="progress-container" id="progressContainer">
      <p>Processing audio...</p>
      <div class="progress-bar">
        <div class="progress-fill" id="progressFill"></div>
      </div>
      <p id="progressText">0%</p>
    </div>

    <div class="status" id="status"></div>
  </div>
</template>

<style scoped></style>
