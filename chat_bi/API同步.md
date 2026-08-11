## 1. 全量同步
删除旧数据，重新全量写入。操作最简单，无需 checkpoint。

## 2. 增量同步
### 正常情况（无异常）
根据 updated_at > last_sync_time 拉取新数据，写入后更新 last_sync_time 。

### 异常情况（同步中途崩溃）
需要考虑断点续传。
#### 有 unique key
- checkpoint 记录 ：last_sync_time
- 启动行为 ：从第 1 页开始， updated_at >= last_sync_time
- 写入方式 ： INSERT ... ON DUPLICATE KEY UPDATE （UPSERT）doris自带删除
- 原理 ：重复数据自动覆盖更新，无需清理 
#### 无 unique key
- checkpoint 记录 ： last_page , last_sync_time
- 启动行为 ：从第 1 页开始， updated_at >= last_sync_time
- 写入方式 ：先删除 updated_at >= last_sync_time 的数据，再普通 INSERT
- 原理 ：先清理边界数据，再重新拉取，保证不重复
#### 分页无 unique key
- 每页一个batch_id,同步的数据库也要增加batch_id。结合last_sync_time和updated_at，这个情况可以避免某页同步了，但是checkpoint没保存的场景。

## 3. 总结
Api提供的数据，也要有unique key和updated_at



