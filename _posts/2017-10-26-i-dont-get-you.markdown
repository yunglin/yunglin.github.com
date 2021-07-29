---
layout: post
title: "我搞不懂你啊 Hibernate"
date: 2017-10-26T00:00:00-08:00
---

前幾天才在跟同事講用 ORM 的好處，雖然說我常常被 Hibernate 整到，但是被整多了，還是會知道有那些眉角的，結果話才說完兩天就又被整到。

Hibernate 把我一個很簡單的 JPAQL 改寫成一個建立 temp table 再用 sub query 去查尋，本來好好一個用主鍵去改資料只要 5 msec 的查詢，變成要 30sec 才跑得完，慢了六千倍…

 JPAQL
    
    UPDATE ImportJob
    SET rows = rows + :increase
    WHERE import_job_id = :jobId
    AND completed = 0
    
SQL
    
    insert into HT_import_job
    select importjob0_.import_job_id as import_job_id
    from import_job importjob0_
    where importjob0_.import_job_id=? and importjob0_.completed=0;
    
    update import_job set rows=rows+?
    where (import_job_id) IN (select import_job_id from HT_import_job);

我還是承認我不懂 Hibernate 好了....