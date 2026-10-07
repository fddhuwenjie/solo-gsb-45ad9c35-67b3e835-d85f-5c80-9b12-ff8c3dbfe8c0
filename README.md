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
- **候补互认**：`POST /waitlists` 提交外校课程与要求编号登记候补。仅当课程、要求、替代规则都存在、要求**正被一条已认/待复核互认占用**、课程**未被互认占用**且该课程**没有其他未兑现候补**时才入队（409 返回原因）。**候补不占用课程或要求，也不计入进度**。
  - `GET /waitlists?requirement_id=R1` 按登记顺序查询某要求的候补队列与兑现结果（`waiting`/`fulfilled`，兑现后带接替产生的互认编号与时间）。
  - **撤销互认**或**整批改派释放名额**后，在**同一事务内**按登记顺序检查候补课程：跳过仍被占用或按当时学分/成绩/替代规则不合格者，由**首个合格者接替为已认并计分**，其余保留候补；无人合格则要求保持空缺。
  - **改派互换**或被**本批重新占用**的要求不触发接替；整批改派失败时不触发任何接替。
  - 撤销/改派响应返回接替产生的互认编号与来源候补编号；接替与原操作同事务原子提交，失败时原候补、互认与进度均不变。成绩更正、复核、方案修订**不触发**接替。
- **查询**：`/progress` 区分已认（计分）、待复核（不计分但占用）、空缺；一次查询取同一提交快照，不会混出改派/修订/接替前后的结果。
- **并发**：所有写操作（申请、成绩更正、复核、撤销、改派、培养方案修订、候补登记与接替）在进程锁 + `BEGIN IMMEDIATE` 事务内串行执行，检查与写入原子完成，并发操作不会重复使用或暴露中间态；改派/修订/接替之后的操作一律按新标准与新占用关系判断。

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

# 14. 候补互认：要求已被占用时排队；撤销后按登记顺序首个合格者接替
#     下列编号独立，可在全新数据库上直接执行
curl -s -X POST localhost:8000/courses -H 'Content-Type: application/json' \
  -d '{"id": "F0", "credits": 4, "score": 90}'
curl -s -X POST localhost:8000/courses -H 'Content-Type: application/json' \
  -d '{"id": "F1", "credits": 4, "score": 88}'
curl -s -X POST localhost:8000/courses -H 'Content-Type: application/json' \
  -d '{"id": "F2", "credits": 4, "score": 55}'
curl -s -X POST localhost:8000/requirements -H 'Content-Type: application/json' \
  -d '{"id": "RF1", "required_credits": 4}'
curl -s -X POST localhost:8000/substitutions -H 'Content-Type: application/json' \
  -d '{"course_id": "F0", "requirement_id": "RF1", "min_score": 60}'
curl -s -X POST localhost:8000/substitutions -H 'Content-Type: application/json' \
  -d '{"course_id": "F1", "requirement_id": "RF1", "min_score": 60}'
curl -s -X POST localhost:8000/substitutions -H 'Content-Type: application/json' \
  -d '{"course_id": "F2", "requirement_id": "RF1", "min_score": 60}'
# F0 先认下 RF1
curl -s -X POST localhost:8000/recognitions -H 'Content-Type: application/json' \
  -d '{"items": [{"course_id": "F0", "requirement_id": "RF1"}]}'
# RF1 已被占用：F1、F2 登记候补（F2 当前成绩 55 不达标也允许排队，兑现时再判断）
curl -s -X POST localhost:8000/waitlists -H 'Content-Type: application/json' \
  -d '{"course_id": "F1", "requirement_id": "RF1"}'
# -> {"id":1,"course_id":"F1","requirement_id":"RF1","status":"waiting",
#     "recognition_id":null,"occupied_by_recognition":1,"occupied_by_status":"confirmed"}
curl -s -X POST localhost:8000/waitlists -H 'Content-Type: application/json' \
  -d '{"course_id": "F2", "requirement_id": "RF1"}'
