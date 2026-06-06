一、概述

此数据库按业务边界拆分为三大子系统独立ER模型：①基础学籍师资子系统、②选课排课成绩子系统、③毕设实践考勤调课子系统，基于Draw.io规范绘制ER图，
采用矩形实体+带两端基数连线、主键加粗PK_、外键FK_标注规范，多对多关系全部拆解为中间实体，无原生多对多连线；三大子系统实体分别使用浅绿、浅橙、浅紫底色区分，
跨子系统关联采用虚线连接线，本报告完整记录实体结构、属性、主键外键、实体关系与基数约束，作为后续数据表SQL建表、约束编写、索引设计的基准文档。



二、统一绘图与数据库设计规范（Draw.io+SQL通用）


（一）ER绘图规范
1. 实体：基础矩形，框内排版：首行实体名称，逐行罗列属性； PK_xxx 为主键字段（Draw.io字体加粗）， FK_xxx 为外键字段；
2. 关联连线：实线为本子系统内部关联、虚线为跨子系统引用关联；连线左右两端标注基数，图中基数取值： 1、0..1、0..n ； 1 — 0..n （左侧箭头为 || ，右侧为三叉）； 
   0..1一0..n ；（左侧箭头为 | ，右侧为三叉），所有多对多逻辑通过新建中间实体拆分，消除多对多连线；
3. 配色规范
- 子系统1（学籍师资）：实体填充浅绿色；
- 子系统2（选课成绩）：实体填充浅橙色；
- 子系统3（毕设考勤）：实体填充浅紫色；
- 跨子系统引用实体（STUDENT/TEACHER/COURSE/SCHEDULE等）：灰色虚线引用。

（二）SQL设计前置约定
1. 主键：单字段主键/复合主键均通过PK标识，建表添加PRIMARY KEY约束；
2. 外键：FK字段参照对应主表主键，后续SQL添加FOREIGN KEY外键约束；
3. 属性括号内为字段数据类型，SQL建表可直接沿用字段名与类型。


 
三、分系统ER详细设计说明


子系统1：基础学籍师资管理子系统（浅绿色实体）
1. 实体清单与属性明细

DEPARTMENT 院系                                             实体名称
PK_dept_id(CHAR4)                                           主键（PK加粗）
dept_name(VARCHAR50)                                        普通属性/外键
院系唯一编码为主键，存储院系名称                               备注

MAJOR 专业 
PK_major_id(INT)
FK_dept_id(CHAR4)、major_name(VARCHAR50) 
FK_dept_id关联院系主键，专业归属院系 

STUDENT 学生 
PK_stu_id(CHAR10) 
stu_name、gender、birth、id_card、enroll_year、edu_level、dept_id、major_name、cls_name、study_status、phone、email 
dept_id逻辑关联院系，专业名称冗余存储 

TEACHER 教师 
PK_tea_id(CHAR8) 
tea_name、gender、birth、title、dept_id、edu_level、phone、email 
dept_id归属对应院系 
 
2. 实体关系+两端基数（实线连线，本子系统内关联）
- DEPARTMENT（1｜0..n）MAJOR：一个院系可包含0或多个专业，一个专业仅归属1个院系；
- DEPARTMENT（1｜0..n）STUDENT：一个院系容纳0~N名学生，一名学生隶属于单一院系；
- DEPARTMENT（1｜0..n）TEACHER：一个院系配置0~N名教师，一名教师归属单一院系；
- MAJOR（1｜0..n）STUDENT：一个专业招收多名学生，一名学生归属单一专业。

 
子系统2：选课排课成绩管理子系统（浅橙色实体，核心业务）
（跨子系统关联：引用子1的TEACHER、STUDENT，采用灰色虚线连线）
1. 实体清单与属性明细
 
COURSE课程                                                                                                      实体名称
PK_course_id(CHAR8)                                                                                             主键
course_name、course_type、credit_default、class_hour、dept_id、FK_pre_course_id、course_desc                     属性/外键
FK_pre_course_id关联前置课程ID                                                                                   备注

COURSE_EDU_RULE课程学分规则 
PK(course_id,edu_level)复合主键 
FK_course_id、edu_level、course_nature、credit
按课程+学历分层配置学分 

COURSE_OFFERING开课教学班 
PK_offering_id(CHAR12)
FK_course_id、FK_tea_id、semester、class_seq、max_stu、curr_stu、teach_lang、has_exp
FK_course_id绑定课程、FK_tea_id绑定授课教师 

CLASSROOM教室
PK_room_id(CHAR6) 
room_name、capacity、room_type、has_projector、proj_count、desk_movable、campus 
无

SCHEDULE排课记录 
PK_sch_id(INT) 
FK_offering_id、FK_room_id、week_day、start_section、end_section、start_week、end_week、week_flag、sch_type、remark
绑定教学班+上课教室 

ENROLL选课中间表 
PK(enroll_id) 
FK_stu_id、FK_offering_id、enroll_time、enroll_status、course_nature
拆分学生与教学班多对多关系 

SCORE_TEMPLATE评分模板 
PK_temp_id(INT) 
FK_offering_id、item_name、weight、score_standard
单个教学班一套评分细则 

