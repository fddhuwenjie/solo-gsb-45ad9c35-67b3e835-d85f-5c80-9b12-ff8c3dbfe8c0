# 学分互认 API

用 FastAPI + SQLite 实现的外校课程学分互认服务。

## 业务规则

- **登记**：外校课程（编号、正整数学分、0~100 分成绩）、本校要求（编号、所需学分）、课程可替代的要求与最低成绩。
- **批量互认**：一次提交多条，仅当每条的外课学分足额且成绩达标、源课与要求在本批及现存"已认/待复核"记录中均未重复时，**整批确认**并按要求学分计入进度；否则**整批不写**（409 并返回全部原因）。
- **成绩更正**：更正后不再合格的"已认"互认原子地转为**待复核**，停止计分，但课程与要求两端的占用保留。
- **复核**：成绩恢复后须显式复核才能重新计分；复核仍不合格则维持待复核。
- **撤销**：删除互认记录，释放两端占用，之后可重新申请。
- **整批改派**：一次把多条**已认**互认改派到各自的目标要求（`POST /recognitions/reassign`）。按本批原占用**同时释放后**的状态核对：互认编号与目标要求在批内均不得重复、来源须是现存已认记录（待复核不可改派）、课程对目标要求的替代规则/当前成绩/当前学分均合格、目标要求未被批外的已认或待复核记录占用（两条记录互换要求不会相互冲突）。任一项不合格则**原互认与进度整批不变**；成功则保留原互认编号、更新要求与进度。
- **培养方案修订**：`POST /program/revisions` 一次提交多项**要求所需学分**与**替代规则最低成绩**改动（两类各为可选数组，至少提交一项）。同一要求/同一替代规则不得在本批重复，引用不存在或数值无效则**整次不生效**（409/422，事务回滚）；所有改动在一个事务内**共同生效**，随后按新标准复查现存互认：失去资格的**已认**转待复核、停止计分但继续占用两端，原本**待复核**的记录即使按新标准重新达标也须显式复核。成功返回已修改项目与转待复核的互认编号；已认进度按新学分计算，之后的申请、改派与复核均按新标准判断。
- **查询**：`/progress` 区分已认（计分）、待复核（不计分但占用）、空缺；一次查询取同一提交快照，不会混出改派/修订前后的结果。
- **候补互认**：要求已被占用时可登记候补（`POST /waitlists`，提交外校课程与要求编号）。仅当课程、要求、替代规则都存在，要求正由一条"已认/待复核"互认占用，课程未被互认占用且没有其它未兑现候补时才入队；候补不占用课程或要求，也不计入进度。撤销互认或整批改派释放名额后，在**同一事务内**按候补登记顺序依次核对：要求仍空缺、课程仍空闲、没有其它未兑现候补、且按**当时**的学分/成绩/替代规则合格——跳过不合格者，由首个合格者接替为已认并计分，其余保留候补；无人合格则保持空缺。改派互换或本批重新占用的要求不触发接替。撤销/改派响应返回接替出来的互认编号与来源候补编号；操作失败（如改派整批拒绝）时原候补、互认与进度都不变。`GET /waitlists` 可按要求（看候补顺序）、课程、状态查询候补与兑现结果。
- **并发**：所有写操作（申请、成绩更正、复核、撤销、改派、培养方案修订、候补登记）在进程锁 + `BEGIN IMMEDIATE` 事务内串行执行，检查与写入原子完成，并发操作不会重复使用或暴露中间态；接替与释放名额在同一事务内完成，并发申请、更正和方案修订不能越过接替过程；改派/修订之后的操作一律按新标准判断。

## 启动

```bash
docker compose up --build
```

服务监听 **8000** 端口，交互式文档见 <http://localhost:8000/docs>。
SQLite 数据库写入命名卷 `credit_data`（容器内 `/data/app.db`），首次启动自动建表，**重启后记录仍在**。

本地开发（不用 Docker）：

```bash
pip install -r requirements-dev.txt
uvicorn app.main:app --port 8000   # 数据库默认在 ./data/app.db，可用 DB_PATH 覆盖
pytest                             # 运行测试
```

## HTTP 示例

