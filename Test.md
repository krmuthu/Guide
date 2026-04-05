https://huggingface.co/nomic-ai/CodeRankEmbed/tree/main









https://huggingface.co/unsloth/Qwen3-235B-A22B-Instruct-2507-GGUF/tree/main/UD-Q4_K_XL


https://huggingface.co/unsloth/Qwen3-235B-A22B-Instruct-2507-GGUF/resolve/main/UD-Q4_K_XL/Qwen3-235B-A22B-Instruct-2507-UD-Q4_K_XL-00001-of-00003.gguf?download=true

https://huggingface.co/unsloth/Qwen3-235B-A22B-Instruct-2507-GGUF/resolve/main/UD-Q4_K_XL/Qwen3-235B-A22B-Instruct-2507-UD-Q4_K_XL-00002-of-00003.gguf?download=true

https://huggingface.co/unsloth/Qwen3-235B-A22B-Instruct-2507-GGUF/resolve/main/UD-Q4_K_XL/Qwen3-235B-A22B-Instruct-2507-UD-Q4_K_XL-00003-of-00003.gguf?download=true


$proxy = "http://your-nexus-proxy:8443"
$base = "https://huggingface.co/unsloth/Qwen3-235B-A22B-Instruct-2507-GGUF/resolve/main/UD-Q4_K_XL"
$dir = "C:\Models\qwen3-235b-2507\UD-Q4_K_XL"
mkdir -Force $dir
cd $dir

# File 1
curl -L -C - --retry 20 --retry-delay 30 --connect-timeout 120 --max-time 0 `
  -x $proxy `
  -o "Qwen3-235B-A22B-Instruct-2507-UD-Q4_K_XL-00001-of-00003.gguf" `
  "$base/Qwen3-235B-A22B-Instruct-2507-UD-Q4_K_XL-00001-of-00003.gguf"

# File 2
curl -L -C - --retry 20 --retry-delay 30 --connect-timeout 120 --max-time 0 `
  -x $proxy `
  -o "Qwen3-235B-A22B-Instruct-2507-UD-Q4_K_XL-00002-of-00003.gguf" `
  "$base/Qwen3-235B-A22B-Instruct-2507-UD-Q4_K_XL-00002-of-00003.gguf"

# File 3
curl -L -C - --retry 20 --retry-delay 30 --connect-timeout 120 --max-time 0 `
  -x $proxy `
  -o "Qwen3-235B-A22B-Instruct-2507-UD-Q4_K_XL-00003-of-00003.gguf" `
  "$base/Qwen3-235B-A22B-Instruct-2507-UD-Q4_K_XL-00003-of-00003.gguf"

https://huggingface.co/unsloth/Qwen3.5-122B-A10B-GGUF/resolve/main/UD-Q4_K_XL/Qwen3.5-122B-A10B-UD-Q4_K_XL-00001-of-00003.gguf?download=true
https://huggingface.co/unsloth/Qwen3.5-122B-A10B-GGUF/resolve/main/UD-Q4_K_XL/Qwen3.5-122B-A10B-UD-Q4_K_XL-00002-of-00003.gguf?download=true

https://huggingface.co/unsloth/Qwen3.5-122B-A10B-GGUF/resolve/main/UD-Q4_K_XL/Qwen3.5-122B-A10B-UD-Q4_K_XL-00003-of-00003.gguf?download=true
