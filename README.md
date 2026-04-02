#!/bin/bash
# ==============================================================================
# Library_Transcription_Backfill_FFmPeg_Start_Kill
# Version: 3.8.1 (Public Release)
#
# Description:
#   Automated transcription script using local FFmpeg and Whisper AI containers.
#   Scans media directories, extracts audio, generates subtitles, and supports
#   custom watermark injection.
# ==============================================================================

# --- CONFIGURATION - PLEASE EDIT THESE VALUES ---

# 1. Set the paths to your media folders. Make sure they end with a slash /
MOVIE_DIR="/path/to/your/media/Movies/"
TV_DIR="/path/to/your/media/TV Shows/"

# 2. Whisper container's API endpoint (Replace with your Whisper server IP)
WHISPER_API_URL="http://YOUR_WHISPER_IP:9000/asr"

# 3. Docker Container names
WHISPER_CONTAINER_NAME="your-whisper-container"
FFMPEG_CONTAINER_NAME="your-ffmpeg-container"

# 4. Output format for subtitles
OUTPUT_FORMAT="srt"

# 5. Force language tag for filename (leave blank to use 'en' as default)
TRANSCRIPTION_LANGUAGE="en"

# 6. Time window (24h format) - Set when the script is allowed to run
export START_HOUR=15
export END_HOUR=7

# 7. Log file path
LOG_FILE="/path/to/your/appdata/whisper-script/transcription.log"

# 8. Maximum file size in MB (Skip files larger than this to save processing time)
MAX_FILE_SIZE_MB=54000

# 9. Initial Prompt (Formatting and punctuation guide for the Whisper model)
INITIAL_PROMPT="Hello! Welcome to the show. Dr. Smith, Mr. Jones... fuck, shit, damn, okay, alright."

# 10. Custom Watermark (Added to the very beginning of the subtitle file)
# Note: The {\an8} tag pins the watermark to the top of the screen to prevent overlapping dialogue.
ENABLE_WATERMARK=true
WATERMARK_TEXT="{\\an8}<i>Brought to you by Fine ChAInesium</i>"

# --- INTERNAL CONFIG ---
# Using the mounted /tmp directory for clean audio extraction
TEMP_AUDIO_FILE="/tmp/transcribe_temp_audio.wav"
mkdir -p "$(dirname "$LOG_FILE")"

# --- GLOBAL COUNTERS ---
export TOTAL_SCANNED=0
export ACTUALLY_PROCESSED=0
export SKIPPED_EXISTING=0
export SKIPPED_FAILED=0
export SKIPPED_SIZE=0

# --- FUNCTIONS ---
log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

cleanup() {
    log "Performing cleanup..."
    if docker ps --filter "name=^/${WHISPER_CONTAINER_NAME}$" --filter "status=running" | grep -qw "${WHISPER_CONTAINER_NAME}"; then
        log "Stopping Whisper container: ${WHISPER_CONTAINER_NAME}..."
        docker stop "${WHISPER_CONTAINER_NAME}"
        log "Whisper container stopped."
    else
        log "Whisper container was not running, nothing to stop."
    fi
    rm -f "$TEMP_AUDIO_FILE" 2>/dev/null
}
trap cleanup EXIT

