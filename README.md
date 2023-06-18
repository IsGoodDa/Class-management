### 班级管理模块是发展性测评系统中的一个重要部分，提供了对班级信息和学生组织的管理功能。用户可以方便地添加和管理班级信息，包括班级名称、学生人数等，同时可以对学生组织进行细致的管理，例如添加和删除班干部、记录学生的考勤情况等，以便更好地管理和了解班级的状态和运营情况。

## The Class Management module is an important part of the developmental assessment system, providing management functions for class information and student organization. Users can easily add and manage class information, including class name, number of students, etc., and can carefully manage student organizations, such as adding and deleting class leaders, recording students' attendance, etc., so as to better manage and understand the status and operation of the class.

### 以下是班级管理模块的Java代码示例：

## Here is a Java code example for the class management module:


package com.ruoyi.kx.controller;

import com.ruoyi.common.annotation.Excel;
import com.ruoyi.common.annotation.Log;
import com.ruoyi.common.core.controller.BaseController;
import com.ruoyi.common.core.domain.AjaxResult;
import com.ruoyi.common.core.domain.entity.SysRole;
import com.ruoyi.common.core.domain.entity.SysUser;
import com.ruoyi.common.core.page.TableDataInfo;
import com.ruoyi.common.enums.BusinessType;
import com.ruoyi.common.utils.StringUtils;
import com.ruoyi.common.utils.poi.ExcelUtil;
import com.ruoyi.common.utils.security.Md5Utils;
import com.ruoyi.kx.bean.KxStudentBean;
import com.ruoyi.kx.bean.KxStudentStatusBean;
import com.ruoyi.kx.bean.ScoreSort;
import com.ruoyi.kx.domain.*;
import com.ruoyi.kx.mapper.*;
import com.ruoyi.kx.service.IKxStudentService;
import com.ruoyi.system.mapper.SysUserRoleMapper;
import com.ruoyi.system.service.ISysUserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.ModelMap;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.math.BigDecimal;
import java.text.DecimalFormat;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

/**
 * 学生信息Controller
 *
 * @author huang
 * @date 2023-01-14
 */
@Controller
@RequestMapping("/kx/student")
public class KxStudentController extends BaseController {
    private final String prefix = "kx/student";
    @Autowired
    KxScoreRecordMapper kxScoreRecordMapper;
    @Autowired
    KxRateOftenMapper kxRateOftenMapper;
    @Autowired
    KxMarkMapper kxMarkMapper;
    @Autowired
    KxStudentMapper kxStudentMapper;
    @Autowired
    private IKxStudentService kxStudentService;
    @Autowired
    private ISysUserService iSysUserService;
    @Autowired
    private KxTeacherMapper kxTeacherMapper;
    @Autowired
    private KxClassesMapper kxClassesMapper;

    @Autowired
    KxTimingMapper kxTimingMapper;

    /**
     * 查询学生信息列表
     */
    //@RequiresPermissions("kx:student:list")
    @PostMapping("/list")
    @ResponseBody
    public TableDataInfo list(KxStudent kxStudent) {
        KxTeacher kxTeacher = kxTeacherMapper.selectKxTeacherByUserId(getUserId());
        if (StringUtils.isNotNull(kxTeacher)) {
            KxClasses kxClasses = kxClassesMapper.selectKxClassesByTeacherId(kxTeacher.getId());
            if (StringUtils.isNotNull(kxClasses)) {
                kxStudent.setClassId(kxClasses.getId());
            }
        }
        startPage();
        logger.info("kxStudent:{}", kxStudent.toString());
        List<KxStudent> list = kxStudentService.selectKxStudentList(kxStudent);
        return getDataTable(list);

    }

    /**
     * 生成学生信息列表
     */
    //@RequiresPermissions("kx:student:list")
    @PostMapping("/select")
    @ResponseBody
    public TableDataInfo select(KxStudent kxStudent) {
        List<KxStudent> list = kxStudentService.selectKxStudentList(kxStudent);
    /*    list.sort((o1, o2) -> {
            KxRateOften s1 = o1.getOften();
            KxRateOften s2 = o2.getOften();
            if (s1 != null && s2 != null) {
                return s2.getScore().compareTo(s1.getScore()); // 降序排序
            }
            if (s1 != null) {
                return -1; // s2 为 null，将 o1 放到前面
            }
            if (s2 != null) {
                return 1; // s1 为 null，将 o2 放到前面
            }
            return 0; // s1 和 s2 都为 null，保持原有顺序
        });*/
        return getDataTable(list);
    }

