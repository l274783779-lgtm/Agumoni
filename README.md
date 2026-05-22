量化回测项目说明

更新时间：2026-05-22 13:57:22

本文件已做一次整理：
- 合并重复的 baseline_v1_5 固定记录。
- 合并 v1.6 终端打印恢复、窗口独立排名、汇总后置、进度日志节流等过程记录。
- 合并 v1.7 的 60分钟退出、缓存优化、参数等价去重记录。
- 删除已经被后续版本覆盖的临时计划和重复说明。
- 原始 read_me 已备份为 logs/readme_backups/read_me_backup_before_cleanup_20260522_135722.rtf。


一、当前核心策略口径

当前主线仍然是“成交额放大 + 换手率放大 + 相对市场放大”的日线买入信号。

买入信号：
1. 过去20日平均成交额 > 5000万
2. 今日成交额 / 过去20日平均成交额 > 1.5
3. 个股成交额放大倍数 / 全市场成交额放大倍数 > 1.2
4. 今日换手率 / 过去20日平均换手率 > 1.5
5. 个股换手率放大倍数 / 全市场换手率放大倍数 > 1.2

排序规则：
- total_score = 0.6 * amount_rank_score + 0.4 * turnover_rank_score

交易规则：
1. 信号日收盘后产生信号。
2. 下一交易日开盘买入。
3. 买入股数必须是 100 股整数倍。
4. 每天最多买入 MAX_STOCKS_PER_DAY 只股票。
5. 买入当天不能卖出，遵守 A 股 T+1。
6. 从买入后第 2 个交易日开始检查止盈 / 止损。
7. 达到最长持有天数后，按当前版本规则强制卖出。
8. 交易成本包括佣金、最低佣金、印花税、过户费、滑点参数。


二、主要数据文件

data/panel/daily_panel_mainboard.parquet
- 沪深主板日线面板。
- 当前 v1.6 / v1.7 用于日线开盘价、收盘价和交易日历。

data/signals/signal_panel_1_5.parquet
- v1.5 买入信号面板。
- 当前 v1.6 / v1.7 继续沿用，不在 sweep 中重新生成买入信号。

data/panel_60min/stock_60min_panel.parquet
- 股票 60 分钟线总表。
- v1.7 用于买入后的止盈、止损、到期强制卖出判断。


三、当前保留版本

1. v1.5 baseline

文件：
- backtest_filter_hold1_5_baseline.py
- backtest_filter_hold1_5_baseline_core.py

定位：
- 固定当前 baseline，方便和后续版本对比。
- 不新增大盘过滤、板块过滤、二次探底形态或其他买入规则。

输出：
- results/baseline_v1_5/


2. v1.6 日线高速扫描版

文件：
- backtest_filter_hold1_6_core.py
- sweep_params_1_6.py

定位：
- 继续使用 daily_panel_mainboard.parquet 和 signal_panel_1_5.parquet。
- 买入、卖出、仓位、T+1、递进止损、到期卖出逻辑沿用当前日线版本。
- 启用真实交易成本口径。
- 输出 summary、yearly_metrics、Top10、hold_days_summary、diagnostic_summary 等解释性统计。

当前扫描设置：
- 正式参数组合：8100 组。
- 参数等价去重后实际有效组合：1620 组。
- 减少重复回测：6480 组，理论减少 80.00%。
- 资金规模和窗口由 sweep_params_1_6.py 顶部配置控制。

参数等价去重字段：
- effective_target_pct
- effective_daily_pct
- canonical_param_id
- is_reused_result

输出目录：
- data/backtest/filter_hold1_6/


3. v1.7 60分钟退出版

文件：
- backtest_filter_hold1_7_core_60min_exit.py
- sweep_params_1_7_60min_exit.py
- compare_v1_6_vs_v1_7.py

定位：
- 买入信号仍然使用日线 signal_panel_1_5.parquet。
- 买入价格仍然使用 buy_date 的日线 open。
- 买入后第 2 个交易日开始，用股票 60 分钟线逐根检查止盈 / 止损。
- 买入当天不能卖出。
- 同一根 60分钟 K 线同时触发止盈和止损时，保守按止损处理。
- 到期强制卖出使用 forced_sell_date 第一根 60分钟 K 线 open。
- 如果 forced_sell_date 缺少 60分钟线，则 fallback 到日线 open，再 fallback 到 last_price。
- 交易成本沿用 v1.6 口径。

当前性能优化：
- 已按 HOLD_DAYS 缓存 signals_by_buy_date，避免每组参数重复准备信号。
- 已将 60分钟线整理为 stock_date_60min_map，避免核心回测中反复筛 DataFrame。
- 已迁移 v1.6 参数等价去重。