GRADE_MAIN成绩主表 
PK_grade_id(INT) 
FK_stu_id、FK_offering_id、total_score、grade_level、is_makeup、makeup_score、FK_input_tea_id、input_time、remark
FK_input_tea_id为录入成绩教师 

GRADE_ITEM成绩分项 
PK_item_id(INT) 
FK_grade_id、item_name、item_score、weight
成绩主表拆分各项得分 
 
2. 实体关系与基数标注
（1）内部实线关联
- COURSE（1｜0..n）COURSE_EDU_RULE：一门课程可配置多条不同学历学分规则；
- COURSE（1｜0..n）COURSE_OFFERING：一门课程多个学期可多次开课；
- COURSE_OFFERING（1｜0..n）SCHEDULE：单个教学班多条排课记录；
- CLASSROOM（1｜0..n）SCHEDULE：一间教室可安排多个时段排课；
- COURSE_OFFERING（1｜0..n）ENROLL：一个教学班多名学生选课；
- STUDENT（1｜0..n）ENROLL：一名学生可选多门教学班课程（虚线跨子系统）；
- COURSE_OFFERING（1｜0..n）SCORE_TEMPLATE：一个教学班配套一套评分模板；
- COURSE_OFFERING（1｜0..n）GRADE_MAIN：一个教学班对应多名学生成绩；
- GRADE_MAIN（1｜0..n）GRADE_ITEM：一条总成绩拆分成多条分项成绩。
 
（2）跨子系统虚线关联
- STUDENT（1｜0..n）ENROLL：一名学生可选多门教学班课程；
- TEACHER（1｜0..n）COURSE_OFFERING：一名教师承担多个教学班授课；
- TEACHER（1｜0..n）GRADE_MAIN：一名教师录入多名学生成绩。
 

子系统3：毕设实践考勤调课子系统（浅紫色实体）
（全部关联STUDENT/TEACHER/SCHEDULE/CLASSROOM/COURSE均为跨子系统灰色虚线引用）
 
1. 实体清单与属性明细
 
GRADUATION_THESIS毕业设计                                                                                                             实体名称
PK_thesis_id(INT)                                                                                                                    主键
FK_stu_id、FK_adv_tea_id、FK_review_tea_id、title、source、start_date、defend_date、final_score、grade_rank、defend_status             属性
adv导师、review评阅教师                                                                                                                备注

PROJECT_DESIGN项目设计 
PK_proj_id(INT) 
proj_name、FK_course_id、FK_stu_id、FK_adv_tea_id、proj_type、submit_date、score、remark
关联课程、学生、指导老师 

ATTENDANCE考勤记录 
PK_att_id(INT) 
FK_sch_id、FK_stu_id、class_date、att_status、remark
绑定排课+学生日常考勤 

ADJUST_SCHED调课申请 
PK_adj_id(INT) 
FK_origin_sch_id、FK_new_room_id、FK_apply_tea_id、adj_type、origin_date、new_date、new_start_sec、new_end_sec、adj_reason、audit_status、apply_time
原排课、新教室、申请教师 
 
2. 实体关系与基数（全部跨库虚线关联）
- STUDENT（1｜0..n）GRADUATION_THESIS：1名学生至多1份毕设，1个毕设归属单个学生；
- TEACHER（1｜0..n）GRADUATION_THESIS（导师）：一名导师指导多名毕业生；
- TEACHER（1｜0..n）GRADUATION_THESIS（评阅）：一名评阅教师评审多份毕设；
- STUDENT（1｜0..n）PROJECT_DESIGN：一名学生参与多项项目设计；
- TEACHER（1｜0..n）PROJECT_DESIGN：一名老师指导多项项目；
- COURSE（0..1｜0..n）PROJECT_DESIGN：项目可关联/不关联课程，一门课程衍生多个项目；
- SCHEDULE（1｜0..n）ATTENDANCE：一条排课对应多名学生考勤；
  STUDENT（1｜0..n）ATTENDANCE：学生多条上课考勤；
- SCHEDULE（1｜0..n）ADJUST_SCHED：一条原排课可发起多次调课；
  TEACHER（1｜0..n）ADJUST_SCHED：一名教师提交多条调课；
  CLASSROOM（0..1｜0..n） ADJUST_SCHED：调课可选新教室/不换教室，一间教室承接多条调课。
 
 
 
四、ER设计向SQL建表落地指引
1. 主键落地规则
- 单字段PK：SQL采用 PRIMARY KEY(字段名) ；
- 复合主键（COURSE_EDU_RULE）： PRIMARY KEY(course_id,edu_level) ；
- 自增主键（INT类型PK如grade_id、att_id）：建表配置自增属性。
 
2. 外键落地规则
所有 FK_* 字段创建外键约束，参照对应主表主键：
例：MAJOR.FK_dept_id → DEPARTMENT.PK_dept_id；ENROLL.FK_stu_id → STUDENT.PK_stu_id。
 
3. 关系与基数对应SQL约束
-  1 — 0..n 一对多：多方表存放FK，一方无外键，通过外键实现一对多约束；
-  0..1 — 0..n ：多方存FK，FK字段允许NULL实现可选关联；
- 多对多全部拆中间表（ENROLL），中间表存储两端FK，完成多对多拆分。




//     陈奕鸿
//     2026.06.05
