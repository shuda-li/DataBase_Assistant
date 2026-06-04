# C_task.md — 成员C任务书：关系模式设计

> **角色定位**：你把 B 的 E-R 图翻译成数据库能听懂的语言——关系模式（也就是一张张具体的表结构）。D 在你的表上写约束，E 把你的表写进报告。你写错了字段类型，后面全组白干。

---

## 零、先搞懂：关系模式到底是什么？

### 一个你天天见的东西

其实你早就见过关系模式了——就是 **Excel 表格的表头**：

```
Excel 里你看到的:
┌──────┬────────┬──────┬──────────┐
│ 学号  │  姓名   │ 性别  │  院系     │
├──────┼────────┼──────┼──────────┤
│202401│ 张三   │ 男   │ 计算机系  │
│202402│ 李四   │ 女   │ 数学系    │
└──────┴────────┴──────┴──────────┘

关系模式写出来就是:
STUDENT(学号, 姓名, 性别, 院系)
        ───
       主键画下划线
```

**你的任务**：把 Excel 表头那一行的事情说清楚——这个表叫什么名字、有哪些列、每列存什么类型的数据、哪个列是唯一标识、哪个列指向别的表。

---

### 关系模式的标准写法

作业里一般用这两种格式之一，和老师确认用哪种：

**格式A（课本式，一行写完）**：
```
STUDENT(stu_id, stu_name, gender, birth_date, dept_id)
        ──────
        下划线 = 主键
        虚线   = 外键（有时不标）
```

**格式B（表格式，推荐，一目了然）**：
```
STUDENT
┌────────────┬──────────────┬──────────┐
│ 字段名      │ 类型         │ 约束      │
├────────────┼──────────────┼──────────┤
│ stu_id     │ CHAR(10)     │ PRIMARY KEY│
│ stu_name   │ VARCHAR(30)  │ NOT NULL  │
│ dept_id    │ CHAR(4)      │ FK→Department│
└────────────┴──────────────┴──────────┘
```

> 💡 建议用格式B。D 能直接对着抄约束，E 能直接贴进报告，你自己检查也快。

---

## 一、你从 B 那里拿到什么

| B 的产出 | 你用它来做什么 |
|----------|---------------|
| **全局 E-R 图** | 确认 18 个实体一个不少，联系方向对 |
| **E-R 图设计说明** | 理解为什么 1:1、为什么拆表，决定关系模式怎么转 |
| **子系统拆分图** | 分模块写 SQL，不用 18 张表一口气全写 |

> ⚠️ 如果 B 还没交付，你可以先用 `需求分析说明书.md` 第 6 章的关系草图和 `表结构图.md` 开工——但要等 B 正式交付后做一次对齐，确保属性名一致。

---

## 二、你要交付什么

| 编号 | 产出物 | 格式 | 说明 |
|------|--------|------|------|
| ① | **关系模式清单**（18 张表） | Markdown 表格 | 每张表 = 一个表格，列出字段名 / 类型 / 约束 / 说明 |
| ② | **关系模式设计说明** | Markdown | 解释 E-R 图到关系模式的转换过程、关键决策 |
| ③ | **CREATE TABLE SQL 骨架脚本** | `.sql` 文件 | 18 张表的建表骨架（字段名+类型+主键+外键），**不含 CHECK 和触发器**。写完发给 D，D 补约束后出最终版。这是你和 D 的衔接件，必须交。 |

---

## 三、开工前必须掌握的 6 条转换规则

E-R 图是画画的，关系模式是建表的。从图到表有一套死规则，背下来就行：

---

### 规则 1：实体 → 表（最基础）

> 每一个实体（矩形框）变成一张表。实体的属性变成表的字段。实体的主键变成表的 PRIMARY KEY。

```
E-R 图中:                    关系模式:
┌──────────┐                 STUDENT(stu_id, stu_name, gender, ...)
│ STUDENT  │                         ──────
│──────────│                         主键
│ stu_id   │
│ stu_name │
│ gender   │
└──────────┘
```

---

### 规则 2：1:N 联系 → 在 N 端加外键（最常见）

> "一个院系有多个学生" = 1:N。在学生表里加一个 `dept_id` 字段，指向院系表的主键。

```
E-R 图:                     关系模式:
DEPARTMENT ──1:N──▶ STUDENT     STUDENT 表里多一个字段:
                               dept_id CHAR(4)  FK → DEPARTMENT(dept_id)
                                                 ─────────────────────
                                                 这就是外键！
```

> 💡 口诀：**"N 的那头加外键"**。谁是多（N），谁就多存一个字段指向一（1）。

---

### 规则 3：M:N 联系 → 新建一张中间表（最重要，最容易忘）

> "一个学生可以选多门课，一门课可以被多个学生选" = M:N。必须新建一张表，表里至少有两个外键，分别指向两边。

```
E-R 图:
STUDENT ──M:N──▶ COURSE_OFFERING

不是直接在 STUDENT 里面塞一堆课程！
也不是直接在 COURSE_OFFERING 里面塞一堆学生！

而是建一张新表:
┌──────────────┐
│ ENROLLMENT   │
├──────────────┤
│ enroll_id PK │  ← 自增主键
│ stu_id   FK  │  ← 指向 STUDENT
│ offering_id FK│  ← 指向 COURSE_OFFERING
└──────────────┘

这张表就叫「中间表」或「关联表」。
```

> 💡 口诀：**"多对多，中间表"**。这个项目里 ENROLLMENT 就是中间表，A 已经帮你建好了，你只需要写它的关系模式。

