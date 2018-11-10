Basilisk II classic Macintosh emulator in the browser. [Try it out](https://jamesfriend.com.au/projects/basiliskii/BasiliskII-worker.html)

![Screenshot of Basilisk II in the browser](https://jamesfriend.com.au/files/basiliskiijs.png)

### How this works

Basically, the emulator code is compiled with Emscripten and run in a Web Worker. Communication with the main thread happens by reading and writing SharedArrayBuffers shared between the main browser thread and the worker, which allows the emulator code to run without ever yielding. This allows the simulation to be more smooth than if we had to regularly yield the browser event loop, and also simplifies the porting work in some ways.

The original emulator codebase makes use of SDL (a cross-platform set of video/audio/input APIs), and Emscripten provides an implementation of SDL, but it doesn't work when the code is running in a web worker, so I've hacked the emulator's video output code to write to the SharedArrayBuffer instead of calling SDL APIs. You can see [in video_sdl.cpp](https://github.com/jsdf/macemu/blob/80557e8ff1f411f0fe850669502c2ba5fe056b96/BasiliskII/src/SDL/video_sdl.cpp#L2446-L2449) 
where the `EM_ASM_` macro is used to call [this JS function](https://github.com/jsdf/macemu/blob/1730d17db9e8f3832397e91ce2a43a43aaccfeae/BasiliskII/src/Unix/BasiliskII-worker-boot.js#L136-L146) to copy the contents of the video framebuffer to a SharedArrayBuffer. This shared memory is read from the main thread [here](https://github.com/jsdf/macemu/blob/1730d17db9e8f3832397e91ce2a43a43aaccfeae/BasiliskII/src/Unix/BasiliskII-worker-ui-thread.js#L148-L171) and output onto a Canvas. I experimented with locking the video surface when reading and writing, but this hurt performance due to lock contention. However, we don't really need it. Even if the UI thread reads a frame from the video framebuffer while it is currently in the middle of being written to from the emulator worker, it's not really noticable, because in the shared memory the new frame contents is just being written directly over the old one at the same position, so visually it just presents as a bit of 'tearing' (showing part of the old frame, part of the new frame). We could improve on this by using a circular queue of framebuffers for alternating frames (multiple buffering), which would allow locking (with less contention), but in practice this is okay.

Mouse and keyboard input are communicated via another SharedArrayBuffer [here](https://github.com/jsdf/macemu/blob/1730d17db9e8f3832397e91ce2a43a43aaccfeae/BasiliskII/src/Unix/BasiliskII-worker-ui-thread.js#L66-L117) in the UI thread JS, then read in the worker thread JS [here](https://github.com/jsdf/macemu/blob/1730d17db9e8f3832397e91ce2a43a43aaccfeae/BasiliskII/src/Unix/BasiliskII-worker-boot.js#L91-L119), when requested from the C code [here](https://github.com/jsdf/macemu/blob/80557e8ff1f411f0fe850669502c2ba5fe056b96/BasiliskII/src/SDL/video_sdl.cpp#L2167-L2230). I also had to add a mapping to convert JS keycodes to ADB (Apple) keycodes.

The implementation of audio uses a queue of buffers in shared memory, which each have a flag signifying whether they are full and ready to be consumed by the UI thread, or empty and ready to be written by the emulator thread. The emulator thread will ensure that there are several buffers of audio queued up, to avoid gaps in playback in the case of a slowdown. When the emulator is built natively, audio has a dedicated thread to avoid slowness on the emulator thread affecting playback, but to make things simpler I've just hacked in a call to output audio directly into the emulator's main loop [here](https://github.com/jsdf/macemu/blob/80557e8ff1f411f0fe850669502c2ba5fe056b96/BasiliskII/src/Unix/main_unix.cpp#L1261-L1265). This could be improved by using Emscripten's pthreads support to spawn a separate audio thread. The audio buffer is passed to JS [here](https://github.com/jsdf/macemu/blob/80557e8ff1f411f0fe850669502c2ba5fe056b96/BasiliskII/src/Unix/audio_oss_esd.cpp#L615-L618) and copied into a SharedArrayBuffer [here](https://github.com/jsdf/macemu/blob/bas-emscripten-release/BasiliskII/src/Unix/BasiliskII-worker-boot.js#L238-L266). The audio read in the UI thread [here](https://github.com/jsdf/macemu/blob/bas-emscripten-release/BasiliskII/src/Unix/BasiliskII-worker-ui-thread.js#L137-L159) and output using the Web Audio API.

In future this implementation could be improved by moving all of the shared memory communication to C code by leveraging Emscripten's pthreads support. In this scenario the emulator code would start up, spawn a thread to run the emulator main loop, and then yield back to the browser. Instead of creating SharedArrayBuffers to communicate video, audio, and input, Emscripten's pthreads mode makes the entire C heap memory space a SharedArrayBuffer, with each spawned thread running in a web worker. The JS would simply read and write values on the C heap. This would also unblock using threads in the emulator code (currently not easy due to Chrome's lack of support for web workers spawning other web workers). 

## build instructions

See the NOTES file for more options. Basic build:

```sh
source /path/to/emsdk/emsdk_set_env.sh

cd ./BasiliskII/src/Unix/

./_embuild.sh 
make clean
make
./_emafterbuild.sh 
http-server .
open http://localhost:8080/BasiliskII-worker.html
```
