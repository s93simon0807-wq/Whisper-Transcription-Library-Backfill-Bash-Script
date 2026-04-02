Automated Local AI Subtitle Generator (Unraid / Docker)
A robust, fully automated Bash script designed to run on Unraid (or any Linux server) that scans your local media library and generates perfectly timed subtitles (.srt) using a local Whisper AI container.

Instead of relying on third-party subtitle downloaders that often suffer from sync issues, this script uses FFmpeg to extract the actual audio track from your video files and passes it to an OpenAI Whisper API container for high-fidelity, locally processed transcription.

✨ Key Features
Smart Backfilling: The script remembers where it left off. It automatically skips videos that already have subtitles and drops a .failed marker on corrupted video files so it doesn't waste time retrying them on the next run.

VRAM/Resource Management: It automatically starts the Whisper AI container when a job begins and kills it when finished, freeing up your GPU's Video RAM for other services (like local LLMs or Plex transcoding).

Scheduled Time Windows: Configure the script to only run during off-hours (e.g., 3:00 AM to 7:00 AM) to ensure it never bogs down your server while people are actively streaming media.

Profanity Filter Bypass: Uses a custom INITIAL_PROMPT to prime the Whisper model, forcing it to accurately transcribe profanity rather than censoring words with asterisks.

Custom Watermarking: Injects a custom, cinematic watermark (e.g., "Brought to you by Fine ChAInesium") into the first 5 seconds of the subtitle file. It uses the {\an8} alignment tag to pin the watermark to the top of the screen so it never overlaps with actual dialogue.

File Size Limits: Allows you to set a maximum file size in MB, bypassing massive 4K remuxes if you only want to process standard web-rips.

Detailed Analytics: Provides a clean summary at the end of each run detailing exactly how many files were scanned, processed, or skipped.

🛠️ Prerequisites
To run this script, you will need two Docker containers running on your server:

whisper-asr-webservice: The API endpoint that runs the Whisper model.

FFmpeg (with NVIDIA support): Used to rapidly extract the audio track from your video files before sending it to Whisper.

⚙️ Quick Configuration
Open the script and modify the CONFIGURATION section to match your specific setup:

Bash
MOVIE_DIR="/path/to/your/Movies/"
TV_DIR="/path/to/your/TV Shows/"
WHISPER_API_URL="http://YOUR_WHISPER_IP:9000/asr"
WHISPER_CONTAINER_NAME="your-whisper-container"
FFMPEG_CONTAINER_NAME="your-ffmpeg-container"
🚀 Usage
You can run this script manually from your terminal, or schedule it to run automatically using Unraid's User Scripts plugin via a basic cron schedule (e.g., 0 3 * * * to run daily at 3 AM).