---

### 规则 4：1:1 联系 → 外键放在哪边都行，但要有理由

> "一个学生最多一个毕设" = 1:1。可以把外键放在任意一边，但通常会放在"更可能为空"或"后产生的"那一边。

```
STUDENT ──1:1──▶ THESIS

选 STUDENT 放 FK？ → 不行，因为大一到大三的学生没有毕设，会产生一堆 NULL
选 THESIS 放 FK？  → 合理！THESIS.stu_id 指向 STUDENT，永远不会空

所以:
THESIS(stu_id CHAR(10) FK → STUDENT(stu_id) UNIQUE)
                                               ──────
                                    UNIQUE 保证 1:1（一个学生只能被引用一次）
```

> 💡 口诀：**"一对一，外键加唯一"**。外键 + UNIQUE = 1:1。

---

### 规则 5：自引用 → 外键指向自己

> 课程有"先修课程"——一门课需要先修另一门课。这就是自己引用自己。

```
COURSE 表中的 prereq_id 字段:
prereq_id CHAR(8)  FK → COURSE(course_id)
                        ─────────────────
                        指向同一张表的主键！

这个字段必须允许为空（因为不是每门课都有先修要求）。
```

---

### 规则 6：弱实体 → 主键 = 父表主键 + 自己的部分键

> "成绩子项"不能独立存在——它必须属于某条成绩记录。这种叫弱实体。

```
GRADE_ITEM 的主键设计:
┌──────────────┐
│ GRADE_ITEM   │
├──────────────┤
│ grade_id FK  │  ← 指向 GRADE，作为主键的一部分
│ item_name    │  ← 也作为主键的一部分
└──────────────┘

复合主键: (grade_id, item_name)
意思是：同一条成绩记录里，不能有两个叫"平时成绩"的子项。
```

> 或者更简单的做法：直接给 GRADE_ITEM 一个自增主键 `item_id`，然后 grade_id 做普通外键。两种都可以，本项目用自增主键更简单。

---

## 四、正式步骤

### 第 1 步：对着 B 的 E-R 图，列出所有 18 张表名（10 分钟）

```
☐ DEPARTMENT      院系
☐ MAJOR           专业
☐ SEMESTER        学期
☐ STUDENT         学生
☐ TEACHER         教师
☐ COURSE          课程
☐ COURSE_LEVEL_RULE  课程-学历规则
☐ COURSE_OFFERING    开课(教学班)
☐ ENROLLMENT      选课
☐ CLASSROOM       教室
☐ SCHEDULE        排课
☐ SCHEDULE_CHANGE    调课
☐ GRADE           成绩
☐ GRADE_ITEM      成绩子项
☐ GRADING_TEMPLATE   评分模板
☐ THESIS          毕业设计
☐ PROJECT         项目设计
☐ ATTENDANCE      考勤
```

---

### 第 2 步：逐张表写出完整的关系模式（核心工作，4-5 小时）

每张表你需要确定 4 件事：

| 要确定的 | 怎么确定 | 
|----------|---------|
| **字段名** | 照抄 A 文档里的属性名。**不要自己改名**，否则 D 和 B 连不上 |
| **SQL 类型** | 按下方「类型速查表」翻译 |
| **主键** | 照抄 A 文档里标了 PK 的字段 |
| **外键** | 照抄 A 文档里标了 FK 的字段，补齐指向 |

#### SQL 类型速查表

| A 文档里的类型写法 | 翻译成 SQL 类型 | 例子 |
|-------------------|----------------|------|
| 定长字符(N) | `CHAR(N)` | CHAR(10) |
| 变长字符(N) | `VARCHAR(N)` | VARCHAR(50) |
| 文本 | `TEXT` | TEXT |
| 整数 | `INT` | INT |
| 小数(M,D) | `DECIMAL(M,D)` | DECIMAL(5,2) |
| 日期 | `DATE` | DATE |
| 时间戳 | `TIMESTAMP` 或 `DATETIME` | TIMESTAMP |
| 布尔 | `BOOLEAN` 或 `TINYINT(1)` | BOOLEAN |

#### 示例：怎么写一张完整的关系模式表

以 STUDENT 为例：

```markdown
### STUDENT（学生表）

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| stu_id | CHAR(10) | **PRIMARY KEY** | 学号，如 2024010001 |
| stu_name | VARCHAR(30) | NOT NULL | 姓名 |
| gender | CHAR(1) | CHECK(gender IN ('男','女')) | 性别 |
| birth_date | DATE | | 出生日期 |
| id_card | CHAR(18) | UNIQUE | 身份证号，留学生可为空 |
| enroll_year | INT | NOT NULL | 入学年份 |
| degree | CHAR(4) | NOT NULL, DEFAULT '本科', CHECK(degree IN ('本科','硕士','博士')) | 学历层次 |
| dept_id | CHAR(4) | NOT NULL, **FK → DEPARTMENT(dept_id)** | 院系编号 |
| major_id | CHAR(6) | **FK → MAJOR(major_id)** | 专业编号 |
| admin_class | VARCHAR(30) | | 管理班级 |
| status | CHAR(4) | DEFAULT '在读', CHECK(status IN ('在读','休学','退学','毕业')) | 学籍状态 |
| phone | CHAR(15) | | 联系电话 |
| email | VARCHAR(50) | | 电子邮箱 |
```

> 关键：外键一定要写出 **`FK → 目标表(目标字段)`**，D 才能直接抄。

---

