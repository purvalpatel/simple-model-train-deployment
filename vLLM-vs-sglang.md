vLLM:
----
- High performance inference engine designed to run LLM
- Best for - Model Hosting

### Run meta-llama/Llama-3.1-70B-Instruct Model into vLLM.
1. Accept terms and condition of this model from huggingface and let the maintainers of the model allow to download.
Login huggingface and search model "meta-llama/Llama-3.1-70B-Instruct". <br>

### Fill details and accept. <br>
<img width="914" height="812" alt="image" src="https://github.com/user-attachments/assets/4d8ce8e3-1755-46f9-801d-eb8bfd4fae4d" />

### This will take some time to approve by the maintainers. <br>
You can check this from the `settings -> Gated Repositories` <br>
<img width="1603" height="446" alt="image" src="https://github.com/user-attachments/assets/bcdf22f0-9ded-4483-86e9-dbddda48360a" />

### Generate token:
`settings -> Access Token`
<img width="1170" height="399" alt="image" src="https://github.com/user-attachments/assets/58d4ee67-2d95-4d68-9802-a64a7757fb1e" />

**Note: Save the Access key at safe location.** <br>

2. Run below command to start the container:
````
docker run -d --ipc=host -p 8000:8000   --device=/dev/kfd   --device=/dev/dri -e HF_TOKEN=hf_xxxxxxxxxxxxxxxx  -v hf_cache:/root/.cache/huggingface   rocm/vllm:rocm7.0.0_vllm_0.11.1_20251103   python3 -m vllm.entrypoints.api_server   --model meta-llama/Llama-3.1-70B-Instruct   --tensor-parallel-size 4   --max-model-len 4096   --dtype auto   --gpu-memory-utilization 0.85   --disable-log-requests
````

sglang:
------
- LLM inference engine + Structured reasoning framework
- Best for - Agents, RAG, Structured LLM workdlows.
