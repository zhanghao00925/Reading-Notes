Shaders
The base class that all shaders in Unreal derive from is FShader. Unreal has two main classifications for shaders, FGlobalShader for cases where only one instance should exist, and FMaterialShader for shaders that are tied to materials. FShader is paired with an FShaderResource which keeps track of the resource on the GPU associated with that particular shader. An FShaderResource can be shared between multiple FShaders if the compiled output from the FShader matches an already existing one.

FGlobalShader
Only one instance of a Global Shader exists, which means that you can’t have per-instance parameters. However, you can have global parameters.

FMaterialShader and FMeshMaterialShader
Both of these classes allow multiple instances, each one associated with its own copy of the GPU resource.

Drawing Policies
Conceptually the drawing policy determines which shader variations are used to draw something, but it doesn’t pick what it draws or when it’s drawn!