### 第 3 步：验证所有 FK 的方向和基数（1 小时）

对着 B 的 E-R 图检查每一条外键：

```
B 图上的 1:N →
    1 的那端 = 被引用的父表（外键指向它）
    N 的那端 = 放外键的子表

B 图上的 1:1 →
    外键在哪边，哪边就标 UNIQUE

B 图上的 M:N →
    一定有中间表，中间表两个外键分别指两边
```

**验证清单**（对照着 B 的图逐条过）：

- [ ] COURSE_OFFERING.course_id → COURSE ✓（1:N 的 N 端）
- [ ] COURSE_OFFERING.teacher_id → TEACHER ✓（1:N 的 N 端）
- [ ] COURSE_OFFERING.sem_id → SEMESTER ✓（1:N 的 N 端）
- [ ] ENROLLMENT.stu_id → STUDENT ✓（M:N 中间表）
- [ ] ENROLLMENT.offering_id → COURSE_OFFERING ✓（M:N 中间表）
- [ ] SCHEDULE.offering_id → COURSE_OFFERING ✓
- [ ] SCHEDULE.room_id → CLASSROOM ✓
- [ ] GRADE.stu_id → STUDENT ✓
- [ ] GRADE.offering_id → COURSE_OFFERING ✓
- [ ] GRADE_ITEM.grade_id → GRADE ✓
- [ ] ATTENDANCE.schedule_id → SCHEDULE ✓
- [ ] ATTENDANCE.stu_id → STUDENT ✓
- [ ] SCHEDULE_CHANGE.schedule_id → SCHEDULE ✓
- [ ] THESIS.stu_id → STUDENT ✓（1:1，需 UNIQUE）
- [ ] THESIS.advisor_id → TEACHER ✓
- [ ] THESIS.reviewer_id → TEACHER ✓（注意和 advisor_id 用不同字段名！）
- [ ] PROJECT.stu_id → STUDENT ✓
- [ ] PROJECT.advisor_id → TEACHER ✓
- [ ] PROJECT.course_id → COURSE ✓
- [ ] COURSE.prereq_id → COURSE ✓（自引用，允许空）
- [ ] STUDENT.dept_id → DEPARTMENT ✓
- [ ] TEACHER.dept_id → DEPARTMENT ✓
- [ ] COURSE.dept_id → DEPARTMENT ✓
- [ ] MAJOR.dept_id → DEPARTMENT ✓
- [ ] GRADING_TEMPLATE.offering_id → COURSE_OFFERING ✓
- [ ] GRADE.input_teacher → TEACHER ✓
- [ ] SCHEDULE_CHANGE.applicant_id → TEACHER ✓
- [ ] COURSE_LEVEL_RULE.course_id → COURSE ✓

> 共 28 条外键。一个一个打勾，少一条都不行。

---

### 第 4 步：处理特殊约束（30 分钟）

有些约束不是外键，但也很重要：

| 约束类型 | 例子 | 在关系模式里怎么写 |
|----------|------|------------------|
| 复合唯一约束 | 同一学生不能重复选同一课程 | `UNIQUE(stu_id, offering_id)` |
| 复合唯一约束 | 同一学生同一节课只一条考勤 | `UNIQUE(schedule_id, stu_id, attend_date)` |
| CHECK 约束 | 性别只能是男或女 | `CHECK(gender IN ('男','女'))` |
| CHECK 约束 | 周次要 1-7 | `CHECK(weekday BETWEEN 1 AND 7)` |
| CHECK 约束 | 结束节次 > 开始节次 | `CHECK(end_period > start_period)` |
| DEFAULT 值 | 学籍状态默认'在读' | `DEFAULT '在读'` |
| DEFAULT 值 | 选课人数默认 0 | `DEFAULT 0` |

在每张表的关系模式表格中，把这类约束也写进"约束"列。

---

### 第 5 步：写关系模式设计说明（1-1.5 小时）

写一份 `关系模式设计说明.md`：

```markdown
# 关系模式设计说明

## 1. 转换总览
- 18 个实体 → 18 张表
- 28 条外键
- 1 个中间表（ENROLLMENT，拆解 M:N）
- 1 个 1:1 关系（STUDENT ↔ THESIS，FK+UNIQUE 实现）

## 2. 关键设计决策

### 2.1 为什么 ENROLLMENT 单独成表
（解释 M:N 必须用中间表，否则要么在学生表里塞数组，要么在课程表里塞数组，都不符合范式）

### 2.2 为什么 COURSE_LEVEL_RULE 单独成表
（解释同一门课本研不同学分的设计——如果写死在 COURSE 表里，一个课只能有一个学分值）

### 2.3 为什么 GRADE 和 GRADE_ITEM 拆成两张表
（解释成绩子项可自定义——如果全平铺在 GRADE 表里，不知道这学期有几个子项）

### 2.4 自引用怎么处理
（COURSE.prereq_id → COURSE.course_id，允许为空）

### 2.5 1:1 怎么处理
（THESIS.stu_id UNIQUE，保证一个学生只有一个毕设）

## 3. 范式说明
- 所有表满足 3NF（第三范式）吗？
- 有没有故意违反范式的地方（如 ENROLLMENT.nature_mark 冗余存储）？为什么？

## 4. 类型选择说明
- 为什么学号用 CHAR(10) 而不是 INT？
- 为什么学分用 DECIMAL(3,1) 而不是 FLOAT？
```

---

### 第 6 步：在 PostgreSQL 里写 CREATE TABLE 骨架，跑通后发给 D（必须，2 小时）