    //@RequiresPermissions("kx:student:view")
    @GetMapping()
    public String student(ModelMap modelMap) {
        modelMap.put("loginName", getSysUser().getLoginName());
        List<KxClasses> kxClasses = kxClassesMapper.selectKxClassesList(null);
        kxClasses = kxClasses.subList(1, kxClasses.size());
        logger.info("kxClass111:{}", kxClasses);

        boolean flag = false;
        for (SysRole role : getSysUser().getRoles()) {
            Long roleId = role.getRoleId();
            if (roleId == 5) {
                flag = true;
                break;
            }
        }
        if (!flag && !getSysUser().getLoginName().equals("admin")) {
            KxTeacher kxTeacher = kxTeacherMapper.selectKxTeacherByUserId(getUserId());
            kxClasses = new ArrayList<>();
            kxClasses.add(kxClassesMapper.selectKxClassesByTeacherId(kxTeacher.getId()));
        }

        modelMap.put("classes", kxClasses);
        logger.info("flag111:{}", flag);
        modelMap.put("flag", flag);

        return prefix + "/student";
    }

    /**
     * 更新班级成绩
     */
    //@RequiresPermissions("kx:student:list")
    @GetMapping("/selectScore/{classId}")
    @ResponseBody
    public TableDataInfo selectScore(@PathVariable("classId") Long classId) {
        KxStudent kxStudent = new KxStudent();
        kxStudent.setClassId(classId);
        ArrayList<KxStudent> list2 = new ArrayList<>();
        List<KxStudent> list = kxStudentService.selectKxStudentList(kxStudent);
        for (KxStudent student : list) {
            list2.add(getSuZhi(student));
        }
        return getDataTable(list2);
    }

    @Autowired
    KxParentMapper kxParentMapper;

    public KxStudent getSuZhi(KxStudent student) {
        char firstChar = student.getClassName().charAt(0);

        //获取期末评测所有分
        BigDecimal subjectTotalScore = null;
        KxMark kxMark = student.getKxMark();
        if (kxMark != null) {
            BigDecimal totalScore = BigDecimal.ZERO;
            BigDecimal[] scores = {kxMark.getChineseScore(), kxMark.getMathScore(), kxMark.getEnglishScore(), kxMark.getScienceScore(), kxMark.getPhysicalScore(), kxMark.getArtScore(), kxMark.getMusicScore(), kxMark.getMoralScore()};
            for (BigDecimal score : scores) {
                BigDecimal gradeScore;
                if (score != null) {
                    if (firstChar != '一' && firstChar != '二') {
                        if (score.compareTo(BigDecimal.valueOf(90)) >= 0) {
                            gradeScore = BigDecimal.valueOf(5);
                        } else if (score.compareTo(BigDecimal.valueOf(80)) >= 0) {
                            gradeScore = BigDecimal.valueOf(4);
                        } else if (score.compareTo(BigDecimal.valueOf(70)) >= 0) {
                            gradeScore = BigDecimal.valueOf(3);
                        } else if (score.compareTo(BigDecimal.valueOf(60)) >= 0) {
                            gradeScore = BigDecimal.valueOf(2);
                        } else if (score.compareTo(BigDecimal.valueOf(0)) > 0) {
                            gradeScore = BigDecimal.valueOf(1);
                        } else {
                            //英语分0分就是没考 可能是三年级以下
                            gradeScore = BigDecimal.valueOf(0);
                        }
                    } else {
                        gradeScore = BigDecimal.valueOf(score.floatValue());
                    }
                    totalScore = totalScore.add(gradeScore);
                }
            }
            subjectTotalScore = totalScore;
        }

        Long kxTimingId = kxTimingMapper.selectKxTimingByNow().getId();
        //获取即时评平均分数
        float avOftenScore = kxRateOftenMapper.selectKxScoreOftenAvRate(student.getId(), kxTimingId);
        //获取评价记录平均分数
        ScoreSort scoreSort = kxScoreRecordMapper.selectKxScoreRecordBySort(student.getId(), "E", 0L, kxTimingId);
        KxStudent kxStudent = new KxStudent();
        kxStudent.setId(student.getId());
        if (subjectTotalScore != null) {

            kxStudent.setScore(scoreSort.getScore() + avOftenScore + subjectTotalScore.floatValue());
        } else {
            kxStudent.setScore(scoreSort.getScore() + avOftenScore);

        }

        kxStudentMapper.updateKxStudent(kxStudent);
        KxStudent temp = kxStudentMapper.selectKxStudentById(kxStudent.getId());
        temp.setTotalScore(kxRateOftenMapper.selectKxScoreOftenTotalRate2(student.getId(), kxTimingId));
        temp.setAvScore(changeFloat(avOftenScore));
        return temp;
    }

