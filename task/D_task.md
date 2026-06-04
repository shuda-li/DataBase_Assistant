# D_task.md — 成员D任务书：完整性约束设计

> **角色定位**：前面三位已经把房子建好了（A 画了户型图，B 出了施工图，C 浇了柱子和大梁），你来装门窗和防盗网——确保数据库里的数据不会乱七八糟，不会出现"选课人数=-5""成绩=999""一个学生对应一个不存在的院系"这种荒唐事。

---

## 零、先搞懂：完整性约束到底是什么？

### 用快递包裹打个比方

你开了一个菜鸟驿站。如果没有任何规则，会发生什么：

```
❌ 包裹没贴快递单 → 不知道谁的      → 这是「实体完整性」要管的
❌ 快递上的收件人不存在 → 送不出去   → 这是「参照完整性」要管的
❌ 包裹里寄了一只活鸡 → 违禁品       → 这是「用户自定义完整性」要管的
```

数据库也一样，每一行数据必须遵守规则，否则就会变成垃圾堆。你的任务就是**为 18 张表制定这些规则**。

---

### 三种完整性，一个表搞定

拿 STUDENT 表举例，三种约束分别标在这里：

```
CREATE TABLE Student (
    -- ========== ① 实体完整性 ==========
    stu_id  CHAR(10)  PRIMARY KEY,    ← 主键值不能空、不能重复

    stu_name VARCHAR(30) NOT NULL,    ← 姓名不能空（用户自定义完整性）

    -- ========== ② 参照完整性 ==========
    dept_id CHAR(4)  NOT NULL,
    FOREIGN KEY (dept_id)        ← 插入学生时，dept_id 必须在 Department 表里存在
        REFERENCES Department(dept_id)
        ON DELETE RESTRICT      ← 院系还有学生时，不允许删除院系
        ON UPDATE CASCADE,      ← 如果院系编号改了，学生的院系编号跟着自动更新

    -- ========== ③ 用户自定义完整性 ==========
    gender  CHAR(1)  CHECK(gender IN ('男','女')),         ← 性别只能是男或女
    degree  CHAR(4)  DEFAULT '本科'
                     CHECK(degree IN ('本科','硕士','博士')), ← 学历只能是这三个
    status  CHAR(4)  DEFAULT '在读'
                     CHECK(status IN ('在读','休学','退学','毕业'))
);
```

> 看到没？你们三个（实体、参照、用户自定义）住在同一张 CREATE TABLE 语句里。C 写了表的骨架，你负责往骨架上加这些 CHECK、FOREIGN KEY 的细节。

---

## 一、你从前面三位那里拿到什么

| 来源 | 产出物 | 你用它来做什么 |
|------|--------|---------------|
| **A** | `需求分析说明书.md` 第 2 章 | 找到每个实体的约束描述（如"性别 CHECK 男/女"） |
| **A** | `需求分析说明书.md` 第 4 章 | 非功能需求——数据完整性指导原则 |
| **B** | E-R 图 + 设计说明 | 确认联系类型（1:1/1:N/M:N），决定外键约束策略 |
| **C** | 关系模式清单（18 张表） | 主数据来源——表名、字段名必须一致 |
| **C** | **`.sql` 骨架脚本** 🔑 | **这就是你的工作底稿**——C 写好了 18 张表的 `CREATE TABLE`（字段+类型+主键+外键），你在上面直接加 CHECK、ON DELETE 策略、UNIQUE、触发器 |

---

## 二、你要交付什么

| 编号 | 产出物 | 格式 | 说明 |
|------|--------|------|------|
| ① | **完整性约束设计文档** | Markdown | 逐张表列出三类约束，含约束名称、约束内容、违反时的处理方式 |
| ② | **约束设计说明** | Markdown | 核心约束的设计理由（为什么要 RESTRICT、级联的逻辑、触发器的思路） |
| ③ | **最终版 CREATE TABLE SQL（必须交）** | `.sql` 文件 | 拿到 C 的骨架脚本，往上加 CHECK、ON DELETE 策略、UNIQUE、DEFAULT、触发器，出一份完整可执行的建表脚本。**这份是最终版，E 直接放进报告附录。** |

### 💡 SQL 流水线：C → 你 → 最终版 — 怎么实际操作

你和 C 不是各自写一份 SQL——而是一条流水线。下面是**用 PostgreSQL 的真实操作步骤**：

---

#### 你收到 C 发来的两个文件：

| 文件 | 是什么 |
|------|--------|
| `schema_skeleton.sql` | C 写好的 18 张表 CREATE TABLE，有字段+类型+PK+**有名字的 FK**（如 `CONSTRAINT fk_enrollment_student`），但 FK 那行**没有 ON DELETE**，也**没有 CHECK、UNIQUE、DEFAULT、触发器** |
| `关系模式设计说明.md` | C 的设计文档，你需要对照检查字段名 |

---

#### 第 0 步：拿到 C 的 SQL，在自己电脑上跑通（15 分钟）

1. 打开 pgAdmin 4（如果没装，让 C 把安装步骤发你——就是他那份任务书里的 6.1 节）
2. 同样创建一个数据库叫 `edu_system`（和 C 保持一致）
3. 打开 Query Tool（Tools → Query Tool）
4. 把 `schema_skeleton.sql` 的**全部内容**复制粘贴到编辑区
5. 按 F5 执行
6. 看 Messages 窗口：18 行 `CREATE TABLE Query returned successfully` = 跑通
7. 左侧刷新 Tables，数一下是不是 18 张

> ⚠️ 如果报错，大概率是建表顺序问题。C 的 SQL 已经排好顺序了，如果你还是报 "relation xxx does not exist"，检查是不是漏复制了前面的表。实在搞不定截图发给 C。

---

#### 第 1 步：先加最简单的东西 —— NOT NULL 和 DEFAULT（30 分钟）

