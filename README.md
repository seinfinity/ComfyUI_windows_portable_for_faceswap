# ComfyUI_windows_portable_for_faceswap
ComfyUI_windows_portable_for_faceswap

# Resolving Errors in `VHS_VideoCombine` Function for FaceSwap

In the process of face-swapping videos using the `VHS_VideoCombine` function within the ComfyUI framework, a series of errors were encountered and resolved. This document outlines the changes made to address these issues.

## Errors Encountered

1. **Invalid Argument Error:**
   ```
   Error occurred when executing VHS_VideoCombine:
   [Errno 22] Invalid argument
   ```
2. **Bytes Object Error:**
   ```
   'bytes' object has no attribute 'tobytes'
   ```

## Modifications Made

### 1. `ffmpeg_process` Function

The `ffmpeg_process` function was updated to handle frame data properly and include logging for better debugging. Key changes include:
- Checking if `frame_data` is `None` or empty before writing to `ffmpeg`.
- Adding logging to track the writing process.

```python
import logging
import subprocess

logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger(__name__)

def ffmpeg_process(args, video_format, video_metadata, file_path, env):
    res = None
    frame_data = yield
    total_frames_output = 0
    if video_format.get('save_metadata', 'False') != 'False':
        os.makedirs(folder_paths.get_temp_directory(), exist_ok=True)
        metadata = json.dumps(video_metadata)
        metadata_path = os.path.join(folder_paths.get_temp_directory(), "metadata.txt")
        metadata = metadata.replace("\\","\\\\").replace(";","\\;").replace("#","\\#").replace("=","\\=").replace("\n","\\\n")
        metadata = "comment=" + metadata
        with open(metadata_path, "w") as f:
            f.write(";FFMETADATA1\n")
            f.write(metadata)
        m_args = args[:1] + ["-i", metadata_path] + args[1:] + ["-metadata", "creation_time=now"]
        with subprocess.Popen(m_args + [file_path], stderr=subprocess.PIPE, stdin=subprocess.PIPE, env=env) as proc:
            try:
                while frame_data is not None:
                    if frame_data:
                        logger.debug("Writing frame data to ffmpeg stdin")
                        proc.stdin.write(frame_data)
                    else:
                        logger.error("Frame data is None or empty")
                    frame_data = yield
                    total_frames_output += 1
                proc.stdin.flush()
                proc.stdin.close()
                res = proc.stderr.read()
            except BrokenPipeError as e:
                err = proc.stderr.read()
                if os.path.exists(file_path):
                    raise Exception("An error occurred in the ffmpeg subprocess:\n" + err.decode("utf-8"))
                print(err.decode("utf-8"), end="", file=sys.stderr)
                logger.warn("An error occurred when saving with metadata")
    if res != b'':
        with subprocess.Popen(args + [file_path], stderr=subprocess.PIPE, stdin=subprocess.PIPE, env=env) as proc:
            try:
                while frame_data is not None:
                    if frame_data:
                        logger.debug("Writing frame data to ffmpeg stdin")
                        proc.stdin.write(frame_data)
                    else:
                        logger.error("Frame data is None or empty")
                    frame_data = yield
                    total_frames_output += 1
                proc.stdin.flush()
                proc.stdin.close()
                res = proc.stderr.read()
            except BrokenPipeError as e:
                res = proc.stderr.read()
                raise Exception("An error occurred in the ffmpeg subprocess:\n" + res.decode("utf-8"))
    yield total_frames_output
    if len(res) > 0:
        print(res.decode("utf-8"), end="", file=sys.stderr)
```

### 2. `combine_video` Function

The `combine_video` function was updated to handle both bytes and non-bytes frame data correctly. Key changes include:
- Checking if `image` is an instance of `bytes` before attempting to call `tobytes()`.

```python
def combine_video(
    self,
    images,
    frame_rate: int,
    loop_count: int,
    filename_prefix="AnimateDiff",
    format="image/gif",
    pingpong=False,
    save_output=True,
    prompt=None,
    extra_pnginfo=None,
    audio=None,
    unique_id=None,
    manual_format_widgets=None,
    meta_batch=None
):
    ...
    if output_process is None:
        if 'gifski_pass' in video_format:
            output_process = gifski_process(args, video_format, file_path, env)
        else:
            args += video_format['main_pass'] + bitrate_arg
            output_process = ffmpeg_process(args, video_format, video_metadata, file_path, env)
        # Proceed to first yield
        output_process.send(None)
        if meta_batch is not None:
            meta_batch.outputs[unique_id] = (counter, output_process)

    for image in images:
        if image is None:
            logger.error("Frame data is None")
            continue
        pbar.update(1)
        # Check if image is already bytes
        if isinstance(image, bytes):
            output_process.send(image)
        else:
            output_process.send(image.tobytes())
    ...
```

## Conclusion

These modifications resolved the errors encountered during the execution of the `VHS_VideoCombine` function. For a complete reference, the entire updated `node.py` file can be found in the project repository.

For further details, please refer to the `node.py` file in the project repository.

---
