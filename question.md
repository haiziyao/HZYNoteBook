# Atlas 私域知识上游快照清单

> 生成时间：2026-10-26  
> 扫描范围：`/Users/bilibili/Developer/Task_List/Atlas/task-02`  
> 快照文件：283 份，去重后 145 张表；其中设备标识被判 `field_role=metric` 的 16 张

## 什么是“上游快照”

导入开始前，从 Atlas 读出的**私域表知识**（`domain=privately-owned`）原样存到批次目录的 `ingest_req/*.json`。
它是那一刻的副本，不代表现在的库内状态；后续链路是 `private.json → source_dump.json → payload_entities.json → 导入`。

## 设备标识被判 metric 的 16 张表

| # | 表名 | dtAtomId | 字段数 | 快照份数 | 首份快照 | 误判字段 |
|---|---|---|---|---|---|---|
| 1 | `b_ads.ads_growth_actvy_stgy_lahuo_novisit_postback_i_d` | 2111288656949776385 | 72 | 1 | 2026-10-23 15:50 | oaid |
| 2 | `b_dm_growth.dwb_awake_lahuo_h5_event_attr_l_d` | 2111289255338545153 | 68 | 1 | 2026-10-23 16:48 | idfa, oaid |
| 3 | `b_dm_growth.dwd_cmpn_rta_callback_growth_attr_event_postback_log_l_hr` | 2111278109814525954 | 56 | 1 | 2026-10-23 16:21 | idfa, oaid |
| 4 | `b_dm_growth.dwd_flow_app_awake_deeplink_lahuo_detail_i_d` | 2111278743307292673 | 57 | 6 | 2026-10-21 15:27 | oaid |
| 5 | `b_dwb.dwb_flow_app_growth_actvy_strategy_lahuo_buvid_l_d` | 2111156871248875522 | 47 | 1 | 2026-10-23 16:40 | idfa, oaid |
| 6 | `b_dwd.dwd_cmpn_tunl_xman_ad_clicks_bak_l_hr` | 2111251492052504577 | 44 | 1 | 2026-10-23 17:03 | idfa |
| 7 | `b_dwm.dm_growth_dwb_dsp_action_detail_l_d` | 2111295144757923841 | 56 | 1 | 2026-10-23 17:13 | idfa, oaid |
| 8 | `b_ods.rods_s_growth_rta_click_iceberg_l_hr` | 2111102428426113025 | 45 | 5 | 2026-10-21 16:46 | idfa, oaid |
| 9 | `b_r_dwd.dwd_flow_main_visit_detail_buvid_rt_hr` | 2111122021349691394 | 28 | 5 | 2026-10-21 16:47 | idfa, oaid |
| 10 | `b_r_dwd.dwd_flow_new_device_rt_hr` | 2111300507888619522 | 29 | 1 | 2026-10-23 16:36 | idfa, oaid |
| 11 | `b_r_dwd.dwd_visit_dau_analysis_i_d` | 2111263987957862401 | 19 | 1 | 2026-10-23 16:38 | oaid |
| 12 | `bili_bdp.dwb_tfc_app_activate_a_d` | 2111125422233980930 | 31 | 1 | 2026-10-23 16:51 | idfa, oaid |
| 13 | `bili_dwb.dwb_growth_click_attr_postback_i_d` | 2111174248282034177 | 81 | 1 | 2026-10-23 17:09 | oaid |
| 14 | `bili_dwd.dwd_motl_newuser_daily_inc_d` | 2111052347056427010 | 77 | 1 | 2026-10-23 15:58 | idfa, oaid |
| 15 | `bili_dwd.dwd_tfc_app_buvid_d` | 2111259465915543553 | 27 | 1 | 2026-10-23 16:10 | idfa, oaid |
| 16 | `bili_dwd.dwd_tfc_app_click_d` | 2111104280031375362 | 53 | 1 | 2026-10-23 16:21 | idfa |