> ⚠️ 这一步是你的 SQL 代码交付。D 拿到你的 `.sql` 文件当底稿，往上加约束。你写的每一个外键名 D 都要引用，写错了 D 就卡住了。

---

#### 6.1 安装 PostgreSQL 和 pgAdmin（还没装的话，30分钟）

**第 1 小步：下载安装**

1. 打开浏览器，访问 https://www.postgresql.org/download/windows/
2. 点 "Download the installer"，选最新版本（16.x 或 17.x 都可以）
3. 下载后双击运行，一路点 Next
4. ⚠️ **到这一步停一下**：安装程序会让你设置一个 `postgres` 超级用户的密码。设一个你记得住的，比如 `admin123`。**把这个密码记在手机便签里**，后面每次连接数据库都要用
5. 端口保持默认 `5432`，地域选 `Chinese (Simplified)`
6. 到 "Select Components" 时，确保 **pgAdmin 4** 是勾上的（一般默认就是）
7. 继续 Next 直到完成

**第 2 小步：打开 pgAdmin**

1. 在开始菜单搜 "pgAdmin 4"，打开
2. 第一次打开会要求设一个 pgAdmin 的登录密码（这是 pgAdmin 这个网页工具自己的密码，跟刚才的数据库密码不是一回事）。设一个简单的比如 `pgadmin123`，也记下来
3. 左侧有一个浏览器树，展开 "Servers" → 应该能看到 "PostgreSQL 16"（或17）
4. 点它一下，输入你刚才设的 `postgres` 用户的密码（`admin123`）
5. 连上后，你会在左侧看到 "Databases" 下面有一个默认数据库叫 `postgres`

**第 3 小步：创建你的项目数据库**

1. 右键点 "Databases" → "Create" → "Database..."
2. Database 填 `edu_system`（教务系统）
3. Owner 选 `postgres`
4. 其他不动，点 Save
5. 左侧展开 `edu_system`，你看到 Schemas → public → Tables（现在是空的）

**第 4 小步：打开查询工具**

1. 菜单栏点 Tools → Query Tool（或者直接点工具栏上的 🔍 带 SQL 图标的小按钮）
2. 上面是编辑区（写 SQL 的地方），下面是结果区（看执行结果的地方）
3. 输入 `SELECT 1;` 然后按 F5 或点 ⚡ 闪电图标执行
4. 如果下面出现一行 `1`，说明一切正常。你在 PostgreSQL 里的第一个 SQL 执行成功了！

---

#### 6.2 写 18 张表的 CREATE TABLE（1 小时）

现在正式开始写建表脚本。**重要：你必须按下面的顺序建表**，因为外键要求引用的父表先存在：

```
建表顺序（不能乱！）：

第 1 批：被引用的基础表（这些表没有外键，或只引用先建的表）
  ① DEPARTMENT
  ② MAJOR           ← 引用 DEPARTMENT，所以 DEPARTMENT 先建
  ③ SEMESTER        ← 无外键
  ④ CLASSROOM       ← 无外键

第 2 批：主体实体表
  ⑤ STUDENT         ← 引用 DEPARTMENT, MAJOR
  ⑥ TEACHER         ← 引用 DEPARTMENT
  ⑦ COURSE          ← 引用 DEPARTMENT, COURSE(自引用，允许空)

第 3 批：关联表
  ⑧ COURSE_LEVEL_RULE    ← 引用 COURSE
  ⑨ COURSE_OFFERING      ← 引用 COURSE, TEACHER, SEMESTER

第 4 批：业务子表（引用上面的关联表）
  ⑩ ENROLLMENT           ← 引用 STUDENT, COURSE_OFFERING
  ⑪ SCHEDULE             ← 引用 COURSE_OFFERING, CLASSROOM
  ⑫ SCHEDULE_CHANGE      ← 引用 SCHEDULE, CLASSROOM, TEACHER
  ⑬ ATTENDANCE           ← 引用 SCHEDULE, STUDENT

第 5 批：成绩体系
  ⑭ GRADE                ← 引用 STUDENT, COURSE_OFFERING, TEACHER
  ⑮ GRADE_ITEM           ← 引用 GRADE
  ⑯ GRADING_TEMPLATE     ← 引用 COURSE_OFFERING

第 6 批：毕设&项目
  ⑰ THESIS               ← 引用 STUDENT, TEACHER(×2)
  ⑱ PROJECT              ← 引用 STUDENT, TEACHER, COURSE
```

**开始写！** 在 pgAdmin 的 Query Tool 里，逐张表输入以下 SQL：

> ⚠️ 下面给你完整的 18 张表骨架 SQL。**直接复制到 Query Tool 里**，但注意：
> - 全部字段名、类型来自 A 的需求分析说明书第 2 章
> - **D 会在这份 SQL 上直接改**，所以要给每个外键起一个好认的名字
> - 外键命名规则：`fk_<当前表名>_<引用的父表名>`