    /**
     * 查询学生信息列表
     */
    //@RequiresPermissions("kx:student:list")
    @GetMapping("/scoreRank/{classId}")
    @ResponseBody
    public TableDataInfo scoreRank(@PathVariable("classId") Long classId) {
        List<KxStudent> list = kxStudentMapper.selectKxStudentByScore(classId);
        return getDataTable(list);
    }

    /**
     * 查询学生信息列表
     */
    @PostMapping("/select2")
    @ResponseBody
    public AjaxResult select2(KxStudent kxStudent) {
        List<KxStudent> list = kxStudentService.selectKxStudentList(kxStudent);

        if (list.size() == 1) {
            return success().put("data", list.get(0));
        }
        return error("未找到数据");
    }

    /**
     * 导出学生信息列表
     */
    //@RequiresPermissions("kx:student:export")
    @Log(title = "学生信息", businessType = BusinessType.EXPORT)
    @PostMapping("/export")
    @ResponseBody
    public AjaxResult export(KxStudent kxStudent) {
        List<KxStudent> list = kxStudentService.selectKxStudentList(kxStudent);

        ExcelUtil<KxStudent> util = new ExcelUtil<KxStudent>(KxStudent.class);
        return util.exportExcel(list, "学生信息数据");
    }

    private void setCombo(ArrayList<String> list) throws Exception {
        // 通过反射 获取目标实体类的属性成员-即办公室号号字段
        Field file = KxStudentBean.class.getDeclaredField("className");
        // 获取该字段的上叫Excel的注解
        Excel annotation = file.getAnnotation(Excel.class);

        InvocationHandler h = Proxy.getInvocationHandler(annotation);

        Field hField = h.getClass().getDeclaredField("memberValues");
        // 设置私有可访问
        hField.setAccessible(true);

        Map memberValues = (Map) hField.get(h);

        // 集合转数组
        String[] combo = list.toArray(new String[0]);
        // 修改属性值
        memberValues.put("combo", combo);
    }


    @Log(title = "学生信息", businessType = BusinessType.IMPORT)
    //@RequiresPermissions("kx:student:import")
    @PostMapping("/importData")
    @ResponseBody
    public AjaxResult importData(MultipartFile file, boolean updateSupport) throws Exception {
        logger.info("班干:{}", file.getOriginalFilename());
        if (file.getOriginalFilename().contains("班干")) {
            ExcelUtil<KxStudentStatusBean> util = new ExcelUtil<>(KxStudentStatusBean.class);
            List<KxStudentStatusBean> kxStudentList = util.importExcel(file.getInputStream());
            String message = kxStudentService.importKxStudent2(kxStudentList, updateSupport, getLoginName(), getUserId());
            return AjaxResult.success(message);
        } else {
            ExcelUtil<KxStudentBean> util = new ExcelUtil<>(KxStudentBean.class);
            List<KxStudentBean> kxStudentList = util.importExcel(file.getInputStream());
            String message = kxStudentService.importKxStudent(kxStudentList, updateSupport, getLoginName(), getUserId());
            return AjaxResult.success(message);
        }

    }

    //@RequiresPermissions("kx:student:view")
    @GetMapping("/importTemplate")
    @ResponseBody
    public AjaxResult importTemplate() {
        List<KxClasses> kxClasses = kxClassesMapper.selectKxClassesList(null);
        kxClasses = kxClasses.subList(1, kxClasses.size());
        ArrayList<String> strings2 = new ArrayList<>();

        for (KxClasses kxClass : kxClasses) {
            strings2.add(kxClass.getName());
        }
        try {
            setCombo(strings2);
        } catch (Exception e) {
            e.printStackTrace();
        }
        ExcelUtil<KxStudentBean> util = new ExcelUtil<KxStudentBean>(KxStudentBean.class);
        return util.importTemplateExcel(" 学生信息数据");
    }

