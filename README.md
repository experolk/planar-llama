# llama.cpp with PlanarQuant KV cache

The purpose of this project is to be able to run an LLM model on a garden-variety HP Elitebook with 32 GiB RAM.

## Install Prerequisites

### Windows SDK Build Tools

This package contains the compiler and the linker.

```ps
Install-WinGetPackage `
 -MatchOption Equals `
 -Id Microsoft.VisualStudio.BuildTools `
 -Version 18.5.1 `
 -Scope System
```
(this will download approx. 740 MiB)

```ps
& "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\setup.exe" `
 modify --passive --includeRecommended `
 --installPath "${env:ProgramFiles(x86)}\Microsoft Visual Studio\18\BuildTools" `
 --add Microsoft.Component.MSBuild `
 --add Microsoft.VisualStudio.Component.VC.CoreBuildTools `
 --add Microsoft.VisualStudio.Component.VC.CMake.Project `
 --add Microsoft.VisualStudio.Component.Vcpkg `
 --add Microsoft.VisualStudio.Component.VC.Tools.x86.x64 `
 --add Microsoft.VisualStudio.Component.Windows11SDK.26100 `
 --add Microsoft.Component.VC.Runtime.UCRTSDK
```

### AOCL BLAS

This package contains the parallelized matrix library optimized for AMD processors.

```ps
Start-BitsTransfer `
 -Source https://download.amd.com/developer/eula/aocl/aocl-5-2/AOCL_Windows-setup-5.2.0-AMD.exe `
 -Destination .\AOCL_Windows-setup-5.2.0-AMD.exe
```

```ps
Start-Process -Wait -Verb RunAs -FilePath .\AOCL_Windows-setup-5.2.0-AMD.exe -ArgumentList ""
```

## Build

Initialize a development shell (note the trailing backslash in `installPath`):

```ps
$installPath = "${env:ProgramFiles(x86)}\Microsoft Visual Studio\18\BuildTools\"
Import-Module (Join-Path ${installPath} "Common7\Tools\Microsoft.VisualStudio.DevShell.dll")
Enter-VsDevShell `
 -VsInstallPath ${installPath} `
 -SkipAutomaticLocation `
 -HostArch ${env:Processor_Architecture} `
 -DevCmdArguments "-no_logo"
```

Get the location of the matrix library:

```ps
$AOCL_DIR = Get-ItemPropertyValue -Path "HKLM:\SOFTWARE\AMD\AOCL" -Name "InstallationPath"
```

Configure the build to use CPU and matrix library:

```ps
cmake -S . -B .\build -G "Visual Studio 18 2026" -Wno-dev -A x64 -DCMAKE_BUILD_TYPE=Release `
 -DCMAKE_ASM_COMPILER=ml64 -DCMAKE_CXX_FLAGS="/DNOMINMAX /EHsc /openmp" -DCMAKE_EXE_LINKER_FLAGS="" `
 -DGGML_BLAS=ON -DGGML_BLAS_VENDOR=AOCL `
 -DBLAS_LIBRARIES="${AOCL_DIR}\amd-blis\lib\LP64\AOCL-LibBlis-Win-MT.lib;libomp.lib" `
 -DBLAS_INCLUDE_DIRS="${AOCL_DIR}\amd-blis\include\LP64" `
 -DLLAMA_BUILD_COMMON=ON -DLLAMA_BUILD_SERVER=ON -DLLAMA_BUILD_TOOLS=ON `
 -DGGML_CCACHE=OFF -DGGML_VULKAN=OFF -DLLAMA_CURL=OFF -DLLAMA_BUILD_EXAMPLES=OFF `
 -DLLAMA_BUILD_TESTS=OFF -DLLAMA_OPENSSL=OFF -DLLAMA_BUILD_WEBUI=OFF
```

Do the build itself; this will result in the `llama-server` executable:

```ps
cmake --build .\build --config RelWithDebInfo --target llama-server
```

## Download

Store your Huggingface token in a file called `HF_TOKEN.txt` and download the model:

```ps
$headers = @"
Authorization: Bearer $(Get-Content -Path .\HF_TOKEN.txt -Raw)
"@
$model = "unsloth/Qwen3.6-35B-A3B-MTP-GGUF"
$filename = "Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf"
Start-BitsTransfer `
 -Source https://huggingface.co/${model}/resolve/main/${filename} `
 -Destination C:\projects\PlanarLlama\${filename} `
 -CustomHeaders ${headers}
```

Note that the `IQ4` quantifications cannot be used with PlanarQuant.

## Run

Launch the server. Notice that the context size has been reduced from the supported 256k
tokens to 128k tokens, to reduce the memory foot-print.

```ps
.\build\bin\RelWithDebInfo\llama-server.exe `
 --model ".\Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf" `
 --alias "unsloth/Qwen3.6-35B-A3B-MTP-GGUF" `
 --no-webui --no-mmap --ctx-size 131072 --seed 3407 --prio 2 --jinja -ngl 0 `
 --temp 0.6 --top_p 0.95 --top_k 20 --min_p 0.0 --presence_penalty 0.0 --repeat_penalty 1.0 `
 --cache-type-k planar4 --cache-type-v f16 `
 --spec-type ngram-mod,draft-mtp --spec-draft-n-max 3 `
 --parallel 1 `
 --host 127.0.0.1 --port 11434 `
 --api-key hunter2
```

Make a request to the model from another console:

```ps
curl `
  http://127.0.0.1:11434/v1/chat/completions `
  --header "Content-Type: application/json" `
  --header "X-API-Key: hunter2" `
  -d "{ `
  `"model`": `"unsloth/Qwen3.6-35B-A3B-MTP-GGUF`", `
  `"messages`": [`
    { `"role`": `"system`", `"content`": `"Brief professional developer. Only one line explanation.`"}, `
    { `"role`": `"user`", `"content`": `"What is 2 + 2`" }, `
    { `"role`": `"assistent`", `"content`": `"2 + 2 is 4`" }, `
    { `"role`": `"user`", `"content`": `"What is \`"2\`" + \`"2\`"`" } `
  ], `
  `"extra_body`": { `"chat_template_kwargs`": { `
    `"enable_thinking`": true, `
    `"preserve_thinking`": true `
  }}}"
```