打开 `schema_skeleton.sql`，**直接在这份文件上改**。从第一张表 DEPARTMENT 开始，逐行看：

- 哪个字段 A 的文档说"非空"？ → 加 `NOT NULL`
- 哪个字段 A 的文档说"默认xxx"？ → 加 `DEFAULT xxx`

动手前，把文件另存为 `schema_final.sql`（这样 C 的原始骨架不动，你改的是最终版）。

**以 STUDENT 表为例**，你现在看到的是 C 写的：

```sql
CREATE TABLE Student (
    stu_id      CHAR(10)    PRIMARY KEY,
    stu_name    VARCHAR(30) NOT NULL,
    gender      CHAR(1),
    birth_date  DATE,
    id_card     CHAR(18),
    enroll_year INT         NOT NULL,
    degree      CHAR(4)     NOT NULL,
    ...
```

你要改成：

```sql
CREATE TABLE Student (
    stu_id      CHAR(10)    PRIMARY KEY,
    stu_name    VARCHAR(30) NOT NULL,            -- C 已经写了 NOT NULL
    gender      CHAR(1),                         -- 可为空，不动
    birth_date  DATE,                            -- 可为空，不动
    id_card     CHAR(18),                        -- 可为空，不动
    enroll_year INT         NOT NULL,            -- C 已经写了
    degree      CHAR(4)     NOT NULL
                          DEFAULT '本科',        -- ← 你加的
    ...
```

---

#### 第 2 步：加 CHECK 约束（1 小时）

对着 D_task.md 第三章的逐表约束速查表，找到每个表需要 CHECK 的字段。直接在字段定义后面加。

**示例：给 STUDENT 表加 CHECK**：

```sql
    gender      CHAR(1)     CHECK(gender IN ('男','女')),   -- ← 你加的
    ...
    degree      CHAR(4)     NOT NULL
                            DEFAULT '本科'
                            CHECK(degree IN ('本科','硕士','博士')),  -- ← 你加的
    ...
    status      CHAR(4)     DEFAULT '在读'
                            CHECK(status IN ('在读','休学','退学','毕业')),  -- ← 你加的
```

**示例：给 SCHEDULE 表加 CHECK**：

```sql
    weekday      INT  NOT NULL CHECK(weekday BETWEEN 1 AND 7),
    start_period INT  NOT NULL CHECK(start_period BETWEEN 1 AND 12),
    end_period   INT  NOT NULL CHECK(end_period BETWEEN 2 AND 13),
    -- 跨字段 CHECK（写在所有字段之后，表结束之前）
    CONSTRAINT chk_period_order CHECK(end_period > start_period),
    CONSTRAINT chk_week_order CHECK(end_week IS NULL OR start_week IS NULL OR end_week > start_week),
```

> ⚠️ `end_period > start_period` 这种跨字段比较无法写在单个字段后面，必须写成表级约束 `CONSTRAINT chk_xxx CHECK(...)`，放在表的最后（右括号之前）。

---

#### 第 3 步：加 UNIQUE 约束（30 分钟）

对着 A 文档找所有标了 UNIQUE 的字段：

| 表 | 字段 | SQL |
|----|------|-----|
| Student | id_card | `UNIQUE` 写在字段后面 |
| Teacher | email | `UNIQUE` 写在字段后面 |
| Thesis | stu_id | `UNIQUE`（保证 1:1） |
| Grade | (stu_id, offering_id) | 表级约束（见下方） |
| Enrollment | (stu_id, offering_id) | 表级约束 |
| Attendance | (schedule_id, stu_id, attend_date) | 表级约束 |
| CourseOffering | (course_id, sem_id, class_seq) | 表级约束 |

**单字段**：直接在字段定义后面写 `UNIQUE`，如：

```sql
    id_card  CHAR(18) UNIQUE,       -- ← 你加的
```

**多字段复合唯一**（比如同一学生不能重复选同一门课）：在表的最后、右括号之前加：

```sql
    CONSTRAINT uq_enrollment_stu_offering UNIQUE(stu_id, offering_id)
```

---

#### 第 4 步：给每个外键补 ON DELETE 和 ON UPDATE（核心，1 小时）

这是你最重要的工作。C 已经帮你写好了所有外键，但**没有指定删除策略**。你要找到每一个 `CONSTRAINT fk_xxx` 的位置，在后面补上 `ON DELETE ... ON UPDATE ...`。

**C 的外键现在长这样**：

```sql
    CONSTRAINT fk_student_department
        FOREIGN KEY (dept_id) REFERENCES Department(dept_id)
```

**你要改成**：

```sql
    CONSTRAINT fk_student_department
        FOREIGN KEY (dept_id) REFERENCES Department(dept_id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE
```

> 💡 怎么快速找到所有外键？在 `schema_final.sql` 里 Ctrl+F 搜索 `CONSTRAINT fk_`，一个一个定位，一个一个改。

**逐条外键策略速查表（对着改）**：