    public Float changeFloat(Float number) {
        DecimalFormat decimalFormat = new DecimalFormat("#.#");
        String formattedNumber = decimalFormat.format(number);
        return Float.parseFloat(formattedNumber);
    }

    /**
     * 新增学生信息
     */
    @GetMapping("/add")
    public String add(ModelMap modelMap) {
        modelMap.put("loginName", getSysUser().getLoginName());
        List<KxClasses> kxClasses = kxClassesMapper.selectKxClassesList(null);
        kxClasses = kxClasses.subList(1, kxClasses.size());

        boolean flag = false;
        for (SysRole role : getSysUser().getRoles()) {
            Long roleId = role.getRoleId();
            if (roleId == 5) {
                flag = true;
                break;
            }
        }
        modelMap.put("flag", flag);
        if (!flag && !getSysUser().getLoginName().equals("admin")) {
            KxTeacher kxTeacher = kxTeacherMapper.selectKxTeacherByUserId(getUserId());
            kxClasses = new ArrayList<>();
            kxClasses.add(kxClassesMapper.selectKxClassesByTeacherId(kxTeacher.getId()));
        }
        modelMap.put("classes", kxClasses);
        ArrayList<KxParent> kxParents = new ArrayList<>();
        kxParents.add(new KxParent());
        kxParents.add(new KxParent());
        modelMap.put("parents", kxParents);


        return prefix + "/add";
    }

    /**
     * 新增保存学生信息
     */
    //@RequiresPermissions("kx:student:add")
    @Log(title = "学生信息", businessType = BusinessType.INSERT)
    @PostMapping("/add")
    @ResponseBody
    public AjaxResult addSave(KxStudent kxStudent) {

        if (StringUtils.isEmpty(kxStudent.getCard())) {
            return error("身份证号不能为空");
        }


        if (StringUtils.isEmpty(kxStudent.getName())) {
            return error("请输入学生姓名");
        }

        if (StringUtils.isEmpty(kxStudent.getNo().toString())) {
            return error("请设置班级编号");
        }

        if (StringUtils.isEmpty(kxStudent.getCard())) {
            return error("请设置身份证号");
        }


        KxStudent student = kxStudentMapper.selectKxStudentByCard(kxStudent.getCard());
        if (student != null) {
            return error("已有相同身份证号的学生存在");
        }

        int gender;
        if (kxStudent.getCard().length() == 18) {
            //如果身份证号18位，取身份证号倒数第二位
            char c = kxStudent.getCard().charAt(kxStudent.getCard().length() - 2);
            gender = Integer.parseInt(String.valueOf(c));
        } else {
            //如果身份证号15位，取身份证号最后一位
            char c = kxStudent.getCard().charAt(kxStudent.getCard().length() - 1);
            gender = Integer.parseInt(String.valueOf(c));
        }
        if (gender % 2 == 1) {
            kxStudent.setImagePath("/profile/avatar/nan.png");

        } else {
            kxStudent.setImagePath("/profile/avatar/nv.png");
        }

        kxStudentService.insertKxStudent(kxStudent);
        for (KxParent parent : kxStudent.getParents()) {
            parent.setStudentId(kxStudent.getId());
            kxParentMapper.insertKxParent(parent);
        }
        return success();
    }

