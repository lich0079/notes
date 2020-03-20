# Transactional messaging

## 先发 half message

 * 防止本地DB操作成功， 但消息系统失败的情况
    * Y 公司的自研MQ处理这个问题的方法是，消息系统发消息就是插入表记录，而某个topic所属的表和本地操作的表在同一个DB，可以用本地事物来保证一致性，主业务DB操作和消息写入一定是原子的

 * 如果half发送失败，本地后续DB操作不执行，前序操作回滚


## half 成功发送之后

 * 执行本地事物
    * 执行失败， 让 MQ rollback half msg
    * 执行成功，让 MQ commit msg
    * 上面2种情况，如果对MQ 发出的rollback/commit 它都没收到，MQ会有定时任务来回调询问主系统该rollback还是commit。

 * MQ half 保存成功，但没能通知到消息发送方
    * 本地会以为half发送失败，本地rollback
    * 过一回MQ会来回调查询本地系统，然后rollback half