| 外键名 | ON DELETE | ON UPDATE | 理由 |
|--------|:---------:|:---------:|------|
| fk_major_department | RESTRICT | CASCADE | 院系有专业不能删 |
| fk_student_department | RESTRICT | CASCADE | 院系有学生不能删 |
| fk_student_major | RESTRICT | CASCADE | 专业有学生不能删 |
| fk_teacher_department | RESTRICT | CASCADE | 院系有教师不能删 |
| fk_course_department | RESTRICT | CASCADE | 院系有课程不能删 |
| fk_course_prereq | **SET NULL** | CASCADE | 删了先修课，本课还在 |
| fk_rule_course | CASCADE | CASCADE | 课程删了，规则也没用 |
| fk_offering_course | RESTRICT | CASCADE | 课程有教学班不能删 |
| fk_offering_teacher | RESTRICT | CASCADE | 教师有课不能删 |
| fk_offering_semester | RESTRICT | CASCADE | 学期内有开课 |
| fk_enrollment_student | CASCADE | CASCADE | 学生退学，选课记录清 |
| fk_enrollment_offering | CASCADE | CASCADE | 开课取消，选课记录清 |
| fk_schedule_offering | CASCADE | CASCADE | 教学班取消，排课清 |
| fk_schedule_classroom | RESTRICT | CASCADE | 教室被占用不能删 |
| fk_change_schedule | CASCADE | CASCADE | 排课删了，调课记录没意义 |
| fk_change_classroom | RESTRICT | CASCADE | |
| fk_change_teacher | RESTRICT | CASCADE | |
| fk_attendance_schedule | CASCADE | CASCADE | |
| fk_attendance_student | CASCADE | CASCADE | |
| fk_grade_student | CASCADE | CASCADE | ⚠️ 有争议，见下方注释 |
| fk_grade_offering | CASCADE | CASCADE | 开课取消，成绩跟着清 |
| fk_grade_teacher | RESTRICT | CASCADE | |
| fk_item_grade | CASCADE | CASCADE | 成绩删了，子项没意义 |
| fk_template_offering | CASCADE | CASCADE | 开课取消，模板也没用 |
| fk_thesis_student | CASCADE | CASCADE | |
| fk_thesis_advisor | RESTRICT | CASCADE | |
| fk_thesis_reviewer | RESTRICT | CASCADE | |
| fk_project_course | **SET NULL** | CASCADE | 课程删了，项目变独立 |
| fk_project_student | CASCADE | CASCADE | |
| fk_project_advisor | RESTRICT | CASCADE | |

> ⚠️ `fk_grade_student` 用 CASCADE 意味着学生退学→成绩也没了。实际教务系统不会这样（成绩是历史记录需要保留）。你可以在这里标一个注释说明这个设计权衡，留给全组讨论。

---

#### 第 5 步：写触发器（难点，1-2 小时）

触发器的语法和普通 CREATE TABLE 不同，**不能写在建表语句里**。要在 `schema_final.sql` 的 **18 张表全部建完之后**，另起一段写。

**触发器 1：选课人数不超容量**

```sql
-- ============================================================
-- 触发器
-- ============================================================

-- 触发器1：选课时检查教学班容量，并自动更新当前选课人数
CREATE OR REPLACE FUNCTION check_enrollment_capacity()
RETURNS TRIGGER AS $$
DECLARE
    max_cap INT;
    cur_cnt INT;
BEGIN
    -- 查出这个教学班的容量和当前人数
    SELECT max_capacity, cur_enroll INTO max_cap, cur_cnt
    FROM CourseOffering
    WHERE offering_id = NEW.offering_id;

    -- 检查是否已满
    IF max_cap IS NOT NULL AND cur_cnt >= max_cap THEN
        RAISE EXCEPTION '选课失败：教学班 % 已满员（容量 %，当前 %）',
            NEW.offering_id, max_cap, cur_cnt;
    END IF;

    -- 没满，自动 +1
    UPDATE CourseOffering
    SET cur_enroll = cur_enroll + 1
    WHERE offering_id = NEW.offering_id;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_enrollment_insert
    BEFORE INSERT ON Enrollment
    FOR EACH ROW
    EXECUTE FUNCTION check_enrollment_capacity();


-- 退选时自动 -1
CREATE OR REPLACE FUNCTION decrease_enrollment_count()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE CourseOffering
    SET cur_enroll = cur_enroll - 1
    WHERE offering_id = OLD.offering_id;
    RETURN OLD;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_enrollment_delete
    AFTER DELETE ON Enrollment
    FOR EACH ROW
    EXECUTE FUNCTION decrease_enrollment_count();
```

**触发器 2：教室时间不冲突检测**（如果觉得太难，可以写伪代码 + 注释说明逻辑，不算错）

```sql
-- 触发器2：教室时间冲突检测
CREATE OR REPLACE FUNCTION check_classroom_conflict()
RETURNS TRIGGER AS $$
DECLARE
    conflict_count INT;
BEGIN
    -- 查找同一教室、同一星期、节次重叠的排课记录
    SELECT COUNT(*) INTO conflict_count
    FROM Schedule
    WHERE room_id = NEW.room_id
      AND schedule_id <> NEW.schedule_id       -- 排除自己
      AND weekday = NEW.weekday
      AND start_period < NEW.end_period         -- 时间区间重叠判定
      AND end_period > NEW.start_period
      -- 同时检查周次是否重叠
      AND (NEW.start_week IS NULL OR NEW.end_week IS NULL
           OR (start_week <= NEW.end_week AND end_week >= NEW.start_week));

    IF conflict_count > 0 THEN
        RAISE EXCEPTION '排课冲突：教室 % 在星期% 第%-%节已被占用',
            NEW.room_id, NEW.weekday, NEW.start_period, NEW.end_period;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_schedule_classroom_conflict
    BEFORE INSERT OR UPDATE ON Schedule
    FOR EACH ROW
    EXECUTE FUNCTION check_classroom_conflict();
```

**触发器 3：成绩子项权重之和不超过 1**

```sql
-- 触发器3：同一成绩的所有子项权重之和不超过 1
CREATE OR REPLACE FUNCTION check_grade_weight_sum()
RETURNS TRIGGER AS $$
DECLARE
    total_weight DECIMAL(4,2);
BEGIN
    SELECT COALESCE(SUM(weight), 0) INTO total_weight
    FROM GradeItem
    WHERE grade_id = NEW.grade_id;

    total_weight := total_weight + NEW.weight;

    IF total_weight > 1 THEN
        RAISE EXCEPTION '成绩子项权重之和不能超过 1（当前为 %）', total_weight;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_grade_item_weight
    BEFORE INSERT OR UPDATE ON GradeItem
    FOR EACH ROW
    EXECUTE FUNCTION check_grade_weight_sum();
```

