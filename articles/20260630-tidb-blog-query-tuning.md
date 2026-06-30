---
title: "もくもくTiDBチューニング(ぱ〜と1)"
type: "tech"
category: []
description: ""
publish: false
---

## はじめに

このブログのトップページで実行している記事一覧APIで呼ばれている記事一覧クエリについてチューニングしてみる。

## 試してみる

TiDBにはTiDB Dashboardというものがついておりなかなかイケている。

```sql
SELECT
  `a`.`article_id`,
  `a`.`title`,
  `a`.`slug`,
  `a`.`user_id`,
  `a`.`content`,
  `a`.`thumbnail`,
  `a`.`description`,
  `a`.`status`,
  `a`.`type`,
  `a`.`published_at`,
  `a`.`created_at`,
  `a`.`updated_at`
FROM
  `articles` `a`
  JOIN `users` `u` ON `a`.`user_id` = `u`.`user_id`
WHERE
  `a`.`status` = ?
  AND `a`.`type` = ?
  AND `u`.`name` = ?
ORDER BY
  `a`.`published_at` DESC
```


```md

| id                             | estRows | estCost   | actRows | task      | access object                                | execution info                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | operator info                                                                                                                                                             | memory   | disk     |
| Sort_10                        | 41.76   | 41266.11  | 40      | root      |                                              | time:7.93ms, loops:2                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | blog_prd.articles.published_at:desc                                                                                                                                       | 398.1 KB | 0 Bytes  |
| └─IndexHashJoin_20             | 41.76   | 12322.78  | 40      | root      |                                              | time:7.7ms, loops:2, inner:{total:6.95ms, concurrency:5, task:1, construct:3.64µs, fetch:6.76ms, build:9.6µs, join:178.5µs}                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | inner join, inner:IndexLookUp_17, outer key:blog_prd.users.user_id, inner key:blog_prd.articles.user_id, equal cond:eq(blog_prd.users.user_id, blog_prd.articles.user_id) | 572.0 KB | N/A      |
|   ├─Point_Get_29(Build)        | 1.00    | 242.91    | 1       | root      | table:users, index:uq_users_name(name)       | time:689.8µs, loops:3, Get:{num_rpc:2, total_time:634.8µs}, time_detail: {total_process_time: 62.8µs, total_wait_time: 57.9µs, total_kv_read_wall_time: 125.8µs, tikv_wall_time: 164.5µs}, scan_detail: {total_process_keys: 2, total_process_keys_size: 250, total_keys: 2, get_snapshot_time: 14.6µs, rocksdb: {block: {cache_hit_count: 2}}}                                                                                                                                                                                                                                                                           |                                                                                                                                                                           | N/A      | N/A      |
|   └─IndexLookUp_17(Probe)      | 41.76   | 302143.06 | 40      | root      |                                              | time:6.7ms, loops:2, index_task: {total_time: 932µs, fetch_handle: 930µs, build: 421ns, wait: 1.55µs}, table_task: {total_time: 5.56ms, num: 1, concurrency: 5}, next: {wait_index: 982µs, wait_table_lookup_build: 63.7µs, wait_table_lookup_resp: 5.5ms}                                                                                                                                                                                                                                                                                                                                                                |                                                                                                                                                                           | 486.0 KB | N/A      |
|     ├─IndexRangeScan_14(Build) | 130.00  | 34115.89  | 130     | cop[tikv] | table:a, index:idx_articles_user_id(user_id) | time:893.9µs, loops:3, cop_task: {num: 1, max: 856.4µs, proc_keys: 130, tot_proc: 271µs, tot_wait: 42µs, copr_cache_hit_ratio: 0.00, build_task_duration: 12.2µs, max_distsql_concurrency: 1}, rpc_info:{Cop:{num_rpc:1, total_time:849.6µs}}, tikv_task:{time:0s, loops:3}, scan_detail: {total_process_keys: 130, total_process_keys_size: 16770, total_keys: 131, get_snapshot_time: 10.8µs, rocksdb: {key_skipped_count: 130, block: {cache_hit_count: 1}}}, time_detail: {total_process_time: 271µs, total_wait_time: 42µs, tikv_wall_time: 420.9µs}                                                                 | range: decided by [eq(blog_prd.articles.user_id, blog_prd.users.user_id)], keep order:false                                                                               | N/A      | N/A      |
|     └─Selection_16(Probe)      | 41.76   | 80056.18  | 40      | cop[tikv] |                                              | time:5.48ms, loops:2, cop_task: {num: 1, max: 5.35ms, proc_keys: 130, tot_proc: 2.54ms, tot_wait: 25.2µs, copr_cache_hit_ratio: 0.00, build_task_duration: 17.6µs, max_distsql_concurrency: 1}, rpc_info:{Cop:{num_rpc:1, total_time:5.35ms}}, tikv_task:{time:2ms, loops:3}, scan_detail: {total_process_keys: 130, total_process_keys_size: 853780, total_keys: 260, get_snapshot_time: 7.28µs, rocksdb: {key_skipped_count: 130, block: {cache_hit_count: 257}}}, time_detail: {total_process_time: 2.54ms, total_suspend_time: 11.1µs, total_wait_time: 25.2µs, total_kv_read_wall_time: 2ms, tikv_wall_time: 2.76ms} | eq(blog_prd.articles.status, "published"), eq(blog_prd.articles.type, "tech")                                                                                             | N/A      | N/A      |
|       └─TableRowIDScan_15      | 130.00  | 67082.18  | 130     | cop[tikv] | table:a                                      | tikv_task:{time:2ms, loops:3}                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | keep order:false                                                                                                                                                          | N/A      | N/A      |
```


## 気をつけること

* TiDB DashboardのSQL Statementsで確認したクエリ、実行計画をAIで解析させて診断ドキュメントを作る場合、そのStatementsのURLを紐付けておく
  * これをしないとサンプリングしたデータ期間がわからなくなったり意思決定の材料となった記録がなくなるので注意


## メモ


* TiDBダッシュボードのURL(個人メモ)
  * dashboard/#/statement/detail?query=%7B%22digest%22%3A%22f62ed22ccb8705c71f30fdfa02792baa585b57ea5bb3bc5bf3aae5a3ba254ab1%22%2C%22schema%22%3A%22blog_prd%22%2C%22beginTime%22%3A1782708749%2C%22endTime%22%3A1782795150%7D

