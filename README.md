## V2 Changes (NEW)
*   Album cover art fetching issues fixed
*   Spotify API instead of MusicBrainz API for better cover art fetching
*   Can now take either data from last.fm or directly from Spotify
*   Optimizations to rendering including parallelization
*   Different sampling rates (frame aggregation periods) available so you don't have to render as many frames

## OLD README
Will fix later once completely finished, working on v3 now to add rolling 7 day and 30 day windows to display top track of the past 7 days and top track of the past 30 days. Also looking to add a top artists chart as well. 

Takes your Last.fm listening history for 2024 and generates a funny bar chart "race" video of your top 10 songs throughout the year complete with album cover art. 

## Demo Output

Generates an MP4 video file named `spotify_top_songs_2024_4k.mp4`. 
Mine is currently in the repo with this name, but will overwrite if you decide to run locally.

[https://www.youtube.com/watch?v=z108hBXyb9w]

[![Youtube](https://img.youtube.com/vi/z108hBXyb9w/hqdefault.jpg)](https://www.youtube.com/watch?v=z108hBXyb9w)

## Project Structure

```
spotify2024/
├── .venv/                   # Virtual environment
├── album_art_cache/         # Downloaded album cover art and MBIDs
│   ├── mbid_cache.json
│   └── dominant_color_cache.json
├── fonts/                   # Fonts (helps with foreign characters)
├── main_animator.py         # Main as well as runs the animation
├── data_processor.py        # Data cleaning / preparation
├── album_art_utils.py       # Functions for fetching album art and colors (done calling MusicBrainz api)
├── lastfm_data.csv          # Your Last.fm listening history (UTF-8 encoded CSV)
├── README.md                # This file
└── (other generated files like .mp4 output)
```

## Issues
*   Some album art not fetched properly (most notably 晴る - Yorushika in demo)
*   Has "bandaid" solutions for the album cover art. It begins showing at 110 plays as that's when clipping stops for my scale, but this will be different depending on how many    plays your top song has. You can change this here in main_animator.py (if pil_image and values_for_actual_bars[i] >= 110:)
*   Album cover art is also only fetched if more than 50 plays were found and not seen in January (part of another bandaid solution) Fetch threshold can be changed in `main_animator.py` (`def pre_fetch_album_art_and_colors(race_df, song_album_map, unique_song_ids_in_race, fetch_threshold=50):`), and checking the January part can be changed in the function (`for song_id in unique_song_ids_in_race:`). Technically this logic useless for me as there's no point in fetching album covers if they're not going to display at less than 110 plays but I'm too lazy to get rid of this. 
*   If I fix this, it'll probably be for EOY 2025 so I can run this again for 2025 lmao

## Prerequisites/Requirements

1.  **FFmpeg**: needed to save animation as mp4
    *   Download from [https://ffmpeg.org] and add to path
2.  **Python Dependencies:**
    pandas
    matplotlib
    Pillow
    requests
3.  **Last.fm Data:**
    *   Download from [https://lastfm.ghan.nl/export/]
    *   Name the file `lastfm_data.csv` and place it in the project directory
4. **API Configuration:**
    *   Make sure the `USER_AGENT` in `album_art_utils.py` is set to your own email address for API calls
    *   So replace `lightningfalconyt@gmail.com` with your email
5. **Run**
    *   Make sure to use terminal (not "output" in VSCode)
    *   utf-8 is necessary if music has foreign characters

    Command Prompt (cmd):
    ```shell
    set PYTHONIOENCODING=utf-8
    python "path\to\your\project\directory\main_animator.py"
    ```
    PowerShell:
    ```shell
    $env:PYTHONIOENCODING="utf-8"
    python "path\to\your\project\directory\main_animator.py"
    ```

    *   Rendering takes around 40 minutes on my machine (only uses cpu, no gpu encoding so its pretty unoptimized lol)

## Configuration

You can adjust some settings by changing the constants at the top of `main_animator.py` and `album_art_utils.py`.

### In `main_animator.py`:

*   **N_BARS**: `10`  
    Number of top songs to display in the bar chart
*   **TARGET_FPS**: `30`  
    FPS for video (higher means longer rendering time)
*   **ANIMATION_INTERVAL**: `1000 / TARGET_FPS`  
    Milliseconds between animation frames.
*   **OUTPUT_FILENAME**: `"spotify_top_songs_2024_4k.mp4"`  
    Name of video file.
*   **VIDEO_RESOLUTION_WIDTH**: `3840`  
    Width of output video (pixels)
*   **VIDEO_RESOLUTION_HEIGHT**: `2160`  
    Height of output video (pixels)
*   **VIDEO_DPI**: `165`  
    Dots Per Inch for the Matplotlib plot rendering; impacts relative size of graph and text
*   **DEBUG_CACHE**: `True`  
    Set to `False` to disable debug messages for album art pre-fetching
*   **plt.rcParams['animation.ffmpeg_path']**: commented out  
    Uncomment if having trouble setting ffmpeg to path
*   **PREFERRED_FONTS**: `['DejaVu Sans', 'Noto Sans JP', ...]`  
    Font list

### In `album_art_utils.py`:

*   **DEBUG_CACHE**: `True`  
    Set to `False` to disable console logging for MusicBrainz ID (MBID) caching, dominant color caching, and image download caching

## How It Works

1.  **Data Loading & Cleaning (`data_processor.py`):**
    *   Loads `lastfm_data.csv`.
    *   Converts Unix timestamps to datetime objects.
    *   Filters data for the year 2024.
    *   Cleans data by dropping rows with missing essential information (artist, album, track).

2.  **Data Preparation for Race (`data_processor.py`):**
    *   Creates a unique `song_id` (Artist - Track).
    *   Maps `song_id` to album names.
    *   Generates a "race-ready" DataFrame where:
        *   Index: Timestamps of each individual play event (for smooth animation).
        *   Columns: Unique `song_id`s.
        *   Values: Cumulative play counts for each song at that exact timestamp. This involves pivoting and forward-filling play counts.

3.  **Album Art & Color Fetching (`album_art_utils.py`):**
    *   For each unique album, it tries to find a MusicBrainz Release ID (`mbid`).
    *   Uses the `mbid` to query the Cover Art Archive for an album art URL.
    *   Downloads the album art and saves it to the `album_art_cache` directory.
    *   Calculates the dominant color from the downloaded album art using Pillow.
    *   MBIDs, image paths, and dominant colors are cached in JSON files (`mbid_cache.json`, `dominant_color_cache.json`) and the `album_art_cache` folder to avoid re-fetching on subsequent runs.

4.  **Animation (`main_animator.py`):**
    *   Initializes a Matplotlib figure and axes.
    *   Pre-fetches album art and dominant colors for songs appearing in the race.
    *   The `draw_frame` function is called for each frame (each play event timestamp):
        *   It gets the top N songs by play count at that specific timestamp.
        *   Draws horizontal bars for these songs.
        *   Sets the bar color using the cached dominant color for the song's album.
        *   Overlays album art (if found) next to the corresponding bar.
        *   Updates the date/time counter on the animation.
    *   Uses `matplotlib.animation.FuncAnimation` to create the animation.
    *   Saves the animation as an MP4 file using FFmpeg.