---

#### 第 6 步：在 pgAdmin 里跑最终版 SQL，验证

1. 先把旧表全部删掉：在 Query Tool 里执行 `DROP TABLE IF EXISTS ... CASCADE;`（或直接删掉 `edu_system` 数据库重建一个）
2. 把 `schema_final.sql` 全部内容复制进 Query Tool
3. 按 F5 执行
4. 你应该看到 18 个 CREATE TABLE + 若干 CREATE TRIGGER 全部成功
5. 测试一下：手动 INSERT 一些测试数据，触发约束看看是否生效。比如：

```sql
-- 测试：插入一个超出容量的选课记录应该报错
INSERT INTO CourseOffering (offering_id, course_id, teacher_id, sem_id,
    class_seq, max_capacity, cur_enroll)
VALUES ('TEST001', 'CS21001', 'T2021001', '2024-2025-1', 1, 2, 2);

-- 现在容量=2，已选=2。再来一个学生选课应该被拒绝：
-- （因为没有真实的 Student 和 Course 数据，先插两条假的）
INSERT INTO Student (stu_id, stu_name, enroll_year, degree, dept_id)
VALUES ('S001', '测试学生', 2024, '本科', 'CS');
INSERT INTO Enrollment (stu_id, offering_id)
VALUES ('S001', 'TEST001');
-- 这一条应该报错！因为容量 2 且已选 2
```

如果你看到红色的 `ERROR: 选课失败：教学班 TEST001 已满员`，触发器就生效了！🎉

---

#### 第 7 步：导出最终版，发给 E

1. 把 `schema_final.sql` 文件（已经包含 18 张表 + 全部约束 + 触发器）发给 E
2. 微信附言："最终版 SQL 已跑通，约束和触发器都在里面。你放进报告附录 C。"
3. 同时把自己的 `完整性约束设计说明.md` 也发给 E（放报告第四章）

---

### 📦 最终版 `schema_final.sql` 长什么样（结构一览）

```sql
-- ============================================================
-- 教务管理系统 — 数据库完整建表脚本（最终版）
-- 骨架：C（关系模式设计）
-- 约束：D（完整性约束设计）
-- 数据库：edu_system (PostgreSQL)
-- ============================================================

-- 第1批：基础字典
CREATE TABLE Department (...)      -- D加了 NOT NULL 约束
CREATE TABLE Major (...)           -- D加了 ON DELETE RESTRICT
...

-- 第2批：主体实体
CREATE TABLE Student (             -- D加了 CHECK(gender IN...)
    ...                            -- D加了 DEFAULT '本科'
    ...                            -- D加了 UNIQUE(id_card)
    CONSTRAINT fk_student_department
        FOREIGN KEY (dept_id) REFERENCES Department(dept_id)
        ON DELETE RESTRICT         -- ← D 加的
        ON UPDATE CASCADE          -- ← D 加的
);

-- ... 18 张表全部 ...

-- 触发器（D 独立编写）
CREATE FUNCTION check_enrollment_capacity() ...
CREATE TRIGGER trg_enrollment_insert ...
CREATE FUNCTION check_classroom_conflict() ...
CREATE TRIGGER trg_schedule_classroom_conflict ...
CREATE FUNCTION check_grade_weight_sum() ...
CREATE TRIGGER trg_grade_item_weight ...
```

---

## 三、一张表一张表地加约束（核心工作）

你需要对每张表回答 4 个问题：

```
问题1：这张表的主键是什么？                 → 实体完整性
问题2：这张表有外键吗？指向谁？删了父表怎么处理？ → 参照完整性
问题3：哪些字段有取值范围限制？              → CHECK 约束
问题4：哪些字段有特殊的业务规则？            → 触发器或更复杂的约束
```

---

### 3.1 逐表约束速查表（你的施工清单）

下面按表列出所有需要加的约束。**你需要做的**：把每条约束翻译成 SQL 语法，写进文档。

---

#### ① DEPARTMENT（院系表）

| 类别 | 约束内容 | SQL 实现 |
|------|---------|----------|
| 实体完整性 | dept_id 为主键 | PRIMARY KEY |
| 自定义 | dept_name 不能为空 | NOT NULL |
| 自定义 | dept_id 格式：4 位字符编码 | 不强制，可选 CHECK |

---

#### ② MAJOR（专业表）

| 类别 | 约束内容 | SQL 实现 |
|------|---------|----------|
| 实体完整性 | major_id 为主键 | PRIMARY KEY |
| 参照完整性 | dept_id → Department | FK + ON DELETE RESTRICT + ON UPDATE CASCADE |
| 自定义 | major_name 不能为空 | NOT NULL |

---

#### ③ SEMESTER（学期表）

| 类别 | 约束内容 | SQL 实现 |
|------|---------|----------|
| 实体完整性 | sem_id 为主键 | PRIMARY KEY |
| 自定义 | sem_id 格式检查（如 '2024-2025-1'） | CHECK(LENGTH(sem_id)=11) — 可选 |
| 自定义 | end_date > start_date | CHECK(end_date > start_date) |

---

#### ④ STUDENT（学生表）

| 类别 | 约束内容 | SQL 实现 |
|------|---------|----------|
| 实体完整性 | stu_id 为主键 | PRIMARY KEY |
| 参照完整性 | dept_id → Department | FK + ON DELETE RESTRICT + ON UPDATE CASCADE |
| 参照完整性 | major_id → Major | FK + ON DELETE RESTRICT + ON UPDATE CASCADE |
| 自定义 | stu_name 不能为空 | NOT NULL |
| 自定义 | gender ∈ {'男','女'} | CHECK |
| 自定义 | degree ∈ {'本科','硕士','博士'} | CHECK |
| 自定义 | status ∈ {'在读','休学','退学','毕业'} | CHECK，DEFAULT '在读' |
| 自定义 | id_card 唯一（非空时） | UNIQUE（允许空值在某些 DBMS 中自动满足） |
| 自定义 | enroll_year 合理范围（如 2000-2099） | CHECK(enroll_year BETWEEN 2000 AND 2099) |

