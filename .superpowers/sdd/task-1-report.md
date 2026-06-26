# Task 1 Report: AI-Generated Product Images

## Summary

Successfully generated 4 product images for the class reunion merchandise (T-shirt and coffee mug) using the Qwen-Image API via ModelScope.

## API Details

- **Endpoint**: `https://api-inference.modelscope.cn/v1/images/generations`
- **Model**: `Qwen/Qwen-Image`
- **Image Size**: 1024x1024
- **Mode**: Async task submission with status polling
- **Auth**: Bearer token with `ms-*` API key

## Request Format

Two key headers were required for async operation:
- `X-ModelScope-Async-Mode: true` (on POST to submit task)
- `X-ModelScope-Task-Type: image_generation` (on GET to poll task status)

Polling endpoint: `GET /v1/tasks/{task_id}` (polled every 5 seconds until `SUCCEED`).

## Results

| File | HTTP Status | File Size | Format | Polls |
|------|-------------|-----------|--------|-------|
| `tee_front.png` | 200 | 651,189 bytes | PNG | 9 polls (~45s) |
| `tee_back.png` | 200 | 437,228 bytes | PNG | 9 polls (~45s) |
| `mug_front.png` | 200 | 321,309 bytes | PNG | 10 polls (~50s) |
| `mug_back.png` | 200 | 649,053 bytes | PNG | 22 polls (~110s) |

All images returned as URLs (89-char strings pointing to ModelScope-hosted PNG files) which were then downloaded.

## Issues Encountered

1. **Initial prompt-length error**: The `input.prompt` nesting format (ModelScope-native) was rejected with "invalid prompt or prompt length more than 2000". The flat OpenAI-compatible format (`prompt` at top level) worked correctly.

2. **Polling endpoint discovery**: Testing showed:
   - `POST` returns `task_id` with immediate `task_status: SUCCEED`
   - `GET /v1/images/generations/{task_id}` returns 404
   - `GET /v1/tasks/{task_id}` works correctly with the right headers

3. **Model selection**: `Qwen/Qwen-Image-2512` returned sync-style responses with no image data. `Qwen/Qwen-Image` with async headers was the correct combination.

4. **Task status naming**: The API uses `PROCESSING` (not `PENDING` or `RUNNING`) during generation. Some tasks took significantly longer than others (mug_back.png took ~110s vs ~45s for tee images).

## Output Directory

`generated_images_qwen/`

## Scripts

- `generate_images.py` - Main generation script
- `test_formats.py` - Used to test various API payload formats
- `debug_task.py` - Used to debug task polling endpoint
