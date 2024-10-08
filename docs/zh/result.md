
# 结果集说明


goInception给用户返回的信息有两种，
* 一种是提交给goInception的基础信息存在错误，比如源信息不全，或者源信息有错误等，这种情况下，直接报异常，包括错误码及错误信息，与MySQL服务器的异常是一样的，在外面正常处理即可。
* 二是如果没有上面的问题，都会以结果集的方式将检查结果告诉客户端。和mysql原生结果集一致。返回的结果集中，每一个行数据，就是一条提交的SQL语句，goInception内部将所有提交的语句块一条条的拆开，以结果集的方式返回，针对每一条语句，有什么问题或者状态，在结果集中是一目了然。


**注意：** 如果在语句中出现语法错误，则不能继续了，因为goInception已经不能将剩下的语句分开了，那么此时前面已经正常检查的多行为多个结果集的行返回，后面出错的语句为一行返回，当然这个的错误信息是语法错误。

goInception返回结果集的结构如下：

1. `order_id` sql序号，从1开始。
1. `stage` 当前语句已经进行到的阶段，包括CHECKED、EXECUTED、RERUN、NONE，
	- NONE表示没有做过任何处理，有可能前面有语法错误直接就提前返回了,
	- CHECKED表示这个语句只做过审核，而没有再进行下一步操作，
	- EXECUTED表示已经执行过，如果执行失败，也是用这个状态表示，
	- RERUN表示的是，对于影响上下文的语句，已经执行成功，但为了与EXECUTED区分，用RERUN表示，主要是因为在执行过程中，如果某一条语句执行失败了，则上层可能需要将没有执行的语句提取出来，再次执行，那么影响上下文的语句是需要加上的，所以用RERUN来表示。影响上下文的语句一般包括set names和use db这两种，而当前Inception支持的只有这两种。
1. `error_level` 错误级别。返回值为非0的情况下，说明是有错的。0表示审核通过。1表示警告，不影响执行，2表示严重错误，必须修改
1. `stage_status` 阶段状态，用来表示检查及执行的过程是成功还是失败，
	- 如果审核成功，则返回 Audit completed。
	- 如果执行成功则返回Execute Successfully，否则返回Execute failed，
	- 如果备份成功，则在后面追加Backup successfully，否则追加Backup failed，
	- 这个列的返回信息是为了将结果集直接输出而设置的，如果在具体使用过程中，为了更友好的显示，可以在这基础上再做加工处理。
1. `error_message` 错误信息。用来表示出错错误信息，这里包括一条语句中所有的错误信息，用换行符分隔，但有时候如果某一个错误导致不能继续分析了，则后面的错误就不能显示出来。如果没有出错，则返回NULL。而对于执行及备份错误，因为对于一条语句，这样的错误只会有一次，那么执行错误会在后面追加“execute:具体的执行错误原因”，如果是备份出错，则在后面追加“backup:具体的备份错误原因”。如果设置参数alter_auto_merge=true，则合并后的SQL行该字段会显示“MERGED”。
1. `sql` 用来表示当前检查的是哪条sql语句
1. `affected_rows` 执行时预计影响行数，在执行时显示的是真实影响行数。
1. `sequence` 这个列与上面说的备份功能有关系，其实就是对应**$$Inception_backup_information$$**表中的 opid_time 这个列，一一对应，这就为前端应用在针对某一操作回滚找到了入口，每次执行都会产生一个序号，如果要回滚，则就使用这个值从备份表中找到对应的回滚语句执行即可。详见[备份功能](backup.html)
1. `backup_dbname` 这个列表示的是当前语句产生的备份信息，存储在备份服务器的哪个数据库中，这是一个字符串类型的值，只针对需要备份的语句，数据库名由IP地址、端口、源数据库名组成，由下划线连接。详见[备份功能](backup.html)
1. `execute_time` 这个列表示当前语句执行时间，单位为秒，精确到小数点后两位。列类型为字符串，使用时可能需要转换成DOUBLE类型的值，如果只是审核而不执行，则这个列返回的值为0。
1. `sqlsha1` 这个列用来存储当前这个语句的一个HASH值，这是用来标识这个语句是不是会使用OSC功能，如果返回信息中有值，则表示这个语句在执行的时候会使用OSC，因为在执行前，会有一次单独的审核操作，此时上层已经可以拿到这个值，审核通过之后，语句是不会改变的，当然这个值也不会改变，那么在执行时就可以使用这个值来查看OSC执行的进度等信息，这个值一般长的样子如下：*D0210DFF35F0BC0A7C95CD98F5BCD4D9B0CA8154，具体其它信息，请参考 [DDL变更:pt-osc](osc.html)和 [DDL变更:gh-ost](ghost.html)
1. `backup_time` 生成当前SQL的备份语句耗时。
1. `need_merge` 当前SQL是否可以和其他SQL合并成一条。0代表不可以；-1代表当前SQL是已经合并过的；其他大于0的数字代表可以合并，具体数值和已合并SQL行（即当前字段值为-1的行）的order_id值一样。
