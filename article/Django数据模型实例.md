Django使用模型(model)来定义数据，模型包含数据的字段以及行为信息，通常可以认为每个模型对应于一个数据库表。Django的模型系统实现了数据库的常见关系（一对一、一对多、多对多），本文主要介绍这三种关系模型的使用。

## 一对一关系

Django使用`OneToOneField`处理一对一关系。我们大学和地点来进行一对一关系的演示。一个地点可以是一所大学。

首先定义模型：

    :::python
    from django.db import models
    
    class Place(models.Model):
        name = models.CharField(max_length=50)
        address = models.CharField(max_length=50)
    
        def __str__(self):
            return self.name
        
    class University (models.Model):
        name = models.CharField(max_length=50)
        place = models.OneToOneField(
            Place,
            primary_key = True
        )
    
        def __str__(self):
            return self.name

之后就可以用Django的API来进行模型操作了：

    :::python
    > python manage.py shell
    >>> from myapp.models import Place, University

    # 创建地点
    >>> place1 = Place(name="Peking University",
            address="No.5 Yiheyuan Road Haidian District, Beijing")
    >>> place2 = Place(name="Tsinghua University",
            address="No.30 Shuangqing Road Haidian District, Beijing")
    >>> place3 = Place(name="Somewhere",
            address="Someroad Somecity")
    >>> place1.save()
    >>> place2.save()
    >>> place3.save()

    # 创建大学
    >>> university1 = University(name="Peking University", place=place1)
    >>> university2 = University(name="Tsinghua University", place=place2)
    >>> university1.save()
    >>> university2.save()

    # 地点和大学模型可以相互访问
    >>> place1.university
    <University: Peking University>
    >>> university1.place
    <Place: Peking University>
    >>> place2.university.name
    'Tsinghua University'
    >>> university2.place.address
    'No.30 Shuangqing Road Haidian District, Beijing'

    # 地点3目前还没有大学
    >>> hasattr(place3, 'university')
    False
    # 使用赋值语句为清华大学设置新地点
    >>> university2.place = place3
    >>> university2.save()
    >>> hasattr(place3, 'university')
    True
    >>> place3.university
    <University: Tsinghua University>

    # 注意place为University的主键，因此上述操作创建了一个新的大学对象
    >>> University.objects.all()
    <QuerySet [<University: Peking University>, <University: Tsinghua University>, <University: Tsinghua University>]>

## 一对多关系

Django使用`ForeignKey`来处理一对多关系，我们用大学和学生的关系来进行演示：一个学校可以有多个学生。

定义学生模型

    :::python
    class Student(models.Model):
        university = models.ForeignKey(University)
        name = models.CharField(max_length=50)

        def __str__(self):
            return self.name

模型操作：

    :::python
    >>> from myapp.models import University, Student

    # 创建学生
    >>> pku = University.objects.get(name="Peking University")
    >>> zhangsan = Student(name="Zhang San", university=pku)
    >>> lisi = Student(name="Li Si", university=pku)
    >>> zhangsan.save()
    >>> lisi.save()

    # 查看张三的学校信息
    >>> zhangsan.university
    <University: Peking University>
    >>> lisi.university.place.address
    'No.5 Yiheyuan Road Haidian District, Beijing'

    # 查看pku的学生信息
    >>> pku.student_set.all()
    <QuerySet [<Student: Zhang San>, <Student: Li Si>]>
    >>> pku.student_set.get(name="Li Si")
    <Student: Li Si>

    # 创建一个没有学校的学生
    >>> wangwu = Student(name="Wang Wu")
    >>> wangwu.save()
    >>> wangwu.university
    Traceback (most recent call last):
      File "<console>", line 1, in <module>
     ...
    django.db.models.fields.related_descriptors.RelatedObjectDoesNotExist: Student has no university.
    
    # 添加到pku
    >>> pku.student_set.add(wangwu)
    >>> wangwu.university
    <University: Peking University>
    >>> pku.student_set.all()
    <QuerySet [<Student: Zhang San>, <Student: Li Si>, <Student: Wang Wu>]>

    # 查看pku有几个学生
    >>> pku.student_set.count()
    3

    # 查看pku名字以"Li"开头的学生
    >>> pku.student_set.filter(name__startswith="Li")
    <QuerySet [<Student: Li Si>]>

    # 查看学校名字为"Peking University"的学生
    >>> Student.objects.filter(university__name="Peking University")
    <QuerySet [<Student: Zhang San>, <Student: Li Si>, <Student: Wang Wu>]>

## 多对多关系

Django使用`ManyToManyField`来处理多对多关系，我们用学生和课程的关系来进行演示：一个学生可以选择多个课程，每个课程可以有多个学生来上课。

定义课程模型

    :::python
    class Course(models.Model):
        student = models.ManyToManyField(Student)
        name = models.CharField(max_length=50)
    
        def __str_(self):
            return self.name
        
模型操作

    :::python
    >>> from myapp.models import University, Student, Course
    >>> hm = Course(name="Higher Mathematics")
    >>> ms = Course(name="Mathematical Statistics")
    >>> hm.save()
    >>> ms.save()
    >>> pku = University.objects.get(name="Peking University")
    >>> zhangsan = Student.objects.get(name="Zhang San")
    >>> lisi = Student.objects.get(name="Li Si")
    >>> wangwu = Student.objects.get(name="Wang Wu")

    # lisi目前还没有选择课程
    >>> lisi.course_set.all()
    <QuerySet []>

    # 高等数学目前还没有学生选择
    >>> hm.student.all()
    <QuerySet []>

    # 我们给lisi选择两门课程
    >>> lisi.course_set.add(hm, ms)
    >>> lisi.course_set.all()
    <QuerySet [<Course: Higher Mathematics>, <Course: Mathematical Statistics>]>
    >>> ms.student.all()
    <QuerySet [<Student: Li Si>]>

    # 我们再给数理统计课程添加两名学生
    >>> ms.student.add(zhangsan, wangwu)
    >>> ms.student.all()
    <QuerySet [<Student: Zhang San>, <Student: Li Si>, <Student: Wang Wu>]>

    # 直接给高等数学再创建并添加一名学生
    >>> zhaoliu = hm.student.create(name="Zhao Liu", university=pku)
    >>> hm.student.all()
    <QuerySet [<Student: Li Si>, <Student: Zhao Liu>]>

    # 交叉查询
    # 查询选择了高等数学的学生
    >>> Student.objects.filter(course__name="Higher Mathematics")
    <QuerySet [<Student: Li Si>, <Student: Zhao Liu>]>

    # 查询有李四参与的课程
    >>> Course.objects.filter(student__name="Li Si")
    <QuerySet [<Course: Higher Mathematics>, <Course: Mathematical Statistics>]>

    # 张三退掉数理统计
    >>> zhangsan.course_set.remove(ms)
    >>> zhangsan.course_set.all()
    <QuerySet []>

    # 从高等数学课程中删除赵六
    >>> hm.student.remove(zhaoliu)
    >>> hm.student.all()
    <QuerySet [<Student: Li Si>]>

---
参考：

+ [Django Model field reference](https://docs.djangoproject.com/en/1.10/ref/models/fields/#django.db.models.ManyToManyField)