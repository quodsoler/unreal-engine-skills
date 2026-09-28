# Global Shaders and the Render Dependency Graph


For a real HLSL pass, declare a `FGlobalShader` subclass and drive it with the Render Dependency Graph. Build.cs needs `"RenderCore"`, `"RHI"` and `"Projects"`.

```cpp
#include "GlobalShader.h"
#include "ShaderParameterMacros.h"
#include "ShaderParameterStruct.h"

class FMyBlurCS : public FGlobalShader
{
public:
    DECLARE_GLOBAL_SHADER(FMyBlurCS);
    SHADER_USE_PARAMETER_STRUCT(FMyBlurCS, FGlobalShader);

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER(float, Radius)
        SHADER_PARAMETER_RDG_TEXTURE(Texture2D, InputTexture)
        SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float4>, OutputTexture)
    END_SHADER_PARAMETER_STRUCT()
};

IMPLEMENT_GLOBAL_SHADER(FMyBlurCS, "/MyGame/MyBlur.usf", "MainCS", SF_Compute);
```

Passes are added to an `FRDGBuilder` with `GraphBuilder.AddPass(RDG_EVENT_NAME("MyBlur"), Parameters, ERDGPassFlags::Compute, [](FRHICommandList& RHICmdList) { ... })` and submitted with `GraphBuilder.Execute()`. Shader maps come from `GetGlobalShaderMap(EShaderPlatform Platform)`.

The `/MyGame/` virtual path must be registered before shaders compile — do it in the module's `StartupModule`, and give that module `"LoadingPhase": "PostConfigInit"` in the `.uplugin`/`.uproject`; a global shader type registered later hits the "Shader type was loaded too late" `checkf` (`Shader.cpp:315`). The usual `Default` phase is too late:

```cpp
#include "Interfaces/IPluginManager.h"   // "Projects" module
#include "ShaderCore.h"                  // RenderCore; AddShaderSourceDirectoryMapping(const FString& VirtualShaderDirectory, const FString& RealShaderDirectory)

void FMyGameModule::StartupModule()
{
    const FString ShaderDir = FPaths::Combine(
        IPluginManager::Get().FindPlugin(TEXT("MyGame"))->GetBaseDir(), TEXT("Shaders"));   // plugin module; for a game module use FPaths::ProjectDir()
    AddShaderSourceDirectoryMapping(TEXT("/MyGame"), ShaderDir);
}
```