```bash
# 1. 登记外校课程：编号、学分、成绩
curl -s -X POST localhost:8000/courses \
  -H 'Content-Type: application/json' \
  -d '{"id": "C1", "credits": 4, "score": 90}'

# 2. 登记本校要求：编号、所需学分
curl -s -X POST localhost:8000/requirements \
  -H 'Content-Type: application/json' \
  -d '{"id": "R1", "required_credits": 4}'

# 3. 登记替代规则：C1 可替代 R1，最低成绩 60
curl -s -X POST localhost:8000/substitutions \
  -H 'Content-Type: application/json' \
  -d '{"course_id": "C1", "requirement_id": "R1", "min_score": 60}'

# 4. 批量提交互认（整批确认或整批拒绝）
curl -s -X POST localhost:8000/recognitions \
  -H 'Content-Type: application/json' \
  -d '{"items": [{"course_id": "C1", "requirement_id": "R1"}]}'

# 5. 查询进度：已认 / 待复核 / 空缺
curl -s localhost:8000/progress

# 6. 成绩更正到不达标 -> 互认转为待复核，停止计分但保留占用
curl -s -X PUT localhost:8000/courses/C1 \
  -H 'Content-Type: application/json' -d '{"score": 55}'
curl -s 'localhost:8000/recognitions?status=pending'

# 7. 成绩恢复后须显式复核才重新计分（复核仍不合格则维持待复核）
curl -s -X PUT localhost:8000/courses/C1 \
  -H 'Content-Type: application/json' -d '{"score": 88}'
curl -s -X POST localhost:8000/recognitions/1/review

# 8. 撤销互认，释放课程与要求的占用
curl -s -X DELETE localhost:8000/recognitions/1

# 9. 整批改派：两门已认课程互换培养要求
#    （逐条撤销再申请会暂时放出名额，互换还会因相互占用而失败；整批改派一次完成）
#    下列编号独立，可在全新数据库上直接执行；互认编号以第 9 步批量互认返回的 id 为准
curl -s -X POST localhost:8000/courses -H 'Content-Type: application/json' \
  -d '{"id": "D1", "credits": 4, "score": 86}'
curl -s -X POST localhost:8000/courses -H 'Content-Type: application/json' \
  -d '{"id": "D2", "credits": 4, "score": 78}'
curl -s -X POST localhost:8000/requirements -H 'Content-Type: application/json' \
  -d '{"id": "RD1", "required_credits": 4}'
curl -s -X POST localhost:8000/requirements -H 'Content-Type: application/json' \
  -d '{"id": "RD2", "required_credits": 4}'
curl -s -X POST localhost:8000/substitutions -H 'Content-Type: application/json' \
  -d '{"course_id": "D1", "requirement_id": "RD1", "min_score": 60}'
curl -s -X POST localhost:8000/substitutions -H 'Content-Type: application/json' \
  -d '{"course_id": "D1", "requirement_id": "RD2", "min_score": 60}'
curl -s -X POST localhost:8000/substitutions -H 'Content-Type: application/json' \
  -d '{"course_id": "D2", "requirement_id": "RD1", "min_score": 60}'
curl -s -X POST localhost:8000/substitutions -H 'Content-Type: application/json' \
  -d '{"course_id": "D2", "requirement_id": "RD2", "min_score": 60}'
curl -s -X POST localhost:8000/recognitions -H 'Content-Type: application/json' \
  -d '{"items": [{"course_id": "D1", "requirement_id": "RD1"}, {"course_id": "D2", "requirement_id": "RD2"}]}'
# 全新数据库上两条互认编号为 1、2（若跑过前面的示例则编号顺延，按实际 id 替换）
curl -s -X POST localhost:8000/recognitions/reassign -H 'Content-Type: application/json' \
  -d '{"items": [{"recognition_id": 1, "target_requirement_id": "RD2"}, {"recognition_id": 2, "target_requirement_id": "RD1"}]}'
# -> {"reassigned":[{"id":1,"from_requirement_id":"RD1","to_requirement_id":"RD2"}, ...]}

# 10. 目标要求被批外已认/待复核记录占用则整批拒绝（此时 RD1 由互认 2 占用）
curl -s -X POST localhost:8000/recognitions/reassign -H 'Content-Type: application/json' \
  -d '{"items": [{"recognition_id": 1, "target_requirement_id": "RD1"}]}'
# -> 409, detail 说明 RD1 已被批外互认 2 占用；原互认与进度保持不变

# 11. 培养方案修订：一次修改多项要求学分与替代规则最低成绩（共同生效）
#     下列编号独立，可在全新数据库上直接执行；互认编号以批量互认返回的 id 为准
curl -s -X POST localhost:8000/courses -H 'Content-Type: application/json' \
  -d '{"id": "E1", "credits": 4, "score": 90}'
curl -s -X POST localhost:8000/requirements -H 'Content-Type: application/json' \
  -d '{"id": "RE1", "required_credits": 4}'
curl -s -X POST localhost:8000/substitutions -H 'Content-Type: application/json' \
  -d '{"course_id": "E1", "requirement_id": "RE1", "min_score": 60}'
curl -s -X POST localhost:8000/recognitions -H 'Content-Type: application/json' \
  -d '{"items": [{"course_id": "E1", "requirement_id": "RE1"}]}'
# 把 RE1 所需学分提高到 6、E1->RE1 最低成绩提高到 95：E1 学分不足且成绩不达线，
# 互认 1 原子转待复核、停止计分但继续占用 RE1/E1 两端
curl -s -X POST localhost:8000/program/revisions \
  -H 'Content-Type: application/json' \
  -d '{"requirement_changes": [{"requirement_id": "RE1", "required_credits": 6}],
       "substitution_changes": [{"course_id": "E1", "requirement_id": "RE1", "min_score": 95}]}'
# -> {"revised":{"requirements":[{"requirement_id":"RE1","required_credits":6}],
#     "substitutions":[{"course_id":"E1","requirement_id":"RE1","min_score":95.0}]},
#     "moved_to_pending":[1]}
curl -s localhost:8000/progress   # 已认进度按新学分计算：RE1 变为 pending，earned_credits=0

# 12. 修订同样是整批原子：重复改动 / 引用不存在 -> 整次不生效（409，标准全部保持 6/95）
curl -s -X POST localhost:8000/program/revisions -H 'Content-Type: application/json' \
  -d '{"requirement_changes": [
        {"requirement_id": "RE1", "required_credits": 3},
        {"requirement_id": "RE1", "required_credits": 5}]}'
# -> 409, detail 说明 RE1 在本批重复；标准保持 6 不变
curl -s -X POST localhost:8000/program/revisions -H 'Content-Type: application/json' \
  -d '{"substitution_changes": [{"course_id": "E1", "requirement_id": "NOPE", "min_score": 60}]}'
# -> 409, detail 说明替代规则不存在；已有标准全部保持不变

# 13. 标准放宽不会自动恢复：把学分要求与最低成绩调回后，已转待复核的互认 1
#     即使重新达标，也必须显式复核才重新计分（moved_to_pending 为 []）
curl -s -X POST localhost:8000/program/revisions -H 'Content-Type: application/json' \
  -d '{"requirement_changes": [{"requirement_id": "RE1", "required_credits": 4}],
       "substitution_changes": [{"course_id": "E1", "requirement_id": "RE1", "min_score": 60}]}'
curl -s -X POST localhost:8000/recognitions/1/review   # -> status: confirmed
```

