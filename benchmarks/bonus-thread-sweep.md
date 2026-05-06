root@12091515:~# /root/llama.cpp/build/bin/llama-bench \
    -m /root/Day20-Track2-ModelServing-Lab/models/tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf -ngl 0
| model                          |       size |     params | backend    | threads |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | ------: | --------------: | -------------------: |
| llama 1B Q4_K - Medium         | 636.18 MiB |     1.10 B | CPU        |       4 |           pp512 |         93.96 ± 3.31 |
| llama 1B Q4_K - Medium         | 636.18 MiB |     1.10 B | CPU        |       4 |           tg128 |         19.29 ± 0.61 |

build: bbeb89d76 (9037)
root@12091515:~# /root/llama.cpp/build/bin/llama-bench \
  -m /root/Day20-Track2-ModelServing-Lab/models/tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf \
  -ngl 0 -t 1,2,4
| model                          |       size |     params | backend    | threads |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | ------: | --------------: | -------------------: |
| llama 1B Q4_K - Medium         | 636.18 MiB |     1.10 B | CPU        |       1 |           pp512 |         26.30 ± 0.72 |
| llama 1B Q4_K - Medium         | 636.18 MiB |     1.10 B | CPU        |       1 |           tg128 |          6.62 ± 0.19 |
| llama 1B Q4_K - Medium         | 636.18 MiB |     1.10 B | CPU        |       2 |           pp512 |         52.83 ± 2.82 |
| llama 1B Q4_K - Medium         | 636.18 MiB |     1.10 B | CPU        |       2 |           tg128 |         12.48 ± 0.81 |
| llama 1B Q4_K - Medium         | 636.18 MiB |     1.10 B | CPU        |       4 |           pp512 |        100.33 ± 4.94 |
| llama 1B Q4_K - Medium         | 636.18 MiB |     1.10 B | CPU        |       4 |           tg128 |         20.65 ± 0.89 |
