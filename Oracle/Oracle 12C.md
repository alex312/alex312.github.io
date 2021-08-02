# 删除diag目录下大量的trc文件
Oracle tarce文件是oracle数据库在运行时产生的日志，该trace文件是可以删除的，对系统没有什么影响。

在删除前，先查看trace的参数配置

```
SQL> show parameter trace_en
NAME                                TYPE        VALUE
------------------------------------ ----------- ------------------------------
trace_enabled                        boolean    TRUE
SQL>
```

通过find命令查找30天以前创建的文件，通过xargs管道和rm-rf命令，可以直接删除
```
$ find trace -ctime +30 | more
trace/hbcadb1_j000_25349.trc
trace/hbcadb1_j000_28514.trm
trace/hbcadb1_ora_12171.trm
trace/hbcadb1_ora_4595.trm
trace/hbcadb1_j000_3029.trm
trace/hbcadb1_ora_14967.trm
trace/hbcadb1_j000_2237.trm
trace/hbcadb1_j000_13278.trc
trace/hbcadb1_ora_13051.trm

$ find trace -ctime +30 |xargs rm -f
```

## 青岛现场维护命令

```
$ cd /appl/oracle/app/diag/rdbms/qdl8sc/qdl8sc
$ find trace -ctime +30 | xargs rm -rf
```