v1.7 参数等价去重字段：
- effective_daily_position_pct
- effective_target_stock_position_pct
- effective_param_key
- effective_run_tag
- is_dedup_cache_hit

当前去重效果：
- TEST_MODE=True：80 个原始组合，60 个 effective 组合，cache hit 20 行。
- 正式模式：8100 个原始组合，1620 个 effective 组合，减少 6480 次重复回测。

输出目录：
- data/backtest/filter_hold1_7_60min_exit/

主要输出：
- sweep_summary.csv
- sweep_summary_unique_effective.csv
- sweep_summary_sorted_by_return.csv
- sweep_summary_sorted_by_return_drawdown.csv
- top10_by_return_chinese.csv
- top10_by_return_drawdown_chinese.csv


4. v1.7.1 60分钟退出 + 简化停牌处理版

新增文件：
- backtest_filter_hold1_7_1_core_60min_exit_suspension.py
- sweep_params_1_7_1_60min_exit_suspension.py

定位：
- 基于 v1.7 新增简化停牌处理，不覆盖 v1.7 文件和结果。
- 不改变买入信号、排序规则、仓位规则、止盈止损阈值和交易成本口径。
- 买入日不可交易或停牌时，直接放弃该信号，不顺延买入。
- 持仓期间不可交易或停牌时，不检查止盈止损，不卖出，不增加有效持有日，仓位继续占用。
- HOLD_DAYS 从 v1.7.1 开始按“可交易检查日”累计。
- 日线可交易但缺少 60分钟线时，才计入 missing_60min_bar_count。
- 每日买入逻辑改为“候选池补位”：每个 buy_date 先保留 Top30 候选，回测中从高分到低分尝试买入。
- 遇到停牌、不可交易、无有效价格、资金不足或买不起 100 股时跳过，继续尝试后续候选。
- MAX_STOCKS_PER_DAY 仍然是每日实际成功买入上限；BUY_CANDIDATE_POOL_SIZE 只是候选池缓冲数量。
- 该修改不改变买入信号生成规则，只改变买入执行阶段对不可买候选的补位处理。

新增诊断字段：
- buy_skipped_by_suspension_count
- suspension_holding_day_count
- max_hold_paused_by_suspension_count
- long_suspension_unresolved_count
- buy_skipped_by_untradable_count
- buy_skipped_by_cash_count
- buy_skipped_by_lot_size_count
- buy_candidate_checked_count
- buy_candidate_success_count
- buy_candidate_pool_size

输出目录：
- data/backtest/filter_hold1_7_1_60min_exit_suspension/

主要输出：
- sweep_summary.csv
- sweep_summary_unique_effective.csv
- sweep_summary_sorted_by_return.csv
- sweep_summary_sorted_by_return_drawdown.csv
- top10_by_return_chinese.csv
- top10_by_return_drawdown_chinese.csv
- missing_60min_detail.csv
- suspension_diagnostic.csv
- buy_candidate_diagnostic.csv

TEST_MODE 检查记录：
- 2023-01-01 到 2026-05-19，80 个原始组合，60 个 effective 组合，已跑通。
- sh.600157 不再进入普通 60分钟缺失；当前日线数据中 2024-07-25 已为 trade_status=0，因此按买入日停牌规则归类为 buy_skipped。
- 候选池补位测试已跑通：Top30 候选池、每日成功买入上限 5，只出现第 6 名以后补位买入，不会超过每日买入上限。


四、60分钟数据处理脚本

download_all_stocks_60min_baostock.py
- 全量下载沪深主板股票 60 分钟线。
- 输出 data/raw_60min/stocks/ 和下载日志。

clean_stocks_60min_baostock.py
- 清洗股票 60 分钟线原始数据。
- 输出 data/processed_60min/stocks/ 和清洗日志。

build_stock_60min_panel.py
- 合并清洗后的 60 分钟线，生成统一总表。
- 输出 data/panel_60min/stock_60min_panel.parquet。

check_stock_60min_panel.py
- 抽查 60 分钟线总表是否正常，并输出检查日志和抽样图。


五、已合并或清理的旧记录

以下内容不再单独保留长篇记录，已合并到上面的当前版本说明：
1. 两条重复的 baseline_v1_5 固定记录。
2. v1.6 终端摘要打印恢复。
3. v1.6 终端窗口独立排名调整。
4. v1.6 终端汇总统一后置。
5. v1.6 progress log throttling。
6. v1.7 60min exit test record。
7. v1.7 sweep speed optimization record。
8. v1.7 effective parameter dedup record。

后续如果要新增大盘过滤、板块过滤、二次探底形态或其他买入规则，请复制当前稳定版本另建新文件，不要直接覆盖 v1.5 baseline、v1.6 或 v1.7 当前文件。