# 按要求查询候补顺序（id 升序即登记顺序）与兑现结果
curl -s 'localhost:8000/waitlists?requirement_id=RF1'
# 撤销互认 1：同事务内按顺序检查，F1 合格 -> 接替为新互认并计分；F2 保留候补
curl -s -X DELETE localhost:8000/recognitions/1
# -> {"revoked":1,"released":{"course_id":"F0","requirement_id":"RF1"},
#     "promotion":{"requirement_id":"RF1","waitlist_id":1,
#                 "recognition_id":2,"course_id":"F1"}}
curl -s 'localhost:8000/waitlists?requirement_id=RF1'
# -> 候补 1 status=fulfilled recognition_id=2；候补 2 仍 waiting
curl -s localhost:8000/progress   # RF1 由互认 2（F1）占用，earned 计 4

# 15. 全员不合格则保持空缺：F2 成绩仍 55，撤销互认 2 后无人接替，候补保留
curl -s -X DELETE localhost:8000/recognitions/2
# -> {"revoked":2,...,"promotion":null}；RF1 空缺；之后成绩恢复也不自动兑现
curl -s -X PUT localhost:8000/courses/F2 -H 'Content-Type: application/json' -d '{"score": 80}'
curl -s localhost:8000/progress   # RF1 仍 vacant，候补仍 waiting

# 16. 候补登记条件：空缺要求不能登记（应直接申请互认）；同一课程只能有一条未兑现候补
curl -s -X POST localhost:8000/waitlists -H 'Content-Type: application/json' \
  -d '{"course_id": "F2", "requirement_id": "RF1"}'
# -> 409，detail 说明 F2 已有未兑现候补（open waitlist entry 2）；
#    若此时 RF1 空缺则返回 409 "requirement is vacant; submit a recognition instead"

# 17. 整批改派释放名额同样触发接替；互换或本批重新占用的要求不触发
#     下列编号独立，可在全新数据库上直接执行：G0 占用 RG1，G9 候补 RG1；
#     G0 有 RG2 规则，改派后 RG1 净释放 -> G9 同事务接替
curl -s -X POST localhost:8000/courses -H 'Content-Type: application/json' \
  -d '{"id": "G0", "credits": 4, "score": 90}'
curl -s -X POST localhost:8000/courses -H 'Content-Type: application/json' \
  -d '{"id": "G9", "credits": 4, "score": 80}'
curl -s -X POST localhost:8000/requirements -H 'Content-Type: application/json' \
  -d '{"id": "RG1", "required_credits": 4}'
curl -s -X POST localhost:8000/requirements -H 'Content-Type: application/json' \
  -d '{"id": "RG2", "required_credits": 4}'
curl -s -X POST localhost:8000/substitutions -H 'Content-Type: application/json' \
  -d '{"course_id": "G0", "requirement_id": "RG1", "min_score": 60}'
curl -s -X POST localhost:8000/substitutions -H 'Content-Type: application/json' \
  -d '{"course_id": "G0", "requirement_id": "RG2", "min_score": 60}'
curl -s -X POST localhost:8000/substitutions -H 'Content-Type: application/json' \
  -d '{"course_id": "G9", "requirement_id": "RG1", "min_score": 60}'
curl -s -X POST localhost:8000/recognitions -H 'Content-Type: application/json' \
  -d '{"items": [{"course_id": "G0", "requirement_id": "RG1"}]}'   # 互认 1
curl -s -X POST localhost:8000/waitlists -H 'Content-Type: application/json' \
  -d '{"course_id": "G9", "requirement_id": "RG1"}'                # 候补 1
curl -s -X POST localhost:8000/recognitions/reassign -H 'Content-Type: application/json' \
  -d '{"items": [{"recognition_id": 1, "target_requirement_id": "RG2"}]}'
# -> {"reassigned":[{"id":1,"from_requirement_id":"RG1","to_requirement_id":"RG2"}],
#     "promotions":[{"requirement_id":"RG1","waitlist_id":1,
#                   "recognition_id":2,"course_id":"G9"}]}
# 互换场景（两条互认交换要求）promotions 为 []：两个要求都被本批重新占用，不触发接替
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
| POST | `/waitlists` | 登记候补互认（要求被已认/待复核占用、课程空闲且无其他未兑现候补） |
| GET | `/waitlists?requirement_id=&status=waiting\|fulfilled` | 按登记顺序查询候补队列与兑现结果 |
| POST | `/recognitions/{id}/review` | 复核待复核记录 |
| DELETE | `/recognitions/{id}` | 撤销并释放占用；同事务按候补顺序兑现首个合格者 |
| GET | `/progress` | 进度：已认/待复核/空缺与学分统计 |