### 候补互认示例

```bash
# 14. 候补登记与接替：要求 RW 被占用时，其它课程排队，撤销后按登记顺序兑现
#     下列编号独立，可在全新数据库上直接执行
curl -s -X POST localhost:8000/courses -H 'Content-Type: application/json' \
  -d '{"id": "W1", "credits": 4, "score": 90}'
curl -s -X POST localhost:8000/courses -H 'Content-Type: application/json' \
  -d '{"id": "W2", "credits": 4, "score": 85}'
curl -s -X POST localhost:8000/courses -H 'Content-Type: application/json' \
  -d '{"id": "W3", "credits": 4, "score": 70}'
curl -s -X POST localhost:8000/requirements -H 'Content-Type: application/json' \
  -d '{"id": "RW", "required_credits": 4}'
curl -s -X POST localhost:8000/substitutions -H 'Content-Type: application/json' \
  -d '{"course_id": "W1", "requirement_id": "RW", "min_score": 60}'
curl -s -X POST localhost:8000/substitutions -H 'Content-Type: application/json' \
  -d '{"course_id": "W2", "requirement_id": "RW", "min_score": 60}'
curl -s -X POST localhost:8000/substitutions -H 'Content-Type: application/json' \
  -d '{"course_id": "W3", "requirement_id": "RW", "min_score": 80}'
# W1 先占用 RW
curl -s -X POST localhost:8000/recognitions -H 'Content-Type: application/json' \
  -d '{"items": [{"course_id": "W1", "requirement_id": "RW"}]}'
# W2、W3 登记候补（候补不占用、不计进度）；登记顺序即兑现顺序
curl -s -X POST localhost:8000/waitlists -H 'Content-Type: application/json' \
  -d '{"course_id": "W2", "requirement_id": "RW"}'
# -> {"id":1,...,"status":"waiting","recognition_id":null,...}
curl -s -X POST localhost:8000/waitlists -H 'Content-Type: application/json' \
  -d '{"course_id": "W3", "requirement_id": "RW"}'
# 按要求查询候补顺序
curl -s 'localhost:8000/waitlists?requirement_id=RW'
# -> W2(id=1) 在前、W3(id=2) 在后，均为 waiting

# 撤销 W1 的互认：同事务内按顺序核对，W2 合格 -> 接替为已认并计分
curl -s -X DELETE localhost:8000/recognitions/1
# -> {"revoked":1,
#     "released":{"course_id":"W1","requirement_id":"RW"},
#     "promoted":[{"recognition_id":2,"waitlist_id":1,
#                  "course_id":"W2","requirement_id":"RW"}]}
curl -s localhost:8000/progress   # RW 由互认 2（W2）占用，earned_credits=4
curl -s 'localhost:8000/waitlists?requirement_id=RW'
# -> 候补 1 status=fulfilled、recognition_id=2；候补 2（W3）仍 waiting

# 15. 不合格者被跳过、改派释放也触发兑现；改派互换不触发
#     现在 RW 由 W2（互认 2）占用，W3 仍在候补，且成绩 70 < 规则 80
curl -s -X DELETE localhost:8000/recognitions/2
# -> promoted 为 []：W3 当前不合格，无人接替，RW 保持空缺、W3 仍在候补
curl -s localhost:8000/progress   # RW status=vacant，earned_credits=0

# 整批改派释放名额同样触发：W1 占用 RW 后改派到另一要求，W3 按"当时"标准核对
curl -s -X POST localhost:8000/requirements -H 'Content-Type: application/json' \
  -d '{"id": "RO", "required_credits": 4}'
curl -s -X POST localhost:8000/substitutions -H 'Content-Type: application/json' \
  -d '{"course_id": "W1", "requirement_id": "RO", "min_score": 60}'
curl -s -X POST localhost:8000/recognitions -H 'Content-Type: application/json' \
  -d '{"items": [{"course_id": "W1", "requirement_id": "RW"}]}'   # 新互认 id=3
# 先放宽 W3->RW 的最低成绩（方案修订不会自动兑现候补）
curl -s -X POST localhost:8000/program/revisions -H 'Content-Type: application/json' \
  -d '{"substitution_changes":[{"course_id":"W3","requirement_id":"RW","min_score":60}]}'
# 把 W1 从 RW 改派到 RO -> RW 释放，W3 在同事务内接替
curl -s -X POST localhost:8000/recognitions/reassign -H 'Content-Type: application/json' \
  -d '{"items":[{"recognition_id":3,"target_requirement_id":"RO"}]}'
# -> {"reassigned":[{"id":3,"from_requirement_id":"RW","to_requirement_id":"RO"}],
#     "promoted":[{"recognition_id":4,"waitlist_id":2,
#                  "course_id":"W3","requirement_id":"RW"}]}
# 若两条互认互换要求，两个要求都在本批内重新占用，则 promoted 为 []。
```