    /**
     * 修改学生信息
     */
    //@RequiresPermissions("kx:student:edit")
    @GetMapping("/edit/{id}")
    public String edit(@PathVariable("id") Long id, ModelMap modelMap) {
        modelMap.put("loginName", getSysUser().getLoginName());
        KxStudent kxStudent = kxStudentService.selectKxStudentById(id);
        modelMap.put("kxStudent", kxStudent);
        List<KxClasses> kxClasses = kxClassesMapper.selectKxClassesList(null);
        kxClasses = kxClasses.subList(1, kxClasses.size());

        boolean flag = false;
        for (SysRole role : getSysUser().getRoles()) {
            Long roleId = role.getRoleId();
            if (roleId == 5) {
                flag = true;
                break;
            }
        }
        modelMap.put("flag", flag);
        if (!flag && !getSysUser().getLoginName().equals("admin")) {
            KxTeacher kxTeacher = kxTeacherMapper.selectKxTeacherByUserId(getUserId());
            kxClasses = new ArrayList<>();
            kxClasses.add(kxClassesMapper.selectKxClassesByTeacherId(kxTeacher.getId()));
        }
        modelMap.put("classes", kxClasses);
        List<KxParent> kxParents = kxParentMapper.selectKxParentByStudentId(id);

        if (kxParents.size() == 0) {
            kxParents.add(new KxParent());
            kxParents.add(new KxParent());
        } else if (kxParents.size() == 1) {
            kxParents.add(new KxParent());
        }

        kxStudent.setParents(kxParents);
        logger.info("parents111:{}", kxParents);
        modelMap.put("parents", kxParents);
        return prefix + "/edit";
    }


    @Autowired
    SysUserRoleMapper sysUserRoleMapper;

    /**
     * 修改保存学生信息
     */
    //@RequiresPermissions("kx:student:edit")
    @Log(title = "学生信息", businessType = BusinessType.UPDATE)
    @PostMapping("/edit")
    @ResponseBody
    public AjaxResult editSave(KxStudent kxStudent) {
        //logger.info("MkxStudent:{}", kxStudent.toString());
        logger.info("kxStudent111:{}", kxStudent);
        if (kxStudent.getPassword() == null) {
            kxStudent.setPassword("888888");
        }

        if (kxStudent.getParents() != null) {
            for (KxParent parent : kxStudent.getParents()) {
                parent.setStudentId(kxStudent.getId());
                if (StringUtils.isNotEmpty(parent.getName()) && StringUtils.isNotEmpty(parent.getPhone())) {
                    kxParentMapper.insertKxParent(parent);
                } else {
                    return error("家长信息不能为空");
                }
            }
        }


        KxStudent editStudent = kxStudentMapper.selectKxStudentById(kxStudent.getId());
        kxStudent.setScore(editStudent.getScore());

        KxStudent noStudent = kxStudentMapper.selectKxStudentByNo(String.valueOf(kxStudent.getNo()), kxStudent.getClassId());
        if (noStudent != null) {
            if (!kxStudent.getId().equals(noStudent.getId())) {
                return error("学籍编号在该班级已重复");
            }
        }


        if (kxStudent.getStatus() != null && !kxStudent.getStatus().equals("学生")) {
            KxStudent newKxStudent = new KxStudent();
            newKxStudent.setName(kxStudent.getName());
            newKxStudent.setClassId(kxStudent.getClassId());
            newKxStudent.setStatus(kxStudent.getStatus());
            KxStudent statusStudent = kxStudentMapper.selectKxStudentByStatus(newKxStudent.getClassId(), newKxStudent.getStatus());

            if (!editStudent.getStatus().equals("学生") && !kxStudent.getStatus().equals(editStudent.getStatus())) {
                return error("该学生已担任职务");
            }

            if (statusStudent != null && !statusStudent.getId().equals(kxStudent.getId())) {
                statusStudent.setStatus("学生");
                kxStudentMapper.updateKxStudent(statusStudent);
                kxStudentMapper.updateKxStudent(kxStudent);
                //清除学生特殊身份和注销系统账户
                SysUser tempSysUser = iSysUserService.selectUserById(statusStudent.getUserId());
                if (tempSysUser != null) {
                    iSysUserService.deleteUserById(tempSysUser.getUserId());
                    statusStudent.setUserId(0L);
                    kxStudentMapper.updateKxStudent(statusStudent);
                }

                SysUser sysUser = new SysUser();
                sysUser.setLoginName(kxStudent.getName());
                sysUser.setUserName(kxStudent.getName());
                sysUser.setPassword(Md5Utils.hash(kxStudent.getName() + kxStudent.getPassword()));
                sysUser.setDeptId(102L);
                iSysUserService.insertUser(sysUser);
                SysUser user = iSysUserService.selectUserByLoginName(kxStudent.getName());

                iSysUserService.insertUserAuth(user.getUserId(), new Long[]{3L});
                kxStudent.setUserId(user.getUserId());
                kxStudent.setPassword(String.valueOf(kxStudent.getPassword()));

//                return error("班级已有" + kxStudent.getStatus());
            }
        }

        if (kxStudent.getStatus() != null && !editStudent.getStatus().equals("学生") && kxStudent.getStatus().equals("学生")) {
            //将班干设置为普通学生 清除学生特殊身份和注销系统账户
            SysUser tempSysUser = iSysUserService.selectUserById(editStudent.getUserId());
            if (tempSysUser != null) {
                iSysUserService.deleteUserById(tempSysUser.getUserId());
            }
            editStudent.setUserId(0L);
            kxStudentMapper.updateKxStudent(editStudent);
            kxStudentMapper.updateKxStudentByStatus(editStudent);

        } else if (kxStudent.getStatus() != null && editStudent.getStatus().equals("学生") && !kxStudent.getStatus().equals("学生")) {
            SysUser sysUser = new SysUser();

            sysUser.setLoginName(kxStudent.getName());
            sysUser.setUserName(kxStudent.getName());
            sysUser.setPassword(Md5Utils.hash(kxStudent.getName() + kxStudent.getPassword()));
            sysUser.setDeptId(102L);

            iSysUserService.insertUser(sysUser);

            SysUser user = iSysUserService.selectUserByLoginName(kxStudent.getName());
            iSysUserService.insertUserAuth(user.getUserId(), new Long[]{3L});
            kxStudent.setUserId(user.getUserId());
            kxStudent.setPassword(String.valueOf(kxStudent.getPassword()));
        }


        SysUser sysUser = iSysUserService.selectUserById(editStudent.getUserId());
        if (sysUser != null) {
            logger.info("密码修改成功:{}", sysUser);
            sysUser.setPassword(Md5Utils.hash(kxStudent.getName() + kxStudent.getPassword()));
            iSysUserService.updateUser(sysUser);
        }

        SysUser user = iSysUserService.selectUserById(editStudent.getUserId());
        if (user != null) {
            user.setUserName(kxStudent.getName());
            user.setLoginName(kxStudent.getName());
            user.setPassword(Md5Utils.hash(kxStudent.getName() + kxStudent.getPassword()));
            iSysUserService.updateUser(user);
        }

        logger.info("userStudent111:{}", kxStudent);
        return toAjax(kxStudentService.updateKxStudent(kxStudent));
    }

