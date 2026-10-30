# 📱 OmniUpscaler Mobile User Guide

This guide walks you through running **OmniUpscaler** entirely from your smartphone (iPhone / iPad or Android device). No desktop computer required.

---

## 🧭 Prerequisites
* A free **Google Account** (to access Google Colab).
* **Mobile Browser**:
  * iOS: Safari, Chrome, or Brave.
  * Android: Chrome, Firefox, or Samsung Internet.
* A video in your **Photo Library / Camera Roll / Files**.

---

## 📲 Step-by-Step Mobile Walkthrough

### 1. Launch the Notebook
1. Open the [OmniUpscaler Colab Notebook](https://colab.research.google.com/github/NiL4gh/omni-upscaler/blob/main/enhanced_video_upscaler.ipynb) on your mobile browser.
2. Sign in to your Google Account if prompted.

### 2. Verify GPU Runtime
1. In the top navigation bar, tap the **Runtime** menu (or the three dots `...`).
2. Select **Change runtime type**.
3. Ensure **T4 GPU** is selected (this provides free NVIDIA cloud hardware).
4. Tap **Save**.

### 3. Start the Web Studio
1. Locate the main code cell titled **`🎬 Launch AI Video Upscaler Pro Max Studio`**.
2. Tap the circular **Play button ▶** on the left of the title.
3. Colab will download the weights and configure the pipeline (~20–30 seconds).

### 4. Open the Gradio Web Studio
1. Scroll down to the cell's output box.
2. Look for the line:
   ```
   * Running on public URL: https://[random-string].gradio.live
   ```
3. **Tap this link** to open the mobile-optimized web studio in a new tab.

### 5. Upload & Upscale
1. In the **Source Media** card, tap the video upload zone.
2. Choose **Photo Library / Camera Roll** (or **Choose File**).
3. The 4-metric inspector chips will display your video resolution, framerate, and audio track status.
4. Select your preset (e.g. *Cinematic & Real-World* or *Anime & 2D*).
5. Tap the vibrant **✨ Launch AI Upscaler** button.

### 6. Save to Camera Roll
1. When processing completes, your 4K upscaled video will appear in the output player.
2. Tap **📥 Download Enhanced Master Video (.mp4)**.
3. On iOS: Tap **Download**, then open the Files app or tap the download icon in Safari ➔ tap the video ➔ tap the Share icon ➔ **Save Video** to move it to your Photos app.
4. On Android: The file will automatically save to your **Downloads** folder and show up in your Gallery/Google Photos.

---

## ⚡ Mobile Pro-Tips & Best Practices

* **Keep the Colab Tab Active**: Mobile operating systems (especially iOS) aggressively suspend background browser tabs. Keep the Colab tab open or switch between tabs occasionally so the cloud session doesn't go to sleep.
* **Auto-Sync to Google Drive**: If you check **Auto-Save to Google Drive**, your finished videos will automatically appear in your Google Drive under `MyDrive/AI_Upscaled`, so you can easily access them from any device later.
* **5-Second Fast Preview**: Use the preview toggle first to test the output quality before committing to upscaling an entire long video.