## API 一览

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/courses` | 登记外校课程 |
| PUT | `/courses/{id}` | 成绩/学分更正（不合格互认转待复核） |
| GET | `/courses` | 课程列表 |
| POST | `/requirements` | 登记本校要求 |
| GET | `/requirements` | 要求列表 |
| POST | `/substitutions` | 登记替代规则与最低成绩 |
| GET | `/substitutions` | 规则列表 |
| POST | `/recognitions` | 批量互认（整批确认或整批不写） |
| GET | `/recognitions?status=confirmed\|pending` | 按状态查询互认 |
| POST | `/recognitions/reassign` | 整批改派已认互认到新要求（整批成功或整批不变） |
| POST | `/program/revisions` | 培养方案修订：批量改要求学分/规则最低成绩，按新标准复查已认互认 |
| POST | `/recognitions/{id}/review` | 复核待复核记录 |
| DELETE | `/recognitions/{id}` | 撤销并释放占用（同事务按顺序兑现候补） |
| POST | `/waitlists` | 登记候补互认（要求被占用、课程空闲且无其它未兑现候补） |
| GET | `/waitlists?requirement_id=&course_id=&status=waiting\|fulfilled` | 按要求查候补顺序与兑现结果 |
| GET | `/progress` | 进度：已认/待复核/空缺与学分统计（候补不计入） |