    /**
     * 删除学生信息
     */
    //@RequiresPermissions("kx:student:remove")
    @Log(title = "学生信息", businessType = BusinessType.DELETE)
    @PostMapping("/remove")
    @ResponseBody
    public AjaxResult remove(String ids) {
//        iSysUserService.deleteUserByIds(ids);

        String[] str = ids.split(",");
        for (String s : str) {
            KxStudent kxStudent = kxStudentMapper.selectKxStudentById(Long.valueOf(s));
            SysUser user = iSysUserService.selectUserById(kxStudent.getUserId());
            if (user != null) {
                iSysUserService.deleteUserById(user.getUserId());
            }
        }

        return toAjax(kxStudentService.deleteKxStudentByIds(ids));
    }
}


    
#### 在这个示例中，我们使用 Flask 框架和 SQLAlchemy 作为 ORM 工具来搭建班级管理模块。其中，我们定义了一个 Student 数据库模型来存储学生信息，包括姓名、性别、班级名称、家长、老师和班长等。我们还定义了添加学生和学生列表等路由函数，并对其进行了相应的处理。最后，我们通过 app.run() 函数来启动应用程序并监听本地的 HTTP 请求。

In this example, we use the Flask framework and SQLAlchemy as ORM tools to build the class management module. In it, we define a Student database model to store student information, including name, gender, class name, parents, teachers, and class presidents. We also define routing functions such as adding students and student lists and handle them accordingly. Finally, we use the app.run() function to start the application and listen for local HTTP requests.

# <body>
  <header>
    <div class="logo">My Github Page</div>
    <nav>
      <a href="#">Home</a>
      <a href="#">About</a>
      <a href="#">Contact</a>
    </nav>
  </header>
  <h1>Welcome to My Github Page</h1>
  <footer>&copy; 2023 My Github Page. All rights reserved.</footer>
</body>
</html>