```sql
-- ============================================================
-- 教务管理系统 — 数据库建表骨架
-- 作者：C（关系模式设计）
-- 说明：此脚本只含字段+类型+主键+外键（含 ON DELETE / ON UPDATE）
--       CHECK、DEFAULT、UNIQUE、触发器由 D 后续补充
-- 数据库：edu_system (PostgreSQL)
-- ============================================================

-- ==================== 第1批：基础字典表 ====================

-- ① DEPARTMENT 院系表
CREATE TABLE Department (
    dept_id   CHAR(4)      PRIMARY KEY,
    dept_name VARCHAR(50)  NOT NULL
);

-- ② MAJOR 专业表
CREATE TABLE Major (
    major_id   CHAR(6)     PRIMARY KEY,
    major_name VARCHAR(50) NOT NULL,
    dept_id    CHAR(4)     NOT NULL,
    CONSTRAINT fk_major_department  -- ← D 找的就是这个名字
        FOREIGN KEY (dept_id) REFERENCES Department(dept_id)
);

-- ③ SEMESTER 学期表
CREATE TABLE Semester (
    sem_id     CHAR(11)    PRIMARY KEY,   -- 如 '2024-2025-1'
    sem_name   VARCHAR(30) NOT NULL,
    start_date DATE,
    end_date   DATE
);

-- ④ CLASSROOM 教室表
CREATE TABLE Classroom (
    room_id    CHAR(6)     PRIMARY KEY,   -- 如 'J1-201'
    room_name  VARCHAR(30),
    capacity   INT         NOT NULL,
    room_type  VARCHAR(10),               -- 普通|阶梯|实验室|多媒体
    building   VARCHAR(30)
);

-- ==================== 第2批：主体实体表 ====================

-- ⑤ STUDENT 学生表
CREATE TABLE Student (
    stu_id      CHAR(10)    PRIMARY KEY,  -- 如 '2024010001'
    stu_name    VARCHAR(30) NOT NULL,
    gender      CHAR(1),
    birth_date  DATE,
    id_card     CHAR(18),
    enroll_year INT         NOT NULL,
    degree      CHAR(4)     NOT NULL,     -- 本科|硕士|博士
    dept_id     CHAR(4)     NOT NULL,
    major_id    CHAR(6),
    admin_class VARCHAR(30),
    status      CHAR(4),                  -- 在读|休学|退学|毕业
    phone       VARCHAR(15),
    email       VARCHAR(50),
    CONSTRAINT fk_student_department
        FOREIGN KEY (dept_id) REFERENCES Department(dept_id),
    CONSTRAINT fk_student_major
        FOREIGN KEY (major_id) REFERENCES Major(major_id)
);

-- ⑥ TEACHER 教师表
CREATE TABLE Teacher (
    teacher_id   CHAR(8)     PRIMARY KEY,  -- 如 'T2021001'
    teacher_name VARCHAR(30) NOT NULL,
    gender       CHAR(1),
    birth_date   DATE,
    title        VARCHAR(10),              -- 教授|副教授|讲师|助教
    dept_id      CHAR(4)     NOT NULL,
    edu_bg       CHAR(4),                 -- 博士|硕士|学士
    phone        VARCHAR(15),
    email        VARCHAR(50),
    CONSTRAINT fk_teacher_department
        FOREIGN KEY (dept_id) REFERENCES Department(dept_id)
);

-- ⑦ COURSE 课程表（有自引用）
CREATE TABLE Course (
    course_id    CHAR(8)     PRIMARY KEY,  -- 如 'CS21001'
    course_name  VARCHAR(50) NOT NULL,
    course_type  CHAR(4),                  -- 理论|实验|实践
    default_credit DECIMAL(3,1),
    total_hours  INT,
    dept_id      CHAR(4),
    prereq_id    CHAR(8),                 -- 先修课程，自引用，可为空
    description  TEXT,
    CONSTRAINT fk_course_department
        FOREIGN KEY (dept_id) REFERENCES Department(dept_id),
    CONSTRAINT fk_course_prereq          -- 自引用外键
        FOREIGN KEY (prereq_id) REFERENCES Course(course_id)
);

-- ==================== 第3批：关联表 ====================

-- ⑧ COURSE_LEVEL_RULE 课程-学历规则表
CREATE TABLE CourseLevelRule (
    course_id  CHAR(8)     NOT NULL,
    degree     CHAR(4)     NOT NULL,       -- 本科|硕士|博士
    nature     CHAR(4)     NOT NULL,       -- 选修|必修
    credit     DECIMAL(3,1) NOT NULL,
    PRIMARY KEY (course_id, degree),       -- 复合主键
    CONSTRAINT fk_rule_course
        FOREIGN KEY (course_id) REFERENCES Course(course_id)
);

-- ⑨ COURSE_OFFERING 开课表（核心枢纽表）
CREATE TABLE CourseOffering (
    offering_id  CHAR(12)    PRIMARY KEY,   -- 如 'OFF202401001'
    course_id    CHAR(8)     NOT NULL,
    teacher_id   CHAR(8)     NOT NULL,
    sem_id       CHAR(11)    NOT NULL,
    class_seq    INT         NOT NULL,      -- 教学班序号
    max_capacity INT,
    cur_enroll   INT         DEFAULT 0,
    language     CHAR(4)     DEFAULT '中文',
    has_lab      BOOLEAN     DEFAULT FALSE,
    CONSTRAINT fk_offering_course
        FOREIGN KEY (course_id) REFERENCES Course(course_id),
    CONSTRAINT fk_offering_teacher
        FOREIGN KEY (teacher_id) REFERENCES Teacher(teacher_id),
    CONSTRAINT fk_offering_semester
        FOREIGN KEY (sem_id) REFERENCES Semester(sem_id)
);

-- ==================== 第4批：业务子表 ====================

-- ⑩ ENROLLMENT 选课表（M:N 中间表）
CREATE TABLE Enrollment (
    enroll_id    SERIAL      PRIMARY KEY,   -- 自增主键
    stu_id       CHAR(10)    NOT NULL,
    offering_id  CHAR(12)    NOT NULL,
    enroll_time  TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    status       CHAR(3)     DEFAULT '待审核',  -- 待审核|已确认|已退选
    nature_mark  CHAR(2),                  -- 选修|必修（冗余存储）
    CONSTRAINT fk_enrollment_student
        FOREIGN KEY (stu_id) REFERENCES Student(stu_id),
    CONSTRAINT fk_enrollment_offering
        FOREIGN KEY (offering_id) REFERENCES CourseOffering(offering_id)
);

-- ⑪ SCHEDULE 排课表
CREATE TABLE Schedule (
    schedule_id  SERIAL      PRIMARY KEY,
    offering_id  CHAR(12)    NOT NULL,
    room_id      CHAR(6)     NOT NULL,
    weekday      INT         NOT NULL,      -- 星期 1-7
    start_period INT         NOT NULL,      -- 开始节次
    end_period   INT         NOT NULL,      -- 结束节次
    start_week   INT,                      -- 起始周，可为空（实验课待定）
    end_week     INT,                      -- 结束周
    week_parity  CHAR(1)     DEFAULT '全', -- 全|单|双
    sched_type   CHAR(3),                  -- 理论|实验|实践
    note         VARCHAR(100),
    CONSTRAINT fk_schedule_offering
        FOREIGN KEY (offering_id) REFERENCES CourseOffering(offering_id),
    CONSTRAINT fk_schedule_classroom
        FOREIGN KEY (room_id) REFERENCES Classroom(room_id)
);

-- ⑫ SCHEDULE_CHANGE 调课表
CREATE TABLE ScheduleChange (
    change_id    SERIAL      PRIMARY KEY,
    schedule_id  INT         NOT NULL,
    change_type  CHAR(3)     NOT NULL,      -- 停课|调时间|调教室|补课
    orig_date    DATE        NOT NULL,
    new_date     DATE,                      -- 停课时为空
    new_room_id  CHAR(6),
    new_start    INT,
    new_end      INT,
    reason       VARCHAR(200),
    applicant_id CHAR(8)     NOT NULL,      -- 申请教师
    status       CHAR(3)     DEFAULT '待审批',
    apply_time   TIMESTAMP   DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_change_schedule
        FOREIGN KEY (schedule_id) REFERENCES Schedule(schedule_id),
    CONSTRAINT fk_change_classroom
        FOREIGN KEY (new_room_id) REFERENCES Classroom(room_id),
    CONSTRAINT fk_change_teacher
        FOREIGN KEY (applicant_id) REFERENCES Teacher(teacher_id)
);

-- ⑬ ATTENDANCE 考勤表
CREATE TABLE Attendance (
    attend_id    SERIAL      PRIMARY KEY,
    schedule_id  INT         NOT NULL,
    stu_id       CHAR(10)    NOT NULL,
    attend_date  DATE        NOT NULL,
    status       CHAR(2)     NOT NULL,      -- 出勤|迟到|早退|请假|缺勤
    note         VARCHAR(100),
    CONSTRAINT fk_attendance_schedule
        FOREIGN KEY (schedule_id) REFERENCES Schedule(schedule_id),
    CONSTRAINT fk_attendance_student
        FOREIGN KEY (stu_id) REFERENCES Student(stu_id)
);

-- ==================== 第5批：成绩体系 ====================

-- ⑭ GRADE 成绩主表
CREATE TABLE Grade (
    grade_id      SERIAL      PRIMARY KEY,
    stu_id        CHAR(10)    NOT NULL,
    offering_id   CHAR(12)    NOT NULL,
    total_score   DECIMAL(5,2),            -- 总评，未完成时为空
    grade_level   CHAR(4),
    is_retake     BOOLEAN     DEFAULT FALSE,
    retake_score  DECIMAL(5,2),
    input_teacher CHAR(8),
    input_time    TIMESTAMP   DEFAULT CURRENT_TIMESTAMP,
    note          VARCHAR(100),
    CONSTRAINT fk_grade_student
        FOREIGN KEY (stu_id) REFERENCES Student(stu_id),
    CONSTRAINT fk_grade_offering
        FOREIGN KEY (offering_id) REFERENCES CourseOffering(offering_id),
    CONSTRAINT fk_grade_teacher
        FOREIGN KEY (input_teacher) REFERENCES Teacher(teacher_id)
);

-- ⑮ GRADE_ITEM 成绩子项表
CREATE TABLE GradeItem (
    item_id    SERIAL      PRIMARY KEY,
    grade_id   INT         NOT NULL,
    item_name  VARCHAR(20) NOT NULL,       -- 如"平时成绩""期中""期末""实验"
    item_score DECIMAL(5,2),
    weight     DECIMAL(3,2),              -- 如 0.3
    CONSTRAINT fk_item_grade
        FOREIGN KEY (grade_id) REFERENCES Grade(grade_id)
);

-- ⑯ GRADING_TEMPLATE 评分模板表
CREATE TABLE GradingTemplate (
    template_id  SERIAL      PRIMARY KEY,
    offering_id  CHAR(12)    NOT NULL,
    item_name    VARCHAR(20) NOT NULL,
    weight       DECIMAL(3,2),
    score_std    CHAR(3),                  -- 百分制|五级制|二级制
    CONSTRAINT fk_template_offering
        FOREIGN KEY (offering_id) REFERENCES CourseOffering(offering_id)
);

-- ==================== 第6批：毕设 & 项目 ====================

-- ⑰ THESIS 毕业设计表
CREATE TABLE Thesis (
    thesis_id    SERIAL      PRIMARY KEY,
    stu_id       CHAR(10)    NOT NULL,
    advisor_id   CHAR(8)     NOT NULL,      -- 指导教师
    title        VARCHAR(200) NOT NULL,
    source       VARCHAR(20),               -- 题目来源
    start_date   DATE,
    defense_date DATE,
    final_score  DECIMAL(5,2),
    grade_level  CHAR(4),
    reviewer_id  CHAR(8),                   -- 评阅教师
    status       CHAR(3)     DEFAULT '未开题',
    CONSTRAINT fk_thesis_student
        FOREIGN KEY (stu_id) REFERENCES Student(stu_id),
    CONSTRAINT fk_thesis_advisor
        FOREIGN KEY (advisor_id) REFERENCES Teacher(teacher_id),
    CONSTRAINT fk_thesis_reviewer
        FOREIGN KEY (reviewer_id) REFERENCES Teacher(teacher_id)
);

-- ⑱ PROJECT 项目设计表
CREATE TABLE Project (
    project_id   SERIAL      PRIMARY KEY,
    project_name VARCHAR(100),
    course_id    CHAR(8),                   -- 所属课程，独立项目时为空
    stu_id       CHAR(10)    NOT NULL,
    advisor_id   CHAR(8)     NOT NULL,
    proj_type    CHAR(4),                   -- 课程设计|独立实践|大作业
    submit_date  DATE,
    score        DECIMAL(5,2),
    note         TEXT,
    CONSTRAINT fk_project_course
        FOREIGN KEY (course_id) REFERENCES Course(course_id),
    CONSTRAINT fk_project_student
        FOREIGN KEY (stu_id) REFERENCES Student(stu_id),
    CONSTRAINT fk_project_advisor
        FOREIGN KEY (advisor_id) REFERENCES Teacher(teacher_id)
);
```

