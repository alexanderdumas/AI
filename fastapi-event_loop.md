# FastAPI 优雅运行同步阻塞函数
**概述:** 适用于高并发 I/O 密集型场景。本文通过封装全局线程池 ThreadPoolExecutor 与 run_in_executor 桥接异步调度，实现同步函数的非阻塞执行与生命周期优雅管理。


## 全局线程池设计
```python
# threadpool.py
import asyncio
import concurrent.futures

# 全局线程池
global_executor: concurrent.futures.Executor = None

def initialize_threadpool(max_workers=100):
    global global_executor
    if global_executor is None:
        global_executor = concurrent.futures.ThreadPoolExecutor(max_workers=max_workers)

async def run_in_threadpool(func, *args):
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(global_executor, func, *args)


def shutdown_threadpool():
    if global_executor:
        global_executor.shutdown()
```

## 服务初始化
```python
import time
import traceback
import uvicorn
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from starlette.responses import JSONResponse
# 从threadpool.py import
from threadpool import initialize_threadpool, run_in_threadpool, shutdown_threadpool

# 创建FastAPI实例
app = FastAPI()

# 初始化线程池
initialize_threadpool()

def test_sync():# 同步函数
    time.sleep(5)
    return "success"

@app.post('/v1/check_async') # 异步方法
async def check_async(request: Request):
    content = {}
    try:
        result = await run_in_threadpool(test_sync)
        content['code'] = 200
        content['message'] = result
        return JSONResponse(content=content, status_code=200)
    except Exception as e:
        tb = traceback.format_exc(limit=5)
        print("exception:", e, tb)
        content['code'] = 500
        content['message'] = "error"
        return JSONResponse(content=content, status_code=500)

if __name__ == '__main__':
    try:
        uvicorn.run(app='main:app', host="0.0.0.0", port=65502, reload=False, workers=4) # workers=4 表示启动4个进程
    finally:
        shutdown_threadpool()
```
