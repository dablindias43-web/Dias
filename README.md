# tiktok_builder.py
"""
Simple TikTok-style video builder using moviepy.
Place your source files (video/image/audio) in the same folder or give full paths.
"""

from moviepy.editor import (
    VideoFileClip, ImageClip, AudioFileClip,
    concatenate_videoclips, CompositeVideoClip, TextClip
)
import os

# ---------- User settings ----------
OUTPUT = "tiktok_output.mp4"
TARGET_W, TARGET_H = 1080, 1920          # TikTok vertical resolution
CLIPS = [
    # list source clips (video files) or images (jpg/png) with desired duration for images
    {"path": "clip1.mp4", "type": "video"},
    {"path": "clip2.mp4", "type": "video"},
    {"path": "image1.jpg", "type": "image", "duration": 3},  # 3 seconds
]
MUSIC = "background_music.mp3"            # set None if no music
MUSIC_VOLUME = 0.6                        # 0.0..1.0
CAPTION = "I love this moment ❤️"         # Caption text
CAPTION_FONT = "Arial-Bold"               # system font or path to ttf
CAPTION_FONT_SIZE = 70
CAPTION_DURATION = 4                      # seconds caption stays on screen
# ------------------------------------

def load_clip(item):
    path = item["path"]
    if not os.path.exists(path):
        raise FileNotFoundError(f"File not found: {path}")

    if item["type"] == "video":
        clip = VideoFileClip(path)
    elif item["type"] == "image":
        clip = ImageClip(path).set_duration(item.get("duration", 3))
    else:
        raise ValueError("Unknown clip type: must be 'video' or 'image'")

    # Resize while keeping aspect ratio, then center on a TARGET_W x TARGET_H black background
    clip = clip.resize(width=TARGET_W) if clip.w/clip.h > TARGET_W/TARGET_H else clip.resize(height=TARGET_H)
    # center by padding if necessary
    clip = clip.on_color(size=(TARGET_W, TARGET_H), color=(0,0,0), pos=("center","center"))
    # set fps to 30 for consistent output
    return clip.set_fps(30)

def main():
    clips = []
    for i, item in enumerate(CLIPS):
        clip = load_clip(item)
        # Optional: trim long clips to 12 seconds per TikTok short
        max_len = 12
        if clip.duration > max_len:
            clip = clip.subclip(0, max_len)
        # Add short crossfade except for the first clip
        if i != 0:
            clip = clip.crossfadein(0.5)
        clips.append(clip)

    # Concatenate with crossfade
    final = concatenate_videoclips(clips, method="compose")

    # Add caption text as a TextClip, positioned near bottom (safe area)
    txt = TextClip(
        CAPTION,
        fontsize=CAPTION_FONT_SIZE,
        font=CAPTION_FONT,
        method="caption",  # wraps text
        size=(TARGET_W*0.9, None)  # wrap within 90% of width
    ).set_position(("center", TARGET_H - 220)).set_duration(CAPTION_DURATION)

    # Add basic fade in/out for caption
    txt = txt.crossfadein(0.5).crossfadeout(0.5)

    # Composite video + caption
    composed = CompositeVideoClip([final, txt], size=(TARGET_W, TARGET_H))

    # Add background music if provided
    if MUSIC and os.path.exists(MUSIC):
        audio = AudioFileClip(MUSIC).volumex(MUSIC_VOLUME)
        # If the music is shorter than the video, loop it
        if audio.duration < composed.duration:
            audio = audio.audio_loop(duration=composed.duration)
        else:
            audio = audio.subclip(0, composed.duration)
        composed = composed.set_audio(audio)
    else:
        # keep original audio from clips (already composed in final)
        pass

    # Export: use bitrate and preset for good quality & small size
    composed.write_videofile(
        OUTPUT,
        codec="libx264",
        audio_codec="aac",
        fps=30,
        preset="medium",
        threads=4,
        bitrate="4M",
        ffmpeg_params=["-movflags", "+faststart"]  # helps with streaming/upload
    )

if __name__ == "__main__":
    main()
