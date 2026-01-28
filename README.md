# PostgreSQL Internals Course

A comprehensive deep-dive into PostgreSQL's internal architecture, from SQL parsing to disk I/O.

## 🎯 What You'll Learn

This course takes you through PostgreSQL's source code and internal mechanisms:

- **Query Processing Pipeline**: Parser → Analyzer → Rewriter → Planner → Executor
- **Storage Engine**: Heap storage, page layout, TOAST, tablespaces
- **Index Internals**: B-tree, GIN, GiST, BRIN, Hash indexes
- **Memory Management**: Buffer pool, shared memory, work_mem
- **MVCC & Concurrency**: Transaction isolation, visibility, locking
- **WAL & Durability**: Write-ahead logging, checkpoints, recovery
- **VACUUM & Maintenance**: Dead tuple cleanup, XID wraparound prevention
- **Query Optimization**: Join algorithms, parallel execution, EXPLAIN analysis

## 📚 Course Structure

### Module 1: Foundation — From SQL to Execution
| Lesson | Topic |
|--------|-------|
| 1 | [How PostgreSQL Processes a Query](lessons/module_01_foundation/lesson_01_query_journey.md) |
| 2 | [The Parser — Transforming SQL Into Internal Structures](lessons/module_01_foundation/lesson_02_parser.md) |
| 3 | [The Analyzer — Semantic Analysis and Name Resolution](lessons/module_01_foundation/lesson_03_analyzer.md) |
| 4 | [The Rewriter — Query Transformation and View Expansion](lessons/module_01_foundation/lesson_04_rewriter.md) |
| 5 | [Introduction to the Query Planner](lessons/module_01_foundation/lesson_05_planner_intro.md) |
| 6 | [The Executor — Running the Query Plan](lessons/module_01_foundation/lesson_06_executor.md) |
| 7 | [Understanding EXPLAIN in Depth](lessons/module_01_foundation/lesson_07_explain_deep_dive.md) |

### Module 2: Storage Architecture
| Lesson | Topic |
|--------|-------|
| 8 | [Heap Storage and Page Layout](lessons/module_02_storage/lesson_08_heap_storage.md) |
| 9 | [B-tree Index Structure](lessons/module_02_storage/lesson_09_btree_indexes.md) |
| 10 | [Tablespaces and Relation Files](lessons/module_02_storage/lesson_10_tablespaces_files.md) |

### Module 3: Memory Management
| Lesson | Topic |
|--------|-------|
| 11 | [The Buffer Manager](lessons/module_03_memory/lesson_11_buffer_manager.md) |
| 12 | [Work Memory — Sorting, Hashing, and Per-Operation Memory](lessons/module_03_memory/lesson_12_work_mem.md) |

### Module 4: Write-Ahead Logging
| Lesson | Topic |
|--------|-------|
| 14 | [Write-Ahead Logging (WAL) — Durability and Recovery](lessons/module_04_wal/lesson_14_wal_fundamentals.md) |

### Module 5: MVCC and Visibility
| Lesson | Topic |
|--------|-------|
| 13 | [MVCC Fundamentals — Multi-Version Concurrency Control](lessons/module_05_mvcc/lesson_13_mvcc_fundamentals.md) |
| 22 | [Transaction ID Wraparound — Preventing a Silent Disaster](lessons/module_05_mvcc/lesson_22_xid_wraparound.md) |

### Module 6: Transactions and Locking
| Lesson | Topic |
|--------|-------|
| 16 | [Transaction Isolation Levels](lessons/module_06_transactions/lesson_16_isolation_levels.md) |
| 17 | [Locking and Concurrency Control](lessons/module_06_transactions/lesson_17_locking.md) |

### Module 7: VACUUM and Maintenance
| Lesson | Topic |
|--------|-------|
| 15 | [VACUUM — Dead Tuple Cleanup and Space Reclamation](lessons/module_07_vacuum/lesson_15_vacuum_fundamentals.md) |

### Module 8: Index Internals
| Lesson | Topic |
|--------|-------|
| 19 | [GIN, GiST, and Specialized Indexes](lessons/module_08_indexes/lesson_19_specialized_indexes.md) |

### Module 10: Join Optimization
| Lesson | Topic |
|--------|-------|
| 18 | [Join Algorithms — Nested Loop, Hash, and Merge](lessons/module_10_joins/lesson_18_join_algorithms.md) |

### Module 11: Query Execution Optimization
| Lesson | Topic |
|--------|-------|
| 20 | [Parallel Query Execution](lessons/module_11_optimization/lesson_20_parallel_query.md) |

### Module 12: PostgreSQL Architecture
| Lesson | Topic |
|--------|-------|
| 21 | [Background Processes](lessons/module_12_architecture/lesson_21_background_processes.md) |

## 🛠️ Prerequisites

- Basic SQL knowledge
- Familiarity with PostgreSQL usage
- Some C programming exposure (helpful but not required)
- A PostgreSQL installation for hands-on exercises

## 🧪 Hands-On Approach

Each lesson includes:
- **Conceptual explanations** with ASCII diagrams
- **Source code references** to PostgreSQL internals
- **Practical examples** you can run
- **Hands-on exercises** to reinforce learning
- **Key takeaways** summarizing critical concepts

## 📖 How to Use This Course

1. **Sequential Learning**: Follow lessons in order for the best understanding
2. **Reference Material**: Jump to specific topics as needed
3. **Hands-On Practice**: Run the exercises on your own PostgreSQL instance
4. **Source Exploration**: Use the referenced source files to dig deeper

## 🔧 Setting Up for Exercises

```bash
# Install PostgreSQL (if needed)
# macOS
brew install postgresql@16

# Ubuntu/Debian
sudo apt install postgresql-16

# Create a test database
createdb postgres_internals_lab
```

Many exercises use the `pageinspect` and `pgstattuple` extensions:

```sql
CREATE EXTENSION pageinspect;
CREATE EXTENSION pgstattuple;
```

## 📊 Course Statistics

- **22 Lessons** covering core PostgreSQL internals
- **12 Modules** organized by topic
- **100+ Hands-on exercises**
- **50+ ASCII diagrams** explaining internal structures

## 🤝 Contributing

Found an error or want to add content? Contributions welcome!

1. Fork the repository
2. Create a feature branch
3. Submit a pull request

## 📜 License

This course content is provided for educational purposes.

## 🔗 Resources

- [PostgreSQL Documentation](https://www.postgresql.org/docs/current/)
- [PostgreSQL Source Code](https://github.com/postgres/postgres)
- [PostgreSQL Wiki](https://wiki.postgresql.org/)
- [The Internals of PostgreSQL](https://www.interdb.jp/pg/)

---

**Happy Learning! 🐘**
