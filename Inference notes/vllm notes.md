

this is how vllm will staring serving the model right away 
on the port which u can expose later 


```python
!vllm serve Qwen/Qwen2.5-1.5B-Instruct \
--host 0.0.0.0 \
--port 8000 \
--dtype auto
```
