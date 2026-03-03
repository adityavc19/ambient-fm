# WRIT-FM Operator Session

You are the operator for WRIT-FM, a 24/7 talk-first internet radio station. This is a recurring maintenance session.

## Project Location
Run from the project root directory (where this file lives in `mac/`).

## Your Tasks

### 1. Health Check
```bash
# Check if streamer is running
pgrep -af stream_gapless || echo "STREAMER DOWN"

# Check Icecast
lsof -i :8000 | grep icecast || echo "ICECAST DOWN"

# Check ffmpeg encoder connected
lsof -i :8000 | grep ffmpeg || echo "ENCODER DOWN"
```

If any component is down:
- Icecast: `pkill icecast; icecast -c /opt/homebrew/etc/icecast.xml -b`
- Streamer: `pkill -f stream_gapless; tmux send-keys -t writ "uv run python mac/stream_gapless.py" Enter`
- music-gen.server: `bash mac/start_music_gen.sh server`
- Bumper daemon: `bash mac/start_music_gen.sh daemon`
- Both at once: `bash mac/start_music_gen.sh`
- If writ tmux doesn't exist: `tmux new-session -d -s writ` then send commands to it

Check music-gen.server specifically:
```bash
curl -sf http://localhost:4009/health && echo "music-gen: UP" || echo "music-gen: DOWN"
```

### 2. Check Current Show
```bash
uv run python mac/schedule.py now
```
This tells you which show is active, who's hosting, and the topic focus.

### 3. Generate AI Music Bumpers
Check bumper count per show:
```bash
cd mac/content_generator && uv run python music_bumper_generator.py --status
```

If any show has fewer than 5 bumpers **and music-gen.server is running at localhost:4009**, generate more:
```bash
cd mac/content_generator && uv run python music_bumper_generator.py --all --min 5
```

Or for a specific show:
```bash
cd mac/content_generator && uv run python music_bumper_generator.py --show midnight_signal --count 3
```

Note: music-gen.server must be running separately. If it's not available, skip this step — the streamer falls back to local music automatically. Bumpers are saved to `output/music_bumpers/{show_id}/`.

### 4. Generate Talk Segments
Check segment count per show:
```bash
cd mac/content_generator && uv run python talk_generator.py --status
```

If any show has fewer than 6 segments, generate more:
```bash
cd mac/content_generator && uv run python talk_generator.py --show [SHOW_ID] --count 3
```

Or generate for all shows:
```bash
cd mac/content_generator && uv run python talk_generator.py --all --count 2
```

The generator uses:
- `claude -p` for script generation (long-form talk content)
- Kokoro TTS with show-appropriate voices
- Schedule-aware prompts based on host persona and show context

### 5. Process Listener Messages
```bash
cat ~/.writ/messages.json 2>/dev/null | jq '.[] | select(.read == false)' || echo "No messages file"
```
For each unread message:
1. Note the message content
2. Generate a listener_mailbag segment for the appropriate show:
   ```bash
   cd mac/content_generator && uv run python talk_generator.py --show listener_hours --type listener_mailbag --count 1
   ```
3. Mark as read by updating the JSON

### 6. Review Streamer Status
```bash
tmux capture-pane -t writ -p | tail -20
```
Check for:
- Pipe failures or encoder restarts
- Current show and host displayed correctly
- Talk segments playing with music bumpers between them
- Fallback music mode (means that show needs more talk segments!)

### 7. Log Status
Append to daily log:
```bash
LOGFILE="output/operator_$(date +%Y-%m-%d).log"
echo "" >> "$LOGFILE"
echo "## WRIT-FM $(date +%H:%M)" >> "$LOGFILE"
echo "- Show: $(uv run python mac/schedule.py now 2>/dev/null | head -1)" >> "$LOGFILE"
echo "- Encoder: $(lsof -i :8000 | grep ffmpeg > /dev/null && echo 'connected' || echo 'DOWN')" >> "$LOGFILE"
cd mac/content_generator && uv run python talk_generator.py --status 2>/dev/null >> "$LOGFILE"
```

## Key Files
- `mac/stream_gapless.py` - Main streamer (talk-first with AI music bumpers)
- `mac/schedule.py` - Schedule parser and resolver
- `config/schedule.yaml` - Weekly show schedule (8 talk shows)
- `mac/content_generator/talk_generator.py` - Talk segment generator (Claude + Kokoro)
- `mac/content_generator/persona.py` - Multi-host persona system
- `mac/content_generator/music_bumper_generator.py` - AI music bumper generator (ACE-Step)
- `mac/music_gen_client.py` - REST client for music-gen.server
- `output/talk_segments/[show_id]/` - Generated talk segments per show
- `output/music_bumpers/[show_id]/` - Pre-generated AI music bumpers per show

## Schedule Overview
The station runs different talk shows based on time and day:

**Base Schedule (daily):**
- 00:00-04:00: Midnight Signal (Liminal Operator - philosophy)
- 04:00-06:00: The Night Garden (Nyx - dreams/night)
- 06:00-09:00: Dawn Chorus (Liminal Operator - morning reflections)
- 09:00-12:00: Sonic Archaeology (Dr. Resonance - music history)
- 12:00-14:00: Signal Report (Signal - news analysis)
- 14:00-16:00: The Groove Lab (Ember - soul/funk)
- 16:00-18:00: Crosswire (panel/debate format)
- 18:00-20:00: Sonic Archaeology (Dr. Resonance - music history)
- 20:00-22:00: The Groove Lab (Ember - soul/funk)
- 22:00-00:00: The Night Garden (Nyx - dreams/night)

**Weekly Override:**
- Sun 18:00-20:00: Listener Hours (mailbag)

## Hosts & Voices
- **The Liminal Operator** (`am_michael`): Overnight philosophy, morning reflections
- **Dr. Resonance** (`bm_daniel`): Music history, genre archaeology
- **Nyx** (`af_heart`): Nocturnal voice, dreams, night philosophy
- **Signal** (`am_onyx`): News analysis, current events
- **Ember** (`af_bella`): Soul, warmth, groove, music as feeling

## Segment Types
Long-form (primary content):
- `deep_dive` - Extended single-topic exploration (1500-2500 words)
- `news_analysis` - Current events analysis (uses RSS headlines)
- `interview` - Simulated interview with historical/fictional figure
- `panel` - Two hosts discuss topic from different angles
- `story` - Narrative storytelling
- `listener_mailbag` - Listener letters + responses
- `music_essay` - Extended essay on artist/album/genre

Short-form (transitions):
- `station_id`, `show_intro`, `show_outro`

## Notes
- Don't restart the streamer unless it's actually down
- Talk segments are organized by show ID in `output/talk_segments/`
- The streamer plays talk segments then deletes them after playing
- AI music bumpers (70-110s) play between talk segments — fully AI generated via ACE-Step
- If no AI bumpers available, streamer falls back to local music library automatically
- If a show has no talk segments, streamer falls back to music-only mode
- Keep each show stocked with at least 6 talk segments
- Keep each show stocked with at least 5 AI music bumpers
- Priority: generate for shows with the fewest segments first
- music-gen.server runs separately — start it before generating bumpers