---

#### 6.3 执行并验证（关键步骤！）

1. 把上面全部 SQL 复制到 pgAdmin 的 Query Tool 编辑区
2. 按 **F5** 或点 ⚡ 闪电图标执行
3. 看下面的 Messages 窗口。你应该看到 18 行 `CREATE TABLE Query returned successfully`
4. 如果有红色的 ERROR，读错误信息修正（常见错误：引用的表还没建→检查建表顺序；字段类型写错→对照 A 的文档）
5. 跑通后，左侧刷新 `edu_system` → Schemas → public → Tables，你应该看到 18 张表：

```
attendance        course             courselevelrule    enrollment
classroom         courseoffering     department         grade
gradeitem         gradingtemplate    major              project
schedule          schedulechange     semester           student
teacher           thesis
```

数一下，是不是 18 张？少一张都不行。

---

#### 6.4 导出 .sql 文件，发给 D

1. pgAdmin 里，右键点 `edu_system` 数据库
2. 选 "Backup..."
3. Format 选 "Plain"
4. 文件名填 `schema_skeleton.sql`，保存到桌面
5. 或者更简单：把 Query Tool 里的全部 SQL 复制出来，粘贴到记事本，保存为 `schema_skeleton.sql`（**编码选 UTF-8**）
6. 微信/QQ 发给 D，附言：

