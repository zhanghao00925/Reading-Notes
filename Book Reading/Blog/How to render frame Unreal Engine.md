# HOW UNREAL RENDERS A FRAME

https://interplayoflight.wordpress.com/2017/10/25/how-unreal-renders-a-frame/

https://weekly-geekly.github.io/articles/341080/index.html

## Note

Finally a word of advice, if you have stationary lights in the scene make sure that you bake lighting before doing any profiling in the Editor (at least, I am not sure what running the game as “standalone” does), Unreal seems to treat them as dynamic, producing cubemaps instead of using per object shadows, if not.

Obviously, in the case of buffer sizes larger than 1024 bytes, the source code prefers to use cleanup using the compute shader instead of using the API cleanup call.

