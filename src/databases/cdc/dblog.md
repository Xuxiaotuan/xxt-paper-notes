# dblog

- (1) 记录当前binlog位置作为`LOW`偏移量
- (2) 通过执行语句读取并缓冲快照块记录`SELECT * FROM MyTable WHERE id > chunk_low AND id <= chunk_high`
- (3) 记录当前binlog位置作为`HIGH`偏移量
- (4)`LOW` 从offset到`HIGH`offset读取属于snapshot chunk的binlog记录
- (5) 将读到的binlog记录Upsert到缓冲的chunk记录中，并将缓冲区中的所有记录作为快照chunk的最终输出（全部作为INSERT记录）发出
- (6) 在single binlog reader`HIGH`中继续读取并发出属于offset之后的chunk的binlog记录。