---

#### ⑤ TEACHER（教师表）

| 类别 | 约束内容 | SQL 实现 |
|------|---------|----------|
| 实体完整性 | teacher_id 为主键 | PRIMARY KEY |
| 参照完整性 | dept_id → Department | FK + ON DELETE RESTRICT + ON UPDATE CASCADE |
| 自定义 | teacher_name 不能为空 | NOT NULL |
| 自定义 | title ∈ {'教授','副教授','讲师','助教'} | CHECK |
| 自定义 | email 唯一（非空时） | UNIQUE |

---

#### ⑥ COURSE（课程表）

| 类别 | 约束内容 | SQL 实现 |
|------|---------|----------|
| 实体完整性 | course_id 为主键 | PRIMARY KEY |
| 参照完整性 | dept_id → Department | FK + ON DELETE RESTRICT + ON UPDATE CASCADE |
| 参照完整性 | prereq_id → Course（自引用） | FK + **ON DELETE SET NULL**（删了先修课，本课不受影响） + ON UPDATE CASCADE |
| 自定义 | course_name 不能为空 | NOT NULL |
| 自定义 | course_type ∈ {'理论','实验','实践'} | CHECK |
| 自定义 | default_credit > 0 | CHECK |
| 自定义 | total_hours > 0 | CHECK |

---

#### ⑦ COURSE_LEVEL_RULE（课程-学历规则表）

| 类别 | 约束内容 | SQL 实现 |
|------|---------|----------|
| 实体完整性 | (course_id, degree) 组合主键 | **复合主键** PRIMARY KEY(course_id, degree) |
| 参照完整性 | course_id → Course | FK + ON DELETE CASCADE（删了课程，规则也删） |
| 自定义 | degree ∈ {'本科','硕士','博士'} | CHECK |
| 自定义 | nature ∈ {'选修','必修'} | CHECK |
| 自定义 | credit > 0 | CHECK |
| 自定义 | 同一课程同一学历层次不能有两条规则 | 主键自动保证 |

---

#### ⑧ COURSE_OFFERING（开课表）

| 类别 | 约束内容 | SQL 实现 |
|------|---------|----------|
| 实体完整性 | offering_id 为主键 | PRIMARY KEY |
| 参照完整性 | course_id → Course | FK + ON DELETE RESTRICT |
| 参照完整性 | teacher_id → Teacher | FK + ON DELETE RESTRICT |
| 参照完整性 | sem_id → Semester | FK + ON DELETE RESTRICT |
| 自定义 | max_capacity > 0（或不限时为 NULL） | CHECK(max_capacity IS NULL OR max_capacity > 0) |
| 自定义 | cur_enroll >= 0 | CHECK(cur_enroll >= 0) |
| 自定义 | **cur_enroll <= max_capacity** | CHECK(cur_enroll IS NULL OR max_capacity IS NULL OR cur_enroll <= max_capacity) |
| 自定义 | language ∈ {'中文','英文','双语'} | CHECK |
| 自定义 | class_seq > 0 | CHECK |
| 自定义 | (course_id, sem_id, class_seq) 组合唯一 | UNIQUE——同一学期同一课程的两个班序号不能重复 |

---

#### ⑨ ENROLLMENT（选课表）

> ⚠️ 这是业务核心表，约束最多最重要

| 类别 | 约束内容 | SQL 实现 |
|------|---------|----------|
| 实体完整性 | enroll_id 为主键（自增） | PRIMARY KEY AUTO_INCREMENT |
| 参照完整性 | stu_id → Student | FK + ON DELETE CASCADE（学生退学，选课记录删除） |
| 参照完整性 | offering_id → CourseOffering | FK + ON DELETE CASCADE（开课取消，选课记录删除） |
| 自定义 | enroll_time 不能为空 | NOT NULL |
| 自定义 | status ∈ {'待审核','已确认','已退选'} | CHECK，DEFAULT '待审核' |
| 自定义 | nature_mark ∈ {'选修','必修'} | CHECK |
| 自定义 | **同一学生不能重复选同一课程同一学期** | UNIQUE(stu_id, offering_id) |
| 自定义 | **选课人数不能超过容量** | ⚡ 触发器（见 §4.1） |

---

#### ⑩ CLASSROOM（教室表）

| 类别 | 约束内容 | SQL 实现 |
|------|---------|----------|
| 实体完整性 | room_id 为主键 | PRIMARY KEY |
| 自定义 | capacity > 0 | CHECK |
| 自定义 | room_type ∈ {'普通','阶梯','实验室','多媒体'} | CHECK |
| 自定义 | projector_cnt >= 0 | CHECK |

---

#### ⑪ SCHEDULE（排课表）

| 类别 | 约束内容 | SQL 实现 |
|------|---------|----------|
| 实体完整性 | schedule_id 为主键（自增） | PRIMARY KEY AUTO_INCREMENT |
| 参照完整性 | offering_id → CourseOffering | FK + ON DELETE CASCADE |
| 参照完整性 | room_id → Classroom | FK + ON DELETE RESTRICT |
| 自定义 | weekday ∈ {1,2,3,4,5,6,7} | CHECK(weekday BETWEEN 1 AND 7) |
| 自定义 | start_period BETWEEN 1 AND 12 | CHECK |
| 自定义 | end_period BETWEEN 2 AND 13 | CHECK |
| 自定义 | **end_period > start_period** | CHECK |
| 自定义 | start_week BETWEEN 1 AND 20 | CHECK |
| 自定义 | end_week BETWEEN 1 AND 20 | CHECK |
| 自定义 | **end_week > start_week**（非空时） | CHECK |
| 自定义 | week_parity ∈ {'全','单','双'} | CHECK，DEFAULT '全' |
| 自定义 | sched_type ∈ {'理论','实验','实践'} | CHECK |
| 自定义 | **教室时间不冲突** | ⚡ 触发器（见 §4.2） |
| 自定义 | **教师时间不冲突** | ⚡ 触发器（见 §4.2） |