> "骨架 SQL 已跑通，18张表在 edu_system 数据库里。每个外键都起了名字（fk_xxx_yyy 格式），你直接在上面加 CHECK 和 ON DELETE 就行。建表顺序已经排好了。"

---

#### 6.5 ⚠️ D 靠你的外键名工作——外键名速查表

D 拿到你的 SQL 后会用 `ALTER TABLE` 修改你建好的外键约束，加上 `ON DELETE` 策略。他需要知道你给每个外键起的名字。下面是完整清单（你不需要额外写，SQL 里已经有了——给 D 看一眼就行）：

| 你的外键名 | 在哪个表 | 指向哪个表 |
|-----------|---------|-----------|
| `fk_major_department` | Major | Department |
| `fk_student_department` | Student | Department |
| `fk_student_major` | Student | Major |
| `fk_teacher_department` | Teacher | Department |
| `fk_course_department` | Course | Department |
| `fk_course_prereq` | Course | Course（自引用） |
| `fk_rule_course` | CourseLevelRule | Course |
| `fk_offering_course` | CourseOffering | Course |
| `fk_offering_teacher` | CourseOffering | Teacher |
| `fk_offering_semester` | CourseOffering | Semester |
| `fk_enrollment_student` | Enrollment | Student |
| `fk_enrollment_offering` | Enrollment | CourseOffering |
| `fk_schedule_offering` | Schedule | CourseOffering |
| `fk_schedule_classroom` | Schedule | Classroom |
| `fk_change_schedule` | ScheduleChange | Schedule |
| `fk_change_classroom` | ScheduleChange | Classroom |
| `fk_change_teacher` | ScheduleChange | Teacher |
| `fk_attendance_schedule` | Attendance | Schedule |
| `fk_attendance_student` | Attendance | Student |
| `fk_grade_student` | Grade | Student |
| `fk_grade_offering` | Grade | CourseOffering |
| `fk_grade_teacher` | Grade | Teacher |
| `fk_item_grade` | GradeItem | Grade |
| `fk_template_offering` | GradingTemplate | CourseOffering |
| `fk_thesis_student` | Thesis | Student |
| `fk_thesis_advisor` | Thesis | Teacher |
| `fk_thesis_reviewer` | Thesis | Teacher |
| `fk_project_course` | Project | Course |
| `fk_project_student` | Project | Student |
| `fk_project_advisor` | Project | Teacher |

