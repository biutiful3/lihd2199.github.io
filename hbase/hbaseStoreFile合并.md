Mem Store每次flush都会生成一个新的HFile，同一个字段的不同版本（TimeStamp），不同类型(Put/Delete)的数据，可能分布在不同的HFile中。因此查询时就要遍历所有的HFile。为了保证查询效率，要定期清理过期和删除的数据。

- Minor Compaction会将临近的若干个较小的 HFile 合并成一个较大的 HFile，但不会清理过期和删除的数据。
- Major Compaction会将一个 Store下的所有的 HFile 合并成一个大 HFile，并且会清理掉过期和删除的数据。