---

#### ⑫ SCHEDULE_CHANGE（调课表）

| 类别 | 约束内容 | SQL 实现 |
|------|---------|----------|
| 实体完整性 | change_id 为主键（自增） | PRIMARY KEY AUTO_INCREMENT |
| 参照完整性 | schedule_id → Schedule | FK + ON DELETE CASCADE |
| 参照完整性 | new_room_id → Classroom | FK + ON DELETE RESTRICT，允许空 |
| 参照完整性 | applicant_id → Teacher | FK + ON DELETE RESTRICT |
| 自定义 | change_type ∈ {'停课','调时间','调教室','补课'} | CHECK |
| 自定义 | status ∈ {'待审批','已通过','已驳回'} | CHECK，DEFAULT '待审批' |
| 自定义 | 停课时 new_date 必须为空 | CHECK(change_type='停课' AND new_date IS NULL OR change_type<>'停课') |

---

#### ⑬ GRADE（成绩表）

| 类别 | 约束内容 | SQL 实现 |
|------|---------|----------|
| 实体完整性 | grade_id 为主键（自增） | PRIMARY KEY AUTO_INCREMENT |
| 参照完整性 | stu_id → Student | FK + ON DELETE CASCADE |
| 参照完整性 | offering_id → CourseOffering | FK + ON DELETE CASCADE |
| 参照完整性 | input_teacher → Teacher | FK + ON DELETE RESTRICT |
| 自定义 | **total_score BETWEEN 0 AND 100**（非空时） | CHECK(total_score IS NULL OR total_score BETWEEN 0 AND 100) |
| 自定义 | grade_level ∈ {'A','B','C','D','F'} | CHECK — 也可不存，通过 total_score 计算 |
| 自定义 | retake_score BETWEEN 0 AND 100（非空时） | CHECK |
| 自定义 | 一个学生同一学期同一课程只有一条成绩 | UNIQUE(stu_id, offering_id) |

---

#### ⑭ GRADE_ITEM（成绩子项表）

| 类别 | 约束内容 | SQL 实现 |
|------|---------|----------|
| 实体完整性 | item_id 为主键（自增） | PRIMARY KEY AUTO_INCREMENT |
| 参照完整性 | grade_id → Grade | FK + ON DELETE CASCADE（成绩删了，子项也没意义） |
| 自定义 | item_score BETWEEN 0 AND 100 | CHECK |
| 自定义 | weight BETWEEN 0 AND 1 | CHECK(weight > 0 AND weight <= 1) |
| 自定义 | **同一条成绩的所有子项权重之和 = 1** | ⚡ 触发器（见 §4.3） |

---

#### ⑮ GRADING_TEMPLATE（评分模板表）

| 类别 | 约束内容 | SQL 实现 |
|------|---------|----------|
| 实体完整性 | template_id 为主键（自增） | PRIMARY KEY AUTO_INCREMENT |
| 参照完整性 | offering_id → CourseOffering | FK + ON DELETE CASCADE |
| 自定义 | weight BETWEEN 0 AND 1 | CHECK |
| 自定义 | score_std ∈ {'百分制','五级制','二级制'} | CHECK |

---

#### ⑯ THESIS（毕业设计表）

| 类别 | 约束内容 | SQL 实现 |
|------|---------|----------|
| 实体完整性 | thesis_id 为主键（自增） | PRIMARY KEY AUTO_INCREMENT |
| 参照完整性 | stu_id → Student | FK + **UNIQUE**（1:1 关系） + ON DELETE CASCADE |
| 参照完整性 | advisor_id → Teacher | FK + ON DELETE RESTRICT |
| 参照完整性 | reviewer_id → Teacher | FK + ON DELETE RESTRICT |
| 自定义 | title 不能为空 | NOT NULL |
| 自定义 | final_score BETWEEN 0 AND 100（非空时） | CHECK |
| 自定义 | status ∈ {'未开题','进行中','已答辩','未通过'} | CHECK |
| 自定义 | **指导教师 ≠ 评阅教师** | CHECK(advisor_id <> reviewer_id) |

> 🔑 CHECK(advisor_id <> reviewer_id) 这条非常重要：自己指导的毕设不能自己评阅。这是业务规则，不是 A 明确写的，但逻辑上必须有。

---

#### ⑰ PROJECT（项目设计表）

| 类别 | 约束内容 | SQL 实现 |
|------|---------|----------|
| 实体完整性 | project_id 为主键（自增） | PRIMARY KEY AUTO_INCREMENT |
| 参照完整性 | stu_id → Student | FK + ON DELETE CASCADE |
| 参照完整性 | advisor_id → Teacher | FK + ON DELETE RESTRICT |
| 参照完整性 | course_id → Course | FK + ON DELETE SET NULL（独立项目允许课程为空） |
| 自定义 | proj_type ∈ {'课程设计','独立实践','大作业'} | CHECK |
| 自定义 | score BETWEEN 0 AND 100（非空时） | CHECK |

---

#### ⑱ ATTENDANCE（考勤表）

| 类别 | 约束内容 | SQL 实现 |
|------|---------|----------|
| 实体完整性 | attend_id 为主键（自增） | PRIMARY KEY AUTO_INCREMENT |
| 参照完整性 | schedule_id → Schedule | FK + ON DELETE CASCADE |
| 参照完整性 | stu_id → Student | FK + ON DELETE CASCADE |
| 自定义 | attend_date 不能为空 | NOT NULL |
| 自定义 | status ∈ {'出勤','迟到','早退','请假','缺勤'} | CHECK |
| 自定义 | 同一学生同一节课只一条考勤记录 | UNIQUE(schedule_id, stu_id, attend_date) |

