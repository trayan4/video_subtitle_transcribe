# Video Subtitle Translator (AutoGen + Whisper)

This notebook turns the speech in a video into timestamped subtitles in another language.
Two AutoGen agents work together: an LLM-driven `chatbot` decides which tool to call next,
and a `user_proxy` agent runs those tools on your machine.

- **Speech-to-text:** [OpenAI Whisper](https://github.com/openai/whisper), running locally with the `small` model.
- **Translation:** OpenAI's Chat Completions API (`gpt-4`).
- **Orchestration:** [AG2](https://github.com/ag2ai/ag2), the community fork of AutoGen 0.2, imported as `autogen`.

## Workflow

```
            ┌───────────────────────────── user_proxy ─────────────────────────────┐
 prompt ──▶ │ "For the video located in <target_video>, recognize the speech ...  │
            └──────────────────────────────────┬───────────────────────────────────┘
                                               ▼
                               chatbot (gpt-4) picks a function
                                               │
              ┌────────────────────────────────┴────────────────────────────────┐
              ▼                                                                 ▼
 1. recognize_transcript_from_video(audio_filepath)            2. translate_transcript(source, target)
    • Whisper transcribes the video's audio                        • reads transcription.txt line by line
    • writes transcription.txt                                     • translate_text() sends each sentence
      "<start>s to <end>s: <sentence>"                               to the OpenAI API, keeping its timestamp
    • returns the transcript text to the chatbot                   • writes transcription_<target>.txt
              │                                                                 │
              └──────────────────────► chatbot replies "TERMINATE" ◄────────────┘
                                               │
                                               ▼
                                         chat ends
```

Step by step:

1. **Start the chat.** `user_proxy.initiate_chat(...)` sends the chatbot a request naming the video and the source and target languages.
2. **Transcribe.** The chatbot asks for `recognize_transcript_from_video`, and `user_proxy` runs it locally:
   - Whisper (`small` model) transcribes the audio. ffmpeg decodes the file, so `.mp4`, `.webm` and `.mkv` all work.
   - Each Whisper segment becomes one line of `transcription.txt`, e.g. `13.6s to 17.6s: My name is Gideon and my user name is Nangara in Australia.`
   - The file's contents are returned to the chatbot as a string. AG2 only accepts strings or numbers as function results.
3. **Translate.** The chatbot then asks for `translate_transcript`:
   - Each line is split into its timestamp and its text.
   - Only the text goes to `translate_text`, which makes one OpenAI call per line.
   - The translated lines are joined back to their timestamps and saved to `transcription_<target_language>.txt`, e.g. `transcription_es.txt`.
4. **Finish.** When both steps are done, the chatbot replies `TERMINATE`. `user_proxy`'s `is_termination_msg` check catches that and ends the chat. `max_consecutive_auto_reply=10` stops a runaway conversation if something goes wrong.

## Notebook layout

| Cell | What it does |
|---|---|
| 0 | Imports (`whisper`, `openai`, `autogen`, `dotenv`) |
| 1 | Loads `OPENAI_API_KEY` from `.env` and builds `config_list` |
| 2 | Tool functions: `recognize_transcript_from_video`, `translate_text`, `translate_transcript` |
| 3 | `llm_config` (the function schemas shown to the LLM) and the run settings: languages and `target_video` |
| 4 | Creates the `chatbot` and `user_proxy` agents, registers the functions, and starts the chat |

## Setup

**Requirements:** Python 3.13 (tested with Anaconda), [ffmpeg](https://ffmpeg.org/) on your `PATH`, and an OpenAI API key.

1. Install ffmpeg. Whisper calls the `ffmpeg` command to decode audio.
   ```bash
   brew install ffmpeg
   ```
2. Install the Python packages.
   ```bash
   pip install openai-whisper "ag2[openai]<1" "openai>=2,<3" python-dotenv
   ```
   Pin `ag2` below 1.0: from 1.0 on, the module is renamed `ag2` and `import autogen` no longer works.
   Don't install `pyautogen` either. Versions 0.10 and later are a wrapper for Microsoft's newer AutoGen and don't include an `autogen` module.
3. Create a `.env` file in the project folder:
   ```
   OPENAI_API_KEY=sk-...
   ```
   Keep `.env` out of version control.
4. Put a video with spoken English in the project folder and set `target_video` in cell 3 to its file name or path.
   The sample used here is [*The Impact of Wikipedia*](https://commons.wikimedia.org/wiki/File:The_Impact_of_Wikipedia_Gideon_Digby.webm) (3½ min, CC BY-SA 3.0) from Wikimedia Commons.

## Running

Open `video_subtitle_transcribe.ipynb`, select the Python environment where you installed the packages, and run all cells.
The first run downloads the Whisper `small` model (~460 MB) to `~/.cache/whisper`.
For a 3½-minute video, the whole run takes about 2 minutes on an Apple Silicon Mac.

To translate into another language, change `target_language` in cell 3 (e.g. `"French"`).

## Output files

| File | Contents |
|---|---|
| `transcription.txt` | English transcript, one timestamped line per Whisper segment |
| `transcription_<lang>.txt` | Translated transcript with the same timestamps. `<lang>` is whatever value the chatbot passes as the target language, e.g. `es` or `spanish` |

## Known limitations

- **The output file name can change between runs.** The chatbot sometimes passes `"es"` and sometimes `"spanish"` as the target language, so the file name changes with it.
- **Translated lines are often wrapped in quotation marks.** `translate_text` sends each line in quotes, and the model keeps them in its reply.
- **Whisper's mistakes carry into the translation.** Misheard English is translated literally. The `medium` or `large` Whisper model hears speech more accurately but runs slower.
- **Translation cost grows with video length.** It makes one `gpt-4` call per line. A cheaper model such as `gpt-4o-mini` works too: change `model=` in `translate_text`.
- **`use_docker` is `False`.** `user_proxy` only runs the two registered functions here. If you let it execute LLM-generated code, turn Docker on.