check_dependencies() {
    log "Checking for required dependencies..."
    local missing=()
    command -v curl >/dev/null 2>&1 || missing+=("curl")
    command -v docker >/dev/null 2>&1 || missing+=("docker")
    if [ ${#missing[@]} -gt 0 ]; then
        log "ERROR: Missing dependencies: ${missing[*]}"
        exit 1
    fi
    log "SUCCESS: All dependencies available."
}

start_whisper_container() {
    log "Checking Whisper container..."
    if ! docker ps --filter "name=^/${WHISPER_CONTAINER_NAME}$" --filter "status=running" | grep -qw "${WHISPER_CONTAINER_NAME}"; then
        log "Starting Whisper container..."
        docker start "${WHISPER_CONTAINER_NAME}" || {
            log "ERROR: Failed to start Whisper container."
            exit 1
        }
        sleep 15
    else
        log "Whisper container already running."
    fi
}

check_ffmpeg_container() {
    log "Checking FFmpeg container..."
    if ! docker ps --filter "name=^/${FFMPEG_CONTAINER_NAME}$" --filter "status=running" | grep -qw "${FFMPEG_CONTAINER_NAME}"; then
        log "ERROR: FFmpeg container '${FFMPEG_CONTAINER_NAME}' not running."
        exit 1
    fi
    log "FFmpeg container is running."
}

check_time() {
    local CURRENT_HOUR=$(date +%H)
    if [ "$START_HOUR" -ne "$END_HOUR" ]; then
        if [ "$START_HOUR" -lt "$END_HOUR" ]; then
            if [ "$CURRENT_HOUR" -ge "$END_HOUR" ] || [ "$CURRENT_HOUR" -lt "$START_HOUR" ]; then
                log "Outside allowed window. Exiting."
                exit 0
            fi
        else
            if [ "$CURRENT_HOUR" -ge "$END_HOUR" ] && [ "$CURRENT_HOUR" -lt "$START_HOUR" ]; then
                log "Outside allowed window. Exiting."
                exit 0
            fi
        fi
    fi
}

process_video() {
    local VIDEO_FILE="$1"
    if [ ! -f "$VIDEO_FILE" ]; then return; fi

    local BASENAME_WITH_EXT=$(basename "$VIDEO_FILE")
    local DIR_PATH=$(dirname "$VIDEO_FILE")
    local BASENAME_NO_EXT="${BASENAME_WITH_EXT%.*}"

    local LANG_TAG="${TRANSCRIPTION_LANGUAGE:-en}"
    local FINAL_OUTPUT_FILE="${DIR_PATH}/${BASENAME_NO_EXT}.${LANG_TAG}.srt"
    local TEMP_OUTPUT_FILE="${DIR_PATH}/${BASENAME_NO_EXT}.ai.srt"
    local FAILED_MARKER_FILE="${DIR_PATH}/${BASENAME_NO_EXT}.ai.failed"

    # 1. Skip if we previously failed to transcribe this file
    if [ -f "$FAILED_MARKER_FILE" ]; then 
        SKIPPED_FAILED=$((SKIPPED_FAILED+1))
        return
    fi

    # 2. Skip if ANY subtitle already exists for this video
    shopt -s nullglob
    local EXISTING_SUBS=("${DIR_PATH}/${BASENAME_NO_EXT}"*.srt)
    shopt -u nullglob
    
    if [ ${#EXISTING_SUBS[@]} -gt 0 ]; then 
        SKIPPED_EXISTING=$((SKIPPED_EXISTING+1))
        return
    fi
    
    [ -f "$TEMP_OUTPUT_FILE" ] && rm -f "$TEMP_OUTPUT_FILE"

    local FILE_SIZE_BYTES
    FILE_SIZE_BYTES=$(stat -c%s "$VIDEO_FILE" 2>/dev/null || echo 0)
    if [ "$FILE_SIZE_BYTES" -eq 0 ]; then return; fi

    local FILE_SIZE_MB=$(( FILE_SIZE_BYTES / 1024 / 1024 ))
    if [ "$FILE_SIZE_MB" -gt "$MAX_FILE_SIZE_MB" ]; then
        log "Skipping: ${BASENAME_WITH_EXT} (${FILE_SIZE_MB}MB > limit)"
        SKIPPED_SIZE=$((SKIPPED_SIZE+1))
        return
    fi

    log "--------------------------------------------------------"
    log "Processing: ${BASENAME_WITH_EXT} (${FILE_SIZE_MB}MB)"
    log "Step 1/4: Extracting audio..."
    docker exec "${FFMPEG_CONTAINER_NAME}" ffmpeg -nostdin -i "${VIDEO_FILE}" -vn -acodec pcm_s16le -ar 16000 -ac 1 -y "${TEMP_AUDIO_FILE}" -hide_banner -loglevel error
    
    if [ ! -s "${TEMP_AUDIO_FILE}" ]; then
        log "ERROR: FFmpeg failed to extract audio."
        rm -f "$TEMP_AUDIO_FILE"
        touch "$FAILED_MARKER_FILE"  
        return
    fi

    local API_URL_PARAMS="${WHISPER_API_URL}?task=transcribe&output=${OUTPUT_FORMAT}"
    if [ -n "$INITIAL_PROMPT" ]; then
        local ENCODED_PROMPT="${INITIAL_PROMPT// /%20}"
        API_URL_PARAMS="${API_URL_PARAMS}&initial_prompt=${ENCODED_PROMPT}"
    fi

    log "Step 2/4: Uploading audio..."
    curl -s --fail --connect-timeout 30 --max-time 7200 \
         -X POST -F "audio_file=@${TEMP_AUDIO_FILE}" \
         "${API_URL_PARAMS}" \
         -o "${TEMP_OUTPUT_FILE}"
    local CURL_EXIT=$?
    
    rm -f "$TEMP_AUDIO_FILE"
    
    if [ $CURL_EXIT -ne 0 ]; then
        log "ERROR: curl failed with exit code $CURL_EXIT"
        rm -f "$TEMP_OUTPUT_FILE" 2>/dev/null
        touch "$FAILED_MARKER_FILE"  
        return
    fi

    log "Step 3/4: Verifying & Formatting output..."
    local RENAME_SUCCESS=false
    if [ -s "$TEMP_OUTPUT_FILE" ]; then
        if grep -qF -e '-->' "$TEMP_OUTPUT_FILE"; then
            
            # --- WATERMARK LOGIC ---
            if [ "$ENABLE_WATERMARK" = true ] && [ -n "$WATERMARK_TEXT" ]; then
                log "Applying custom watermark..."
                # Create a Block 0 that lasts for the first 5 seconds
                local WATERMARK_BLOCK="0\n00:00:00,000 --> 00:00:05,000\n${WATERMARK_TEXT}\n\n"
                # Prepend it to the generated subtitle file safely
                echo -e "$WATERMARK_BLOCK" | cat - "$TEMP_OUTPUT_FILE" > "${TEMP_OUTPUT_FILE}.tmp"
                mv "${TEMP_OUTPUT_FILE}.tmp" "$TEMP_OUTPUT_FILE"
            fi
            # ---------------------------

            mv "${TEMP_OUTPUT_FILE}" "${FINAL_OUTPUT_FILE}"
            RENAME_SUCCESS=true
        else
            log "ERROR: Not a valid subtitle. Preview:"
            head -n 5 "$TEMP_OUTPUT_FILE" | while read -r line; do log "  $line"; done
            rm -f "$TEMP_OUTPUT_FILE"
            touch "$FAILED_MARKER_FILE"  
        fi
    else
        log "ERROR: Empty response from API."
        rm -f "$TEMP_OUTPUT_FILE" 2>/dev/null
        touch "$FAILED_MARKER_FILE"  
    fi

    log "Step 4/4: Finalizing..."
    if [ "$RENAME_SUCCESS" = true ]; then
        log "SUCCESS: Created ${FINAL_OUTPUT_FILE}"
        ACTUALLY_PROCESSED=$((ACTUALLY_PROCESSED+1))
    else
        log "ERROR: Failed on ${BASENAME_WITH_EXT}"
    fi
    sleep 2
}

# --- MAIN ---
log "========================================================="
log "Initializing Library Transcription Backfill Script"
log "========================================================="
check_dependencies
check_ffmpeg_container
start_whisper_container

log "Waiting for Whisper API..."
RETRY=0; MAX_RETRIES=30
while [ $RETRY -lt $MAX_RETRIES ]; do
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" --connect-timeout 10 "${WHISPER_API_URL}")
    if [ "$STATUS" -eq 200 ] || [ "$STATUS" -eq 405 ] || [ "$STATUS" -eq 422 ]; then
        log "Whisper API online (status $STATUS)."
        break
    fi
    log "API not ready (status $STATUS). Retrying..."
    sleep 10
    RETRY=$((RETRY+1))
done
[ $RETRY -eq $MAX_RETRIES ] && { log "ERROR: API not available"; exit 1; }

ALL_MEDIA_DIRS=("${MOVIE_DIR}" "${TV_DIR}")

log "Starting transcription..."
log "Time window: ${START_HOUR}:00 - ${END_HOUR}:00 | Max size: ${MAX_FILE_SIZE_MB}MB"
check_time

while IFS= read -r -d '' VIDEO_FILE; do
    check_time
    TOTAL_SCANNED=$((TOTAL_SCANNED+1))
    process_video "$VIDEO_FILE"
done < <(find "${ALL_MEDIA_DIRS[@]}" -type f \( -iname "*.mkv" -o -iname "*.mp4" -o -iname "*.avi" -o -iname "*.mov" -o -iname "*.wmv" -o -iname "*.m4v" \) -print0 2>/dev/null)

log "========================================================="
log "Transcription Session Finished!"
log "Total Videos Scanned: ${TOTAL_SCANNED}"
log "Successfully Transcribed: ${ACTUALLY_PROCESSED}"
log "Skipped (Subtitle Already Exists): ${SKIPPED_EXISTING}"
log "Skipped (Previously Failed): ${SKIPPED_FAILED}"
log "Skipped (Exceeds Size Limit): ${SKIPPED_SIZE}"
log "========================================================="