---

## 四、触发器设计（这 3 个是难点，也是加分点）

前面那些 CHECK 和 FK 都是"静态约束"——数据库自己就能检查。但有些约束需要"动态检查"——插入一行时要跨表查数据。这些就需要**触发器**。

---

### 4.1 触发器 1：选课人数不能超过容量

```
触发时机：向 ENROLLMENT 表 INSERT 一行时
触发逻辑：
  1. 查出这行对应的 COURSE_OFFERING.max_capacity 和 cur_enroll
  2. 如果 cur_enroll >= max_capacity，报错："该教学班已满，无法选课"
  3. 如果没问题，自动把 COURSE_OFFERING.cur_enroll + 1

同样，DELETE 一行时自动 cur_enroll - 1
```

```
伪代码：
CREATE TRIGGER trg_enroll_check
BEFORE INSERT ON ENROLLMENT
FOR EACH ROW
BEGIN
    DECLARE max_cap INT;
    DECLARE cur_cnt INT;

    SELECT max_capacity, cur_enroll INTO max_cap, cur_cnt
    FROM COURSE_OFFERING
    WHERE offering_id = NEW.offering_id;

    IF cur_cnt >= max_cap THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = '选课失败：教学班已满员';
    END IF;
END;
```

> 这是你文档里最亮眼的约束。老师看到这个就知道你认真思考了业务。

---

### 4.2 触发器 2：教室/教师时间不冲突

```
触发时机：向 SCHEDULE 表 INSERT 或 UPDATE 时
触发逻辑：
  检查同一教室，同一星期，同一节次，时间重叠的排课记录是否已有其他课
  同样检查同一教师（通过 COURSE_OFFERING.teacher_id）

检测时间重叠的公式：
  已有记录的 (weekday, start_period, end_period)
  和新的记录的 (weekday, start_period, end_period)
  重叠条件：weekday 相同 AND start_period < 已有.end_period AND end_period > 已有.start_period
```

> 这是最难的一个触发器，SQL 写法比较复杂。如果觉得太难，可以在文档里写清楚触发逻辑（伪代码），注明"实现层面留待应用层处理"，也是可以的。

---

### 4.3 触发器 3：成绩子项权重之和 = 1

```
触发时机：向 GRADE_ITEM 表 INSERT 或 UPDATE 时
触发逻辑：
  对同一个 grade_id 的所有子项，SUM(weight) 不允许超过 1

如果超过 1，报错："成绩子项权重之和不能超过 1"
```

---

## 五、外键删除策略速查（ON DELETE 怎么选）

| 策略 | 含义 | 什么时候用 |
|------|------|-----------|
| **CASCADE** | 父表删了，子表跟着删 | 子表离开父表没意义（如成绩→成绩子项，选课→学生删除选课记录） |
| **RESTRICT** | 父表有子记录时禁止删除 | 父表是核心字典，不能随便丢（如院系还有学生时不能删院系） |
| **SET NULL** | 父表删了，子表外键变空 | 关联是可选的（如 PROJECT.course_id、COURSE.prereq_id） |
| **NO ACTION** | 同 RESTRICT，但检查时机不同 | 一般不单独用 |

**本项目各外键的策略总表**：

| 外键 | 策略 | 理由 |
|------|:----:|------|
| Student.dept_id → Department | RESTRICT | 院系有学生不能删 |
| Student.major_id → Major | RESTRICT | 专业有学生不能删 |
| Teacher.dept_id → Department | RESTRICT | 院系有老师不能删 |
| Course.dept_id → Department | RESTRICT | 院系有课程不能删 |
| Major.dept_id → Department | RESTRICT | 院系有专业不能删 |
| Course.prereq_id → Course | **SET NULL** | 删了一门先修课，后修课还在，只是不再要求先修 |
| CourseLevelRule.course_id → Course | CASCADE | 课程删了，其学分规则也没用 |
| CourseOffering.course_id → Course | RESTRICT | 课程有教学班在运行，不能删 |
| CourseOffering.teacher_id → Teacher | RESTRICT | 老师有课在上，至少学期结束前不能删 |
| CourseOffering.sem_id → Semester | RESTRICT | 学期内有开课记录 |
| Enrollment.stu_id → Student | CASCADE | 学生退学，选课记录清理 |
| Enrollment.offering_id → CourseOffering | CASCADE | 开课取消，选课记录清理 |
| Schedule.offering_id → CourseOffering | CASCADE | 教学班取消，排课清理 |
| Schedule.room_id → Classroom | RESTRICT | 教室有排课，不能删 |
| ScheduleChange.schedule_id → Schedule | CASCADE | 排课删了，调课记录也没意义 |
| ScheduleChange.new_room_id → Classroom | RESTRICT | |
| ScheduleChange.applicant_id → Teacher | RESTRICT | |
| Grade.stu_id → Student | CASCADE | 学生退学，成绩保留？ ⚠️ 这个有争议，见下方 |
| Grade.offering_id → CourseOffering | CASCADE | 开课取消，成绩跟着清 |
| Grade.input_teacher → Teacher | RESTRICT | |
| GradeItem.grade_id → Grade | CASCADE | 成绩删了，子项没意义 |
| GradingTemplate.offering_id → CourseOffering | CASCADE | 开课取消，模板也没用 |
| Thesis.stu_id → Student | CASCADE | |
| Thesis.advisor_id → Teacher | RESTRICT | |
| Thesis.reviewer_id → Teacher | RESTRICT | |
| Project.stu_id → Student | CASCADE | |
| Project.advisor_id → Teacher | RESTRICT | |
| Project.course_id → Course | **SET NULL** | 课程被删，项目变成独立项目 |
| Attendance.schedule_id → Schedule | CASCADE | |
| Attendance.stu_id → Student | CASCADE | |