> 共 30 条外键名。D 会在你的 SQL 里搜索 `CONSTRAINT fk_` 找到每一个外键的位置，然后补上 `ON DELETE ... ON UPDATE ...`。

---

### 这一步做完，你发给 D 的东西：

| 文件 | 内容 | 格式 |
|------|------|------|
| `schema_skeleton.sql` | 18 张表的完整 CREATE TABLE（有 PK + FK + 外键名，无 CHECK/触发器/ON DELETE） | UTF-8 文本 |
| `关系模式设计说明.md` | 设计理由、范式分析 | Markdown |

**D 拿到这两个文件后，打开 `schema_skeleton.sql`，直接在 FOREIGN KEY 行后面补 `ON DELETE CASCADE` 之类的东西。**

---

## 五、常见错误（提前避开）

| ❌ 错误 | ✅ 正确做法 |
|---------|-----------|
| 字段名和 A 的不一致 | 完全照抄 A 文档里的属性名，大小写都保持一致 |
| 枚举类型写成 VARCHAR 不加 CHECK | CHAR(4) + CHECK(gender IN ('男','女')) |
| 给学生表直接加"课程"字段 | 选课是 M:N，必须建 ENROLLMENT 中间表 |
| 外键忘了写指向哪张表 | 必须写完整：`FK → DEPARTMENT(dept_id)` |
| 1:1 忘了加 UNIQUE | THESIS.stu_id 必须标 UNIQUE |
| 把 FK 直接当 PK 用 | ENROLLMENT 有自己的 enroll_id PK，FK 只是 FK |
| 用 FLOAT 存金额/学分/成绩 | 用 DECIMAL，FLOAT 有精度问题 |
| 关系模式里漏了某张表 | 最后数一遍，必须 18 张 |

---

## 六、交付前自查清单

- [ ] **18 张表**一张不少？
- [ ] 每张表**主键明确**标注？
- [ ] **28 条外键**全部标注了 FK → 目标表(目标字段)？
- [ ] 所有外键方向正确（N 端放 FK）？
- [ ] M:N 的 ENROLLMENT 中间表存在？
- [ ] 1:1 的 THESIS.stu_id 标了 UNIQUE？
- [ ] 自引用 COURSE.prereq_id 允许为空？
- [ ] 字段类型用了 CHAR/VARCHAR/INT/DECIMAL/DATE/BOOLEAN，不用"字符串""数字"这种模糊写法？
- [ ] CHECK/DEFAULT/UNIQUE 等约束写了？
- [ ] 设计说明文档写了（至少 2-5 节）？
- [ ] 和 B 的图、A 的文档三方对齐——同一个字段在三份文档里的名字一模一样？

---

## 七、建议时间

| 步骤 | 用时 | 节点 |
|------|------|------|
| 等 B 交付 + 读 B 的图 | — | 第 4 天拿到 |
| 安装 PostgreSQL + pgAdmin | 0.5h | 第 4 天 |
| 第 1 步：列 18 张表名 | 0.5h | 第 4 天 |
| 第 2 步：逐张写关系模式 | 4-5h | 第 4-5 天 |
| 第 3 步：验证 FK | 1h | 第 5 天 |
| 第 4 步：特殊约束 | 0.5h | 第 5 天 |
| 第 5 步：设计说明 | 1-1.5h | 第 5 天 |
| 第 6 步：PostgreSQL 实操 — 写骨架 SQL + 跑通 + 导出 → **当天发给 D** | 2h | 第 5 天 |
| 关系模式文档发给 E | 0.5h | 第 5-6 天 |

---

## 八、和 D、E 的对接

### → D：把 SQL 骨架传过去

```
你的 SQL 骨架                     D 拿到后往上加
─────────────────────────────────────────────────
CREATE TABLE Student (            CREATE TABLE Student (
    stu_id CHAR(10) PK,              stu_id CHAR(10) PRIMARY KEY,
    stu_name VARCHAR(30),            stu_name VARCHAR(30) NOT NULL,
    gender CHAR(1),                  gender CHAR(1) CHECK(gender IN ('男','女')),
    dept_id CHAR(4),                 dept_id CHAR(4) NOT NULL,
    FOREIGN KEY (dept_id)            FOREIGN KEY (dept_id)
        REFERENCES Department(dept_id)   REFERENCES Department(dept_id)
);                                      ON DELETE RESTRICT
                                        ON UPDATE CASCADE
                                    );
```

你的工作到此为止。CHECK、ON DELETE、触发器全交给 D。**你写完骨架当天就发给 D**，别让 D 空等。

### → E：你的关系模式表格，E 会直接复制进报告第三章

---

## 九、参考资源

| 资源 | 位置 | 怎么用 |
|------|------|--------|
| A 需求分析（第 2 章属性表） | `需求分析说明书.md` | 每张表的字段名/类型/约束的来源 |
| A 需求分析（第 6 章关系草图） | `需求分析说明书.md` | 确认 30 条关系都在你的 FK 列表里 |
| B 的 E-R 图 | B 交付的图 | 确认 FK 方向和基数一致 |
| 表结构图 | `表结构图.md` | 第五章关系矩阵——速查谁连着谁 |

---

> 📌 **一句话记住你的任务**：把 B 的图翻译成 18 张表的「表头设计」，让任何一个程序员拿着它就能写出 CREATE TABLE 语句。你写的就是数据库的施工图纸——精确到每一个字段的类型和长度。