> ⚠️ GRADE.stu_id ON DELETE CASCADE 意味着学生退学了，成绩也没了——这对教务系统来说可能不合适。建议改为 **RESTRICT 或 SET NULL**，然后在设计说明里解释：成绩是学生的重要历史数据，即使学生退学也应保留。可以让全组讨论。

---

## 六、写约束设计说明（1 小时）

写一份 `完整性约束设计说明.md`：

```markdown
# 完整性约束设计说明

## 1. 设计原则
- 三类完整性全覆盖（实体、参照、用户自定义）
- 约束尽可能下沉到数据库层（而非应用层），保证数据无论从哪里写入都受约束
- 触发器仅用于需要跨表检查的场景

## 2. 核心约束设计理由

### 2.1 为什么选课人数用触发器而不是 CHECK？
（因为 CHECK 只能检查本行的值，不能跨表查 COURSE_OFFERING.cur_enroll）

### 2.2 为什么教室冲突检测复杂？
（因为需要 JOIN SCHEDULE 表和 CLASSROOM 表，且涉及时间区间重叠判断）

### 2.3 为什么 THESIS 表加了 CHECK(advisor_id <> reviewer_id)？
（业务规则：自己不能评阅自己指导的毕设。A 没写，但逻辑上必须）

### 2.4 为什么 COURSE.prereq_id 用 SET NULL 而不是 CASCADE？
（删了先修课，后修课不应被删除——学生已经学过了）

## 3. 妥协与权衡
- 哪些约束因为复杂度放到了应用层？（如教室冲突触发器如果太难写）
- 哪些约束做了简化？

## 4. 约束清单汇总
（把所有 18 张表的约束汇总成一个大表，方便 E 贴进报告）
```

---

## 七、常见错误（提前避开）

| ❌ 错误 | ✅ 正确做法 |
|---------|-----------|
| 所有外键用 CASCADE | 不同的外键场景需要不同的策略——字典表用 RESTRICT，中间表/子表用 CASCADE |
| CHECK 里写子查询 | 标准 SQL 的 CHECK 不能跨表查数据——跨表的用触发器 |
| 忘了复合主键 | COURSE_LEVEL_RULE 用 (course_id, degree) 做复合主键，而不是自增 ID |
| 忘了 1:1 要加 UNIQUE | THESIS.stu_id 必须是 UNIQUE，否则一个学生能有两个毕设 |
| advisor_id 和 reviewer_id 当同一个 FK 处理 | 是两个独立的 FK，分别指向 TEACHER，字段名不同 |
| 触发器写得过于复杂 | 伪代码也行，把逻辑讲清楚，SQL 语法可以留到实现阶段 |
| 忘了自引用外键的特殊处理 | Course.prereq_id → Course 必须允许为空 + 用 SET NULL |

---

## 八、交付前自查清单

- [ ] **18 张表**每张都写了约束？
- [ ] 所有 **28 条外键**都标了 ON DELETE 策略？
- [ ] 主键全部用 PRIMARY KEY 标识？
- [ ] NOT NULL 标在必填字段上？
- [ ] CHECK 约束覆盖了所有枚举字段？
- [ ] UNIQUE 约束标在需要唯一的字段上？（id_card, email, stu_id+offering_id 等）
- [ ] 3 个触发器的逻辑写清楚了吗？（选课容量、教室冲突、权重之和）
- [ ] THESIS 的 CHECK(advisor_id <> reviewer_id) 加了吗？
- [ ] COURSE.prereq_id 允许为空？
- [ ] 设计说明文档写了（至少 2-4 节）？
- [ ] 和 C 的表名/字段名完全一致？

---

## 九、建议时间

| 步骤 | 用时 | 节点 |
|------|------|------|
| 读 C 的关系模式 + A 的第 2 章 | — | 第 5 天拿到 |
| 收到 C 的 SQL 骨架 → 以此为底稿开工 | — | 第 5 天 |
| 给 SQL 骨架加 CHECK/NOT NULL/UNIQUE/DEFAULT | 2-3h | 第 5-6 天 |
| 给 28 条外键加 ON DELETE 策略 | 1-2h | 第 6 天 |
| 写 3 个触发器 | 1-2h | 第 6 天 |
| 写设计说明 | 1h | 第 6 天 |
| 最终 SQL 脚本发给 E（放附录） | 0.5h | 第 6-7 天 |
| 约束文档发给 E（放第四章） | 0.5h | 第 6-7 天 |

---

## 十、和 E 的对接

- E 会把你的约束清单直接复制进报告。所以：
  - 表格格式统一，字段不要缩写
  - 每个约束注明约束名（如 `CHK_gender`、`FK_student_dept`），方便引用
  - 触发器写清楚逻辑（伪代码 + 自然语言都行），E 才能看懂

---

## 十一、参考资源

| 资源 | 位置 | 怎么用 |
|------|------|--------|
| A 需求分析（第 2 章约束列） | `需求分析说明书.md` | 每个字段的 CHECK、NOT NULL、DEFAULT 来源 |
| A 需求分析（第 4 章非功能需求） | `需求分析说明书.md` | NF-1 数据完整性指导原则 |
| C 的关系模式 | C 交付的文档 | 主数据来源——表名、字段名必须一致 |
| 表结构图（关系矩阵） | `表结构图.md` 第 5 章 | 快速查所有 FK，漏了哪个一眼看出 |

---

> 📌 **一句话记住你的任务**：C 建了 18 张表，你在每张表上贴"规则标签"——什么能存、什么不能存、删了父表怎么办。你的目标：任何一个数据，无论从哪个入口进来，都不可能是脏数据。你就是数据库的保安+质检